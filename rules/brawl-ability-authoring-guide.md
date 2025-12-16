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
  - `GA_BasicBase` or `GA_UltimateBase`

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

---

## 3) Targeting policies
Each ability references a TargetingPolicy that defines:
- target selection mode (nearest, lowest HP, backline, threat, cone, etc.)
- behavior on target death during cast:
  - Retarget / Fizzle / LastPosition
- eligibility tag queries (e.g., ignore `State.Dead`, ignore `State.Immune.CC`, etc.)

---

## 4) Projectiles
If travel time matters:
- Use projectile actor policy; impact applies effects/damage.
- Emit event log:
  - ProjectileSpawned
  - ProjectileImpacted

---

## 5) On-hit procs, auras, persistent effects
Preferred patterns:
- On-hit: apply a GE that triggers via tags/events, or use a standardized on-hit ability hook.
- Aura: periodic GE application driven by a standardized aura component/ability pattern.
- Persistent zones: standardized “AOE zone actor” pattern (server-authoritative) rather than custom per ability.

---

## 6) Conditional trait/item interactions
Complex conditions must be implemented using a consistent mechanism (pick one and stick to it):
- Tag-driven + standardized listeners (e.g., “FirstCast”, “After10Seconds”, “Frontline>=2 Tanks”)
- Avoid hardcoding trait names in ability code; use tag queries.

---

## 7) GameplayCues
- Cues are presentation only.
- No gameplay logic in cues.
- Abilities trigger cues through standardized cue tags.

---