---
trigger: model_decision
description: Server validation checklist for competitive integrity
globs: 
---

# Server Validation Checklist (Competitive Integrity)

All entries here are mandatory validations for server RPC handling.

---

## 1) Global validations for every request
- Caller is authenticated and associated with a PlayerState in this match.
- Rate limit repeated requests (anti-spam).
- Validate current PhaseTag and RoundType allows the action.
- Validate referenced UnitInstanceIds belong to the requesting player when appropriate.
- Never accept client-supplied “result” fields (damage amounts, RNG results, etc.).

---

## 2) Shop/Economy requests
### PurchaseUnit(OfferId)
Validate:
- OfferId exists and is currently offered to that player
- Player has enough gold
- Bench has space (hard rule: full bench blocks shop purchases)
- Shared pool has availability for that unit type
Commit:
- Deduct gold
- Consume pool
- Spawn unit on bench
- Replicate roster + bench occupancy + updated offers if needed
Log: `Economy.Purchase`

### SellUnit(UnitId)
Validate:
- Unit exists and owned by player
- Phase allows selling (usually planning/shop; decide and enforce)
Commit:
- Remove unit, return to pool, refund
Log: `Economy.Sell`

### RerollShop()
Validate:
- Phase allows reroll
- Gold sufficient
Commit:
- Deduct gold
- Generate new offers (server-only; seeded if RNG)
Log: `Economy.Reroll`

### BuyXP()
Validate:
- Phase allows
- Gold sufficient
Commit:
- Deduct gold, add XP, handle level up + team-size cap changes
Log: `Economy.BuyXP`

---

## 3) Placement requests
### ServerRequestMoveUnit(UnitId, ToCoord)
Validate:
- Unit exists and is owned by player (or is on the player’s allowed half/bench row as a proxy until full ownership mapping exists)
- Coordinate in bounds (`FBrawlGridCoord` includes `bIsBench`)
- Phase legality (server-derived; do not trust client):
  - planning/shop: bench<->field allowed; guest half forbidden for player placement on home boards
  - combat: bench-to-bench only; field locked; must be within player’s allowed bench row
  - rewards: no player-initiated moves
- Swap rule:
  - if occupied, server performs swap only if allowed by the phase (and any PlacementRuleset flags)

Commit:
- Update occupancy state
- Replicate board mapping
Log: `Grid.UnitMoved` or `Grid.UnitSwapped`

Overflow handling:
- Non-shop overflow spawns at (0,0)
- End of planning: enforce capacity
  - destroy excess, refund cost, prevent combat participation
Log: `Grid.OverflowResolved`

---

## 4) Ability loadout requests
### EquipAbilities(UnitId, BasicAbilityId, UltimateAbilityId)
Validate:
- Unit owned by player
- Ability ids are among the unit’s defined 3 basic/3 ultimate options
- Phase allows changing loadout (default: `Phase.Planning`; also allow `Phase.ItemShop` if ItemShop is planning-like)
Commit:
- Replicate equipped IDs
- Server grants/removes GAS abilities accordingly

---

## 5) Spectator permissions
- Spectators cannot send gameplay requests.
- Spectator visibility rules must be enforced (what economy info they can see).

---