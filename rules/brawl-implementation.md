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

## Match lifecycle + artifacts (v1)

### Authoritative match lifecycle (server-only)
- We do not rely on `AGameMode` match state. Brawl uses `Phase.*` tags + a replicated match-ended signal.
- [ABrawlGameMode::BeginPlay](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Components/BrawlCombatManagerComponent.h:15:1-15:35):
  - ensures `MatchId` exists ([FBrawlMatchId::NewId()](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlCore/Public/Types/BrawlIds.h:15:4-20:5))
  - subscribes to `ABrawlGameState::OnMatchEnded()`
  - starts replay recording (if enabled)
  - starts [UBrawlRoundManagerComponent::StartMatchFlow()](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Components/BrawlRoundManagerComponent.cpp:15:0-34:1)
- [UBrawlRoundManagerComponent](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Components/BrawlRoundManagerComponent.cpp:10:0-13:1) is the phase/round driver. When the `RoundSet` is exhausted, it ends the match via [ABrawlGameState::EndMatch()](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Game/BrawlGameState.cpp:36:0-51:1).
- [ABrawlGameState::EndMatch()](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Game/BrawlGameState.cpp:36:0-51:1):
  - sets replicated `bMatchEnded`
  - broadcasts `OnMatchEnded` (server + clients)

## Combat resolution + internal Rewards phase (v2)

### Combat resolution
- Combat resolution is performed server-side.
- Combat may end early only when all active arena fights have resolved (no phase advance while any arena is unresolved).
- Each arena resolution selects a winner using:
  1) Elimination: one side has 0 alive field units
  2) (If forced at combat end) UnitCount tiebreaker
  3) (If still tied) TotalHealth tiebreaker
  4) (If still tied) DeterministicPlayerId tiebreaker (lowest PlayerId wins)

### Rewards phase semantics (internal)
- Rewards exists as an internal phase used for deterministic reward application and match bookkeeping.
- Rewards has a duration of `0` seconds (immediate transition to Planning after applying rewards).
- While in Rewards:
  - shop actions are forbidden
  - placement is locked
- Rewards application includes:
  - winner/loser gold from round data (`WinGold`, `LossGold`)
  - player life damage computed as `BasePlayerDamage + SurvivingUnits` (survivors = alive field units for the winning side)
  - streak bookkeeping (win/loss streak updates)

### GameMode split (shipping vs sandbox)
- `ABrawlGameMode` is shipping-clean:
  - match lifecycle orchestration
  - replay start/stop
  - match event log export
- [ABrawlSandboxGameMode](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Game/BrawlSandboxGameMode.cpp:15:0-19:1) is dev-only by convention:
  - sandbox board spawning / seeding
  - debug exec commands: [BrawlDebugAdvancePhase](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Game/BrawlSandboxGameMode.h:20:1-20:31), [BrawlDebugAdvanceRound](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Game/BrawlSandboxGameMode.h:23:1-23:31), [BrawlDebugEndMatch](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Game/BrawlSandboxGameMode.h:26:1-26:27)
  - Arena transfer is match-owned (RoundManager/GameState) and is not special-cased by sandbox code.

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

## Overflow + bench-full purchase prevention (v2)

### Bench-full purchase prevention (hard stop)
- Shop purchases must require an available bench slot in the player’s allowed bench row.
- If the bench is full, the server rejects the purchase:
  - gold is not spent
  - the offer is not consumed
  - no unit is spawned
- This is enforced server-side; clients may only send purchase requests.

### Overflow grants (forced unit grants when bench is full)
Overflow applies to server-forced unit grants (e.g., carousel/itemshop-style grants) when the bench is full.

Grant behavior (server):
1) If the granted unit can immediately star-combine, resolve the combine and do not place/track overflow.
2) Else, if there is any free bench slot, place the unit on the bench.
3) Else (bench full), place the unit on the first free field tile found by scanning field coords in canonical order (lowest coord first).
   - Emit `Grid.OverflowSpawned` after placement.

Overflow resolution:
- During Planning, when bench space opens, move overflow units from field to bench in FIFO order.
- On Planning → Combat, enforce team-size cap and resolve overflow (see Team Size Cap / Overflow Enforcement).

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