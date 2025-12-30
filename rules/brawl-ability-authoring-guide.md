---
trigger: model_decision
description: Ability authoring guide
globs: 
---

# Ability Authoring Guide (GAS-first, Data-driven)

## Purpose
This document defines **how abilities must be authored** so they remain:
- modular
- designer-driven
- deterministic
- compatible with centralized combat, replication, and replay systems

It does **not** define combat math, replication rules, or economy behavior.  
Those are owned by their respective contracts.

---

## 1) Hard Rules (non-negotiable)

- Abilities **must not** calculate final damage or healing.
- Abilities **must not** implement bespoke cooldown, energy, or stat logic.
- Abilities **must not** emit duplicate combat events.
- Abilities **must** be deterministic given identical inputs.
- Presentation (GameplayCues, FX) **must never** affect gameplay.

If a rule cannot be expressed with existing base systems, escalate — do not hack around it.

---

## 2) Required Structure

Every ability consists of:
- **AbilityData** (`UBrawlAbilityData`)
  - numbers, tags, targeting policy, projectile policy
- **GameplayAbility class**, derived from:
  - `UBrawlGA_BasicBase`
  - `UBrawlGA_UltimateBase`

Authoring logic belongs in:
- base classes
- standardized helpers
- ExecCalcs / MMCs (for math)

Never in one-off ability subclasses.

---

## 3) Damage & Healing Inputs

Abilities provide **inputs only**. Final values are computed centrally.

### Required inputs
- `Data.Power` (SetByCaller)
- `DamageClass.*` tag
- `Element.*` tag

### Optional standard modifiers
- `Data.Mod` — multiplicative (default 1.0)
- `Data.Flat` — additive (default 0.0)

All damage math is owned by:
- `ExecCalc_Damage`

See:
- `brawl-combat-math-contract.md`

---

## 4) Cooldown Authoring

When applying a cooldown GameplayEffect:
- Set `Data.CooldownBaseSeconds` from `AbilityData.CooldownBaseSeconds`
- Global Spd scaling is applied centrally via MMC

Abilities must not scale cooldowns directly.

---

## 5) Targeting Policies

Each ability references a **TargetingPolicy** DataAsset.

A TargetingPolicy defines:
- selection mode (nearest, lowest HP, cone, etc.)
- eligibility tag queries
- behavior on target death:
  - `Retarget`
  - `Fizzle`
  - `LastPosition`

### Determinism requirements
- Target resolution must be deterministic.
- Never rely on iteration order.
- Tie-breaking rule:
  - choose lowest `FBrawlUnitInstanceId`
- Use a consistent epsilon (`KINDA_SMALL_NUMBER`) for distance ties.

### `LastPosition` invariant
- Snapshot target position **at target resolution time**
- Never re-evaluate at impact time
- Snapshot must be logged via `Combat.TargetChosen` (or projectile spawn)

---

## 6) Central Combat Driver Integration

Combat may be driven centrally (not per-unit tick).

When this is active:
- The server may resolve the target **before** ability activation
- `Combat.TargetChosen` and `Combat.AbilityCast` may already be emitted

### Locked target behavior
If the avatar implements `IBrawlCombatTargetProviderInterface`:
- Use the locked target **only if** `LockedAbilityId == AbilityId`
- Do **not** re-emit:
  - `Combat.TargetChosen`
  - `Combat.AbilityCast`

If no valid lock exists:
- Resolve targeting normally
- Emit events normally

---

## 7) Projectiles

Use a projectile actor **only when travel time matters**.

### Projectile invariants
- Server-authoritative
- Impact applies effects/damage
- Never retarget mid-flight

### Event requirements (server-only)
Emit exactly once:
- `Combat.ProjectileSpawned`
- `Combat.ProjectileImpacted`

All timestamps must use:
- `FBrawlEventBase.ServerTimeSeconds`

---

## 8) On-Hit, Auras, Persistent Effects

Use standardized patterns only:

- **On-hit**
  - tag-driven GE triggers
  - standardized on-hit hooks
- **Auras**
  - periodic GE application via shared aura pattern
- **Persistent zones**
  - standardized server-authoritative AOE actor

Do not invent custom per-ability frameworks.

---

## 9) Trait & Item Interactions

Complex conditions must use **one consistent mechanism**:
- tag-driven listeners
- standardized condition assets (e.g., FirstCast, AfterXSeconds)

Do not hardcode trait or item names in ability logic.

---

## 10) GameplayCues

- Presentation only
- No gameplay logic
- Triggered via standardized cue tags

---

## 11) GameplayEffects

GameplayEffects may be authored in Blueprint.

If a new GE pattern is needed:
- specify required fields
- request it explicitly
- do not clone existing effects ad hoc

---

## References
- Combat math: `brawl-combat-math-contract.md`
- Target tags: `brawl-gameplay-tags-contract.md`
- Event schema: `brawl-event-log-schema.md`
- Replication rules: `brawl-replication-contract.md`
