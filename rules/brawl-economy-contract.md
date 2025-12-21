---
trigger: model_decision
description: When working on BrawlEconomy or economy related tasks
globs: 
---

- Deterministic shop RNG:
  - Server-only FRandomStream seeded by (MatchId, PlayerId, RollSequenceNumber); offers are fully determined by these inputs and replicated to clients.
- Shared pool semantics:
  - Shop offer generation “reserves” pool copies; reroll/roll returns unbought offers to the pool.
  - Default pool copy counts (when tuning not provided): 1-cost 29, 2-cost 22, 3-cost 18, 4-cost 12, 5-cost 10.
- XP thresholds:
  - Progression uses a data-driven XPToNextLevel table; cumulative XP is consumed to compute Level.

- Roster vs placement (bench/field):
  - Roster = authoritative per-player ownership list (ALL units the player owns: bench + field).
    - Implemented as `ABrawlPlayerState::Roster` (`FBrawlPlayerRosterList`, FastArray; owner-only replication).
    - Entries store `{UnitId, UnitDataId, StarLevel}` and intentionally do NOT store bench/field.
  - Bench/field status is derived from canonical board occupancy ([ABrawlBoardActor](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlGrid/Private/Actors/BrawlBoardActor.cpp:115:0-128:1) / `FBrawlGridCoord`),
    or a unit’s `IBrawlGridOccupantInterface` coord snapshot kept in sync when the server commits occupancy changes.
  - Any system that needs “bench vs field” (traits, bench capacity checks, combat movement locks, etc.) must query
    canonical occupancy, not roster size.

- Trait counting semantics (for when Traits are implemented):
  - PotentialCounts = bench + field (UI only; shown like `2(3)/3`).
  - ActiveCounts = field only (effects applied).
  - Active vs Potential must be computed from placement (canonical occupancy), not inferred from roster.

- Bench capacity validation:
  - Purchases are requests; the server validates affordability and available bench space.
  - Bench space is determined by scanning the player’s allowed bench row on the canonical board occupancy
    (NOT by roster length).

- Star combining / StarLevel upgrades:
  - Design decisions generally TFT-like: 3 of the same unit will upgrade (a fielded unit preferentially, followed by a bench unit), with the other two units despawned/returned to the unit actor pool (NOT the shop pool)
  - Combining must never resolve for a unit actively IN combat: `Phase.Combat` AND unit is on the field (NOT bench).
  - Pending combines involving a field unit in combat are deferred and resolved at the start of the next `Phase.Planning`.
  - On StarLevel increase, the server reinitializes pools:
    - `CurrentHealth = MaxHealth`
    - `CurrentEnergy = 0`
