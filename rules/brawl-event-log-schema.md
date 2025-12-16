---
trigger: always_on
description: 
globs: 
---

# Event Log Schema (Structured, Server Auth)

Event Log exists alongside network replay to support debugging/testing/analysis.
It is NOT required for replay playback, but must be stable and comprehensive.

---

## 1) Global requirements
- Server-only emission; clients may view but never author.
- Every event includes:
  - `SchemaVersion` (int)
  - `MatchId` (string/guid)
  - `ServerTimeSeconds` (float, match time since start)
  - `RoundIndex` (int)
  - `PhaseTag` (GameplayTag)
  - `EventType` (name/tag)
  - `InstigatorUnitId` (optional)
  - `TargetUnitId` (optional)
  - `PlayerId` (optional, for economy/placement)

- Use stable IDs:
  - UnitInstanceId is assigned by server and never reused within a match.

---

## 2) Required event types

### Economy
- `Economy.Purchase` (offerId/unitTypeId/cost)
- `Economy.Sell` (unitId/refund)
- `Economy.Reroll` (cost, resulting offer list ids)
- `Economy.BuyXP` (cost, xpGained, newLevel)

### Placement
- `Grid.UnitMoved` (unitId, fromCoord, toCoord, fromBenchFlag, toBenchFlag)
- `Grid.UnitSwapped` (unitAId, unitBId, coordA, coordB)
- `Grid.OverflowSpawned` (unitId, coord)
- `Grid.OverflowResolved` (unitId, destroyedFlag, refund)

### Combat decisions (navmesh drift tolerant)
- `Combat.TargetChosen`
  - abilityId, targetId, policy (Retarget/Fizzle/LastPosition)
  - optional target position snapshot (vector) when using LastPosition
- `Combat.AbilityCast`
  - casterId, abilityId, castStartTime, lockedTargetId (optional)
- `Combat.ProjectileSpawned`
  - projectileId, sourceId, abilityId, targetId/targetPos, spawnTime
- `Combat.ProjectileImpacted`
  - projectileId, impactTime, impactedTargetId (optional), impactPos
- `Combat.DamageApplied`
  - sourceId, targetId, abilityId, amount, damageClassTag, elementTag, finalTagsApplied
  - damageClassTag is a `DamageClass.*` tag (Physical/Special/Mixed/True)
  - elementTag is the ability’s `Element.*` tag (used for effectiveness/STAB when applicable)
- `Combat.UnitDied`
  - unitId, killerId (optional), time

---

## 3) Emission points (guarantees)
- Emit `TargetChosen` when a cast locks a target OR when the targeting policy resolves.
- Emit `ProjectileSpawned` and `ProjectileImpacted` for all non-instant projectiles.
- Emit `DamageApplied` for every damage application (including DoTs, on-hit procs).
- Emit placement and economy events immediately after server commits the state change.

---

## 4) Ordering and stability
- When multiple events occur “at the same time”, order them deterministically:
  - sort by ServerTimeSeconds, then by stable UnitInstanceId, then by EventType priority.
- Trigger ordering invariant to reflect in logs:
  1) OnHit
  2) OnDamaged
  3) OnDeath

---