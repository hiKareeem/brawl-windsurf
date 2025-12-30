---
trigger: always_on
description: 
globs: 
---

# Brawl Project Implementation Rules

## Non-negotiable goals
- Server-authoritative simulation for all gameplay-relevant actions.
- Replays must reproduce the full match exactly as it occurred on the server.
- No crits and no uncontrolled randomness. If randomness exists (i.e., shop, host v guest designation), it must be:
  - server-only
  - seeded
  - logged for replay/debug

## Module boundaries and dependency rules
- **BrawlCore** has no dependencies on other Brawl modules.
- **BrawlGrid  / BrawlAbilities / BrawlAI** depend only on **BrawlCore** (and UE/GAS).
- **BrawlUnit** may additionally depend on the BrawlAbilities for the ASC subclass
- **BrawlEconomy / BrawlMatch / BrawlNet / BrawlUI** may depend on the core modules.
- **BrawlUnit must not depend on UI or Match flow.**
- **BrawlUI must not contain gameplay rules** (only viewmodels + presentation).
- Additional modules can be added and may be suggested to the user but should not be created unless the user outlines them above.

## GameplayTags are the primary rules language
- All rule decisions use GameplayTags and tag queries (avoid enums unless perf-critical).
- Tag namespaces (examples; extend as needed):
  - `Phase.Planning`, `Phase.Combat`, `Phase.Rewards`, `Phase.ItemShop`
  - `State.Alive`, `State.Dead`, `State.CC.Stun`, `State.CC.Silence`, `State.Invulnerable`
  - `DamageClass.Physical`, `DamageClass.Special`
  - `Element.Fire`, `Element.Water`, etc.
  - `Trait.<Name>.*`
  - `Item.<Name>.*`
  - `Ability.Basic.<Name>`, `Ability.Ultimate.<Name>`
  - `Faction.Player`, `Faction.Creep`, `Faction.Summon`

## GAS usage policy
- GAS governs: combat stats, damage, healing, buffs/debuffs, CC, immunities, auras, ability granting.
- Non-GAS governs: economy, shop, shared pool, XP/level, round state machine.
- Player/Board ASC may exist for global effects, but economy state must not be encoded as GameplayEffects.

## Combat rules
- Stats schema: HP / Atk / SpAtk / Def / SpDef / Spd (+ Energy).
- Basics:
  - fire automatically on cooldown
  - generate energy on use/hit (designer-configured)
- Ultimates:
  - cast automatically when energy is full (unless designer overrides)
- Trigger ordering is enforced globally:
  1) OnHit
  2) OnDamaged
  3) OnDeath
- Projectiles:
  - travel time affects impact timing
  - damage/events occur at impact, not at cast time
- Pathing/movement:
  - navmesh-driven during combat
  - grid occupancy is only enforced in planning and at combat start positions
  - if melee can’t reach target: idle (TFT-like)

## Match lifecycle + artifacts (v1)

### Authoritative match lifecycle (server-only)
- We do not rely on `AGameMode` match state. Brawl uses `Phase.*` tags + a replicated match-ended signal.
- [ABrawlGameMode::BeginPlay](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Components/BrawlCombatManagerComponent.h:15:1-15:35):
  - ensures `MatchId` exists ([FBrawlMatchId::NewId()](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlCore/Public/Types/BrawlIds.h:15:4-20:5))
  - subscribes to `ABrawlGameState::OnMatchEnded()`
  - starts replay recording (if enabled)
  - starts [UBrawlRoundManagerComponent::StartMatchFlow()](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Components/BrawlRoundManagerComponent.cpp:15:0-34:1)
- [UBrawlRoundManagerComponent](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Components/BrawlRoundManagerComponent.cpp:10:0-13:1) is the phase/round driver. When the `RoundSet` is exhausted, it ends the match via [ABrawlGameState::EndMatch()](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Game/BrawlGameState.cpp:36:0-51:1).
- [ABrawlGameState::EndMatch()](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Game/BrawlGameState.cpp:109:0-145:1) (server-only) performs best-effort teardown-safe cleanup before broadcasting match end:
  - If `UWorld` is valid and not tearing down:
    - cleanup any active ghost roster snapshot ([ServerCleanupGhostRosterSnapshot()](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Game/BrawlGameState.cpp:573:0-651:1))
    - if an arena transfer plan is still active, return transferred units ([ReturnUnitsFromArenaTransferPlan()](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Game/BrawlGameState.cpp:866:0-999:1))
    - restore every player’s `ActiveBoardActor` back to their stable home `BoardActor`
  - Then set replicated `bMatchEnded` and broadcast `OnMatchEnded` (server + clients)

Rationale:
- Matches can end while units are transferred/away; we must not leave boards, guest ids, or active-board presence in a mid-combat state.
- Cleanup is skipped during teardown to avoid late-replication/PIE shutdown hazards.

## Combat resolution + internal Rewards phase (v2)
### Combat end conditions: double elimination tie-break (v1)

- An arena resolves when either team has 0 alive **field** units (bench excluded).
- If both teams have 0 alive field units:
  - compare the EventLog `SequenceNumber` of the killing `Combat.DamageApplied` event for each team’s final unit death
  - team eliminated second (higher sequence) wins
  - if ordering cannot be determined, fallback winner = host

### Rewards phase semantics (internal)
- Rewards exists as an internal phase used for deterministic reward application and match bookkeeping.
- Rewards has a duration of `0` seconds (immediate transition to Planning after applying rewards).
- While in Rewards:
  - placement is locked
