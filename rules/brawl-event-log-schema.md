---
trigger: always_on
description: 
globs: 
---

# Event Log Schema (Authoritative Decisions)

## Purpose
This document defines:
- the canonical structured events emitted by the server
- required fields per event
- ordering and stability guarantees

The EventLog is used for:
- replay
- spectate
- auditing
- deterministic testing

If a gameplay decision is not replicated, it must be logged here.

---

## 1) Hard Rules (Non-Negotiable)

- Events are emitted **server-side only**.
- Each event is emitted **exactly once**.
- Events are immutable after emission.
- Ordering must be deterministic.
- Events are authoritative over replay.

If an event would be ambiguous or duplicated: STOP.

---

## 2) Common Event Fields

All events include:

| Field                  | Description                         |
|------------------------|-------------------------------------|
| `EventId`              | Stable unique identifier             |
| `EventType`            | Canonical event name                 |
| `ServerTimeSeconds`    | Server-authored timestamp            |
| `MatchId`              | Match identifier                     |
| `RoundIndex`           | Round number                         |

No optional timestamps.

---

## 3) Unit & Actor References

References use:
- `FBrawlUnitInstanceId`
- `FBrawlPlayerId`

Rules:
- Never reference raw actors
- Never reference pointers
- IDs must already be known to the receiver

---

## 4) Canonical Combat Events

### `Combat.AbilityCast`
Emitted when:
- an ability is committed

Fields:
- `SourceUnitId`
- `AbilityId`
- `TargetRef` (unit or position)

---

### `Combat.TargetChosen`
Emitted when:
- targeting is resolved

Rules:
- Emitted once per cast
- Must precede damage events

---

### `Combat.Damage`
Emitted when:
- damage is applied

Fields:
- `SourceUnitId`
- `TargetUnitId`
- `Amount`
- `DamageClass`
- `Element`

---

### `Combat.Heal`
Mirrors `Combat.Damage`.

---

### `Combat.ProjectileSpawned`
Emitted when:
- a projectile is created

---

### `Combat.ProjectileImpacted`
Emitted when:
- a projectile applies impact

---

### `Combat.UnitDeath`
Emitted when:
- a unit transitions to dead

---

## 5) Economy Events

### `Economy.PurchaseCommitted`
### `Economy.UnitSold`
### `Economy.Reroll`

Rules:
- Emitted only on success
- Rejections are not logged

---

## 6) Ordering Guarantees

- Events are totally ordered per match.
- Within a cast:
  1. `TargetChosen`
  2. `AbilityCast`
  3. Damage / Heal
  4. Death (if applicable)

No reordering is permitted.

---

## 7) Stability & Versioning

- Event names are stable.
- Field removal is forbidden.
- Field addition requires:
  - backward compatibility
  - default handling

---

## 8) Forbidden Patterns

- Emitting derived values only
- Logging UI-only actions
- Client-emitted events
- Reconstructing outcomes from multiple events

---

## References
- Replication: `brawl-replication-contract.md`
- Testing: `brawl-testing-matrix.md`
- Validation: `brawl-server-validation-checklist.md`