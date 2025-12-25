---
trigger: always_on
description: 
globs: 
---

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