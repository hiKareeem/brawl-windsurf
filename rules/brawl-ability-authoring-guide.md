---
trigger: model_decision
description: Ability authoring guide
globs: 
---

# Ability Authoring Guide (Data-driven, GAS-first)

Goal: abilities remain modular and designer-friendly. Avoid bespoke one-off logic.

---

## 1) Required structure
Every ability must be built from:
- AbilityData (numbers + tags + targeting policy + projectile policy)
- A GameplayAbility class derived from:
  - `UBrawlGA_BasicBase` or `UBrawlGA_UltimateBase`

Do not implement unique cooldown/energy/damage rules inside individual abilities unless the base system cannot express them.

---

## 2) Damage and healing
- Abilities must not calculate final damage directly.
- Abilities provide inputs:
  - Base power (e.g., SetByCaller `Data.Power`)
  - Damage class tag (`DamageClass.*`: Physical/Special/Mixed/True)
  - Element tag (`Element.*`)
  - Optional modifiers via SetByCaller magnitudes and tags
  - `ExecCalc_Damage` applies mitigation, type effectiveness, and STAB centrally.
Optional standard modifiers (SetByCaller magnitudes):
- `Data.Mod` (multiplicative, defaults to 1.0)
- `Data.Flat` (additive, defaults to 0.0)

Cooldown authoring convention:
- When applying a cooldown GameplayEffect, set SetByCaller `Data.CooldownBaseSeconds` from `AbilityData.CooldownBaseSeconds` so the cooldown MMC can apply the global Spd scaling policy.

---

## 3) Targeting policies
Each ability references a TargetingPolicy that defines:
- target selection mode (nearest, lowest HP, backline, threat, cone, etc.)
- behavior on target death during cast:
  - Retarget / Fizzle / LastPosition
- eligibility tag queries (e.g., ignore `State.Dead`, ignore `State.Immune.CC`, etc.)
- currently global within the current world; arena scoping deferred
- Determinism (required):
  - Target resolution must be deterministic across runs; do not rely on iteration order for tie resolution.
  - When candidates tie (e.g., `NearestEnemy` distance within an epsilon), break ties by lowest stable `FBrawlUnitInstanceId` (i.e., `UnitId.Value`).
  - Use a small epsilon (recommend: `KINDA_SMALL_NUMBER` for `DistSq` comparisons) and apply this tie-break consistently for any new selection modes.
- `LastPosition` clarification:
  - When `LastPosition` is chosen, the “target position snapshot” is taken at target resolution time and should be recorded in `Combat.TargetChosen` (and/or `Combat.ProjectileSpawned` if a projectile is used).

---

## 4) Central combat driver integration (locked target + event emission)
When combat casting is driven centrally (not per-unit tick), the server may resolve a target *before* activating the ability.

- Central driver: [UBrawlCombatManagerComponent](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Components/BrawlCombatManagerComponent.cpp:18:0-21:1) (BrawlMatch)
  - Resolves `UBrawlTargetingPolicy`
  - Publishes `Combat.TargetChosen` and `Combat.AbilityCast` via `UBrawlGA_Base`

- Abilities must avoid double-emitting these events.
  - If the avatar implements `IBrawlCombatTargetProviderInterface` ([GetLockedTarget](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlUnit/Public/Actors/BrawlUnitCharacter.h:43:4-43:64), [GetLockedAbilityId](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlUnit/Public/Actors/BrawlUnitCharacter.h:44:4-44:54)), an ability may:
    - Use the locked target when `LockedAbilityId == AbilityId`
    - Skip re-publishing `Combat.TargetChosen` / `Combat.AbilityCast` (the driver already emitted them)

- If no locked target is provided (or it doesn’t match), the ability resolves target normally and emits events normally.

---

## 5) Projectiles
If travel time matters:
- Use a projectile actor policy; **impact applies effects/damage** (server-authoritative).

- Snapshot timing:
  - If using `LastPosition` (or any “fire at location” behavior), capture the target position snapshot at **target resolution / cast start time**, not at impact time.
  - The projectile uses that snapshot for its aim. It does not “retarget mid-flight”.

- Event log (server-only):
  - Emit `Combat.ProjectileSpawned` when the projectile is spawned.
  - Emit exactly one `Combat.ProjectileImpacted` when the projectile ends (hit, miss, expiry, destroyed).
  - All projectile event timestamps (`SpawnTimeSeconds`, `ImpactTimeSeconds`) must use the same timebase as `FBrawlEventBase.ServerTimeSeconds` (match time since start).

---

## 6) On-hit procs, auras, persistent effects
Preferred patterns:
- On-hit: apply a GE that triggers via tags/events, or use a standardized on-hit ability hook.
- Aura: periodic GE application driven by a standardized aura component/ability pattern.
- Persistent zones: standardized “AOE zone actor” pattern (server-authoritative) rather than custom per ability.

---

## 7) Conditional trait/item interactions
Complex conditions must be implemented using a consistent mechanism (pick one and stick to it):
- Tag-driven + standardized listeners (e.g., “FirstCast”, “After10Seconds”, “Frontline>=2 Tanks”)
- Avoid hardcoding trait names in ability code; use tag queries.

---

## 8) GameplayCues
- Cues are presentation only.
- No gameplay logic in cues.
- Abilities trigger cues through standardized cue tags.

---