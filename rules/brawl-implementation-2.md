---
trigger: always_on
description: 
globs: 
---

### GameMode split (shipping vs sandbox)
- `ABrawlGameMode` is shipping-clean:
  - match lifecycle orchestration
  - replay start/stop
  - match event log export
- [ABrawlSandboxGameMode](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Game/BrawlSandboxGameMode.cpp:15:0-19:1) is dev-only by convention:
  - sandbox board spawning / seeding
  - debug exec commands: [BrawlDebugAdvancePhase](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Game/BrawlSandboxGameMode.h:20:1-20:31), [BrawlDebugAdvanceRound](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Game/BrawlSandboxGameMode.h:23:1-23:31), [BrawlDebugEndMatch](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Game/BrawlSandboxGameMode.h:26:1-26:27)
  - Arena transfer is match-owned (RoundManager/GameState) and is not special-cased by sandbox code.

- Arena transfer (guest-side mirroring) — canonical coord mapping (v2)
  - Guest home-board coords are expressed on the host half (field `Y=0..FieldHalfHeight-1`, bench row `Y=0`).
  - When transferring a guest player’s units onto the arena host board, map guest coords into the host board’s guest half via a 180° rotation:
    - Field (`bIsBench=false`):
      - `X' = FieldWidth - 1 - X`
      - `Y' = (2*FieldHalfHeight - 1) - Y`
      - Example (FieldWidth=9, FieldHalfHeight=4): `(0,0) -> (8,7)`
    - Bench (`bIsBench=true`):
      - `X' = BenchWidth - 1 - X`
      - `Y' = 1` (guest bench row)
  - After transfer, the guest player’s home board occupancy is cleared (avoid duplicate authoritative occupancy).
- Host/guest board designation is server-only and deterministic (seeded) (v0/1v1):
  - Collect 2 valid [ABrawlPlayerState](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Game/BrawlPlayerState.cpp:43:0-52:1) entries and sort by `PlayerId`.
  - Compute a deterministic “coin flip” using `(MatchId, RoundIndex)`.
  - Host is chosen from the sorted list based on the coin flip; the host player’s home `BoardActor` is used as the arena board for that round.
  - During `Phase.Combat`, both players’ `ActiveBoardActor` are set to the arena board. After combat, restore each player’s `ActiveBoardActor` to their home `BoardActor`, and clear `ArenaBoard.GuestPlayerId`.

- Arena transfer return semantics (bench shuffles persist):
  - Field units return to their original home-board coord captured at transfer time.
  - Bench units return based on their final arena bench coord (inverse-mirrored back to home bench row `Y=0`) so bench-only movement during combat is preserved.

- Combat phase:
  - Combat can end early only when one side has no alive field units on **all arena boards**.
  - Do not advance phase while any arena fight is still ongoing.
  - When an arena fight ends early (one side eliminated), apply `State.Victory` to surviving units on the winning side to disable AI/targeting and allow victory presentation while other arenas finish.

- Overflow (team-size cap) enforcement:
  - Enforced at end of Planning/ItemShop only.
  - If the player has more field units than allowed:
    - Move the most-recently-placed field unit to the bench if there is a free bench slot.
    - Otherwise destroy it, refund, and return its shared-pool copies (see economy contract).

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

## Blueprint assets
- if an asset should exist as a Blueprint (eg GameplayEffects etc) let me know how you'd like me to set it up and I'll do it. Please do not try to shoehorn things into C++ that are better handled in Blueprint.