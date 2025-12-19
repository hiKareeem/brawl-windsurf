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

## Data-driven authoring requirements
- Designers author content via DataAssets + GAS assets:
  - UnitData drives stat curves, tags, ability options, visuals
  - AbilityData drives cooldown/power/targeting/projectile behavior
  - Traits/items defined as thresholds + effects (GEs / ability grants / tag changes)
- All “designer knobs” must be Blueprint-editable, including:
  - round phase durations per round
  - economy tuning tables (interest, streaks, roll odds by level)

## Networking, spectating, replay
- Clients are never trusted for:
    - purchases
    - placement legality
    - ability equip selection
    - any combat outcome
- All client actions are requests; server validates and executes.
- Spectators can see all arenas/boards.
- Server records match replays (DemoNetDriver). Replays must include:
    - purchases, sells, rerolls
    - placement swaps/moves
    - combat outcomes and ability usage

## Logging and testing
- Provide log categories:
    - `LogBrawlMatch`, `LogBrawlEconomy`, `LogBrawlCombat`, `LogBrawlAI`, `LogBrawlGrid`, `LogBrawlNet`
- Combat log events must be structured (not just printf strings).
- Support headless simulation tests:
    - spawn units
    - run N seconds
    - assert key invariants (e.g., kill time, ability cast counts, no NaNs)

## Code style and architecture conventions
- C++ systems; Blueprint for content and presentation.
- Prefer composition via components/subobjects over deep inheritance.
- Use interfaces for cross-module communication.
- Avoid tick when event-driven is possible; if ticking, keep it centralized (phase manager, AI scheduler).
- Stable ordering: whenever iterating units for rule resolution, sort deterministically (by stable ID).

--- 
Addendum

### Server model
- **Dedicated server authoritative only.**
- Clients never author gameplay state. All client actions are **requests**; server validates and executes:
    - shop purchases/sells/rerolls
    - XP buys/leveling
    - bench/field placement and swaps
    - ability loadout changes (equipped basic + ultimate)
- Competitive integrity is achieved via server authority + validation, not via client prediction.

### Replay, spectate, and navmesh drift tolerance
- **Primary replay system:** Unreal network replay (DemoNetDriver). Replays reproduce what the server replicated.
- **No requirement to re-simulate combat deterministically** during replay playback.
- **Navmesh / movement determinism is not required** across platforms.
- Add a **structured Event Log** alongside replay recording for debugging/testing and optional offline analysis.
    - Event log may include **combat decisions** (target chosen, retarget/fizzle mode taken) and **projectile impact timestamps/IDs** to allow consistent post-hoc analysis even if movement differs between runs.

### Combat speed
- `Spd` is a first-class combat stat with two effects:
    1) modifies **move speed**
    2) modifies **ability cooldown rate** (global rule)
- Cooldown scaling must be implemented in one place (central policy) so designers can reason about it and tests can assert it.

### Damage formula ownership
- Damage/heal formulas are owned by **global Execution Calculations** (e.g., `ExecCalc_Damage`), not hand-rolled per-ability.
- Abilities provide inputs (power, damage class, element/type tags, etc.); the ExecCalc resolves final numbers.

### Traits: active vs potential
- Trait evaluation is recomputed when roster changes:
    - unit added/removed
    - unit moved between bench/field
    - unit transforms into a different tagged archetype
- The system tracks and replicates both:
    - **Active trait counts** (field-only, currently applied)
    - **Potential trait counts** (field + bench, for UI display)
- UI should be able to display: `Active(Potential)/Threshold` (example: `2(3)/3`).

---