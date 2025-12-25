---
trigger: model_decision
description: When working on BrawlEconomy or economy related tasks
globs: 
---

- Deterministic shop RNG:
  - Server-only FRandomStream seeded by (MatchId, PlayerId, RollSequenceNumber); offers are fully determined by these inputs and replicated to clients.
- Shared pool semantics:
  - Shop offer generation “reserves” pool copies; reroll/roll returns unbought offers to the pool.
- Purchase consumption rule:
  - When a player buys a shop offer that was generated from the shared pool, the reserved pool copy becomes permanently consumed.
  - Implementation detail: on purchase, remove that offer’s `UnitDataId` from the shop’s `ReservedPoolUnitIds` so it is NOT returned to the pool on the next reroll/roll.
- Offer roll exhaustion policy:
  - For each offer slot, if the initially rolled cost bucket has no available shared-pool copies, re-roll among the remaining costs (without repeating a cost) until a unit is found.
  - If no cost bucket has availability, fall back to rolling any available unit from the pool.
  - If the pool is fully exhausted/unavailable (sandbox/dev), fall back to deterministic/test unit IDs.
- Deterministic/fallback offers (sandbox):
  - If the shared pool is unavailable/exhausted, the shop may fall back to deterministic unit IDs (e.g., `DeterministicOfferUnitIds` / `DA_TestUnit`) for sandbox/dev.
  - These fallback offers do not reserve/consume the shared pool.
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

- Trait thresholds + effects (v1):
  - Threshold evaluation is **highest tier wins** (non-cumulative).
  - Effects apply from ActiveCounts only (field-only).
  - Effects may target either ContributorsOnly or AllOwnedFieldUnits, per threshold.

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

- Phase legality (economy actions):
  - Shop actions (purchase/reroll/sell/buy XP) are allowed only during `Phase.Planning` and `Phase.ItemShop` unless a specific round explicitly overrides this.
  - Shop actions are NOT allowed during `Phase.Rewards`.

- Selling refund policy (v2):
  - Refund is based on unit cost per copy (from UnitData) and StarLevel:
    - 1-star: `Refund = 1 * Cost`
    - 2-star: `Refund = 3 * Cost - 1`
    - 3-star: `Refund = 9 * Cost - 2`
  - This should be designer-editable via tuning (e.g., per-star penalty, or a refund curve/table).

- Selling shared-pool return (v2):
  - Selling returns copies to the shared pool based on StarLevel:
    - 1-star returns 1 copy
    - 2-star returns 3 copies
    - 3-star returns 9 copies

- Overflow enforcement timing (v2):
  - Overflow is enforced at end of Planning/ItemShop only.
  - If field unit count exceeds the player’s allowed team size cap:
    - Move the last placed field unit to bench if there is a free bench slot.
    - Otherwise destroy it and apply the same refund + shared-pool return policy as a sell.