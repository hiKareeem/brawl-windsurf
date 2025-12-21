---
trigger: always_on
description: 
globs: 
---

# Brawl Project Implementation Rules

## Non-negotiable goals
- Server-authoritative simulation for all gameplay-relevant actions.
- Replays must reproduce the full match exactly as it occurred on the server.
- No crits and no uncontrolled randomness. If randomness exists (i.e., shop), it must be:
  - server-only
  - seeded
  - logged for replay/debug

### Economy/Shop RNG policy (v0)
- **Server-only RNG**: Clients never compute or provide RNG outcomes for shop offers. They only send requests (reroll/purchase).
- **Seeded + auditable**: Shop offer rolling uses a deterministic seed derived from:
  - `MatchId`
  - `PlayerId`
  - a monotonic `RollSequenceNumber` (increments per roll/reroll)
- **Shared pool**: Shop rolls draw from the server-authoritative shared pool (consume/return rules), not from local client lists.

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
  - Treat `Phase.ItemShop` as **planning-like** for board interactions (bench/field moves + swaps) and **ability equip**.
- Combat phase:
  - field units are non-interactive (no reposition)
  - bench units may be moved between bench tiles only, and only within the player’s allowed bench row (host row for host player, guest row for guest player)
- Rewards phase:
  - placement is locked (no player-initiated moves)
  - Non-arena boards remain in the world but are empty/non-interactive while their owner is fighting elsewhere; newly purchased units spawn on the player’s bench on the current arena board (guest bench if they are the guest).
  - Post-combat: transfer guest units back to their original board; dead units are unpooled/destroyed and damaged units are reinitialized as needed (Match-owned orchestration).
- Coordinate mapping for transfer (guest-side mirroring):
  - Guest-side placement is a 180° rotation relative to the host. Example: a unit at “view (0,0)” on the guest bench corresponds to canonical bench coord `(X=BenchWidth-1, Y=1)` (e.g., `8,1` when BenchWidth is 9).
- Overflow:
  - shop purchase disallowed if bench full
  - overflow from other sources spawns at field (0,0) using canonical coords
  - if over capacity at end of planning: destroy excess unit(s) and refund cost; they must not participate in combat
- Tiles use an interaction trace channel for coord picking.
- Units ignore that channel by design.
- Unit click/hover uses a different channel and resolves to `UnitId -> Occupancy` (not actor transform).
- This preserves TFT-like UX while keeping the board occupancy authoritative.

### Arena terminology (clarification)
An “arena board” is not a special actor type. It refers to an existing ABrawlBoardActor that the Match designates as the host board for a combat pairing for the current round.
Opponent units (and their bench) are transferred onto the host board’s guest half. The guest player’s home board remains spawned but is treated as inactive/empty for that round.

### Star combining / StarLevel changes (v1 rule)
- Unit StarLevel upgrades (combines) must never resolve for a unit actively IN combat. It is in combat when it is Phase.Combat and the unit is on the field NOT on the bench. 
- Units on the bench can still combine at any point.
- Pending combines involving a unit in combat resolve at the start of the next `Phase.Planning` (TFT-like).
- On StarLevel increase, the server reinitializes pools:
  - `CurrentHealth = MaxHealth`
  - `CurrentEnergy = 0`

### Star combining / StarLevel upgrades — implementation notes (v1)

- [ABrawlUnitCharacter::SetStarLevel(...)](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlUnit/Public/Actors/BrawlUnitCharacter.h:56:4-56:41) remains a low-level **server-only** setter used by initialization and tests.
  - Do **not** embed phase/placement guard logic inside [SetStarLevel(...)](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlUnit/Public/Actors/BrawlUnitCharacter.h:56:4-56:41).

- The StarLevel combine guard/deferral must be enforced in the **combine-resolution path** (e.g., [UBrawlStarCombineComponent::RequestStarLevelUpgrade(...)](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlUnit/Public/Components/BrawlStarCombineComponent.h:16:1-16:83)):
  - If `PhaseTag == Phase.Combat` AND unit is on the **field** → do not apply immediately; defer.
  - If unit is on the **bench** → apply immediately (even during `Phase.Combat`).

- Bench vs field is derived from canonical board placement:
  - The authoritative board sets a server-authored `FBrawlGridCoordSnapshot` on units via `IBrawlGridOccupantInterface`.
  - If placement is unknown / snapshot missing, treat as **field** for guard purposes (conservative).

- Pending upgrades resolution:
  - Store pending upgrades as `UnitId -> DesiredStarLevel` (max desired wins).
  - Resolve at the start of the next `Phase.Planning`.
  - Apply in deterministic order sorted by stable `FBrawlUnitInstanceId`.