- `Phase.Rewards` is teardown-only: no placement and no shop/economy interactions are allowed.
- Rewards application includes:
  - winner/loser gold from round data (`WinGold`, `LossGold`)
  - player life damage computed as `BasePlayerDamage + SurvivingUnits` (survivors = alive field units for the winning side)
  - streak bookkeeping (win/loss streak updates)

## Combat-phase shop purchases (arena-aware) (v2)

- Allowed phases for shop purchase: `Phase.Planning`, `Phase.ItemShop`, `Phase.Combat`.
- Forbidden: `Phase.Rewards`.
- Purchases during `Phase.Combat` always spawn onto the purchaser’s allowed bench row on the *active* board.
- Allowed bench row is derived server-side from `Board.HostPlayerId/GuestPlayerId` (never client-authored).
- If the bench row is full: reject purchase (no gold spend, no offer consume, no spawn).
- If `PhaseTag == Phase.Combat` and `ActiveBoardActor != BoardActor`:
  - record an `ArenaTransferPlan` move with `FromCoord.bIsBench = true` so return uses the unit’s current arena bench coord and mirrors X back to the home bench row `Y=0`.

### Replay + Event Log artifacts (server-only)
- Replay:
  - ReplayName: `Brawl_<MatchId>`
  - Output: `Saved/Demos/<ReplayName>/...`
- Event log export (JSONL):
  - Output directory: `Saved/<MatchEventLogExportSubdir>/`
  - Filename: `MatchEventLog_<MatchId>_<UTC timestamp>.jsonl`
  - Controlled by `UBrawlNetSettings` in `Config/DefaultEngine.ini`

### GameLift integration notes (v0, deferred implementation)
- Preferred early approach: write artifacts locally and rely on GameLift fleet log collection to upload to S3.
- On match end:
  - stop replay recording
  - export JSONL event log
  - copy artifacts into the directory GameLift is configured to collect as server logs

### Economy/Shop RNG policy (v0)
- **Server-only RNG**: Clients never compute or provide RNG outcomes for shop offers. They only send requests (reroll/purchase).
- **Seeded + auditable**: Shop offer rolling uses a deterministic seed derived from:
  - `MatchId`
  - `PlayerId`
  - a monotonic `RollSequenceNumber` (increments per roll/reroll)
- **Shared pool**: Shop rolls draw from the server-authoritative shared pool (consume/return rules), not from local client lists.

## Grid and placement rules
- Board layout is **data-driven** and must not be hard-coded in C++ (no magic numbers for `FieldWidth`, `FieldHalfHeight`, `BenchWidth`). Default tuning is 9x4 per side + 9x1 bench, but designers must be able to change this via DataAssets/Blueprint config for testing (e.g., TFT-like 7x4, 10-space bench, etc.). `BenchRowCount` is currently fixed at 2 (host `Y=0`, guest `Y=1`). Bench width may differ from field width (TFT-style).
- Canonical coordinate system uses `FBrawlGridCoord { X, Y, bIsBench }`:
  - Bench:
    - `bIsBench = true`: `X=0..(BenchWidth-1)`
    - `bIsBench = true`: `Y=0` = host bench row, `Y=1` = guest bench row
  - Field:
    - `bIsBench = false`: `X=0..(FieldWidth-1)`
    - `bIsBench = false`: `Y=0..(FieldHalfHeight-1)` = host field rows
    - `bIsBench = false`: `Y=FieldHalfHeight..(2*FieldHalfHeight-1)` = guest field rows
  - UI/view mapping is separate and not authoritative.
- World-space layout is gameplay-relevant (placement movement, targeting, projectile travel): the authoritative board owns deterministic `FBrawlGridCoord -> WorldTransform` and is driven by:
  - Field segment: `TileSpacing` and `TileSize`
  - Bench segment: `BenchTileSpacing` and `BenchTileSize` (if unset/0, fall back to the field values)
  - `BenchFieldGap` is treated as an edge-to-edge gap between the bench and the nearest field row so differing tile sizes do not change the intended separation
  - Cosmetic meshes must be decorative only and must not affect collision/placement legality.
- Board lifecycle (8-player, TFT-like):
  - Planning: **8 boards active**, each board contains only the owning player’s units on the host half; **guest half is forbidden** for placement on a player’s home board.
  - Combat: up to **4 arena boards active** (one per pairing). The paired opponent’s units are **transferred** onto the host board’s guest half so host+guest fields form the combat arena. Bench units are also transferred so the guest bench row is active/interactable.
- Planning phase:
  - units can move freely between bench and field
  - dropping onto an occupied tile swaps units
- ItemShop phase:
  - treat as planning for placement (bench<->field allowed; swap-on-drop rules apply)
- ItemShop phase semantics:
  - `Phase.ItemShop` represents an **ItemShop** round (carousel-like), not the UnitShop.
  - Treat `Phase.ItemShop` as **planning-like** for board interactions (bench/field moves + swaps), **ability equip**, and **item equip**.
- Combat phase:
  - field units are non-interactive (no reposition)
  - bench units may be moved between bench tiles only, and only within the player’s allowed bench row (host row for host player, guest row for guest player)
- Rewards phase:
  - placement is locked (no player-initiated moves)
  - shop actions are not allowed (Rewards is teardown + rewards distribution + planning setup)
  - Post-combat: transfer guest units back to their original home board.
    - Dead units are not destroyed; they are reset/respawned on the home board (full health/energy, etc.) as part of post-combat reinitialization.