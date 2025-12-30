---
trigger: model_decision
description: Combat math
globs: 
---

# Combat Math Contract (Canonical & Centralized)

## Purpose
This document defines the **only authoritative combat formulas** in Brawl and **where they are implemented**.

If a number affects damage, healing, mitigation, cooldowns, or movement speed, it must be derived from rules in this document and implemented via the specified calculation path.

This contract is higher precedence than:
- Ability authoring
- Traits/items
- Designer expectations

---

## 1) Hard Rules (Non-Negotiable)

- Abilities provide **inputs only**, never final values.
- All combat math is centralized in:
  - **ExecCalcs** (one-shot resolution)
  - **MMCs** (derived attributes)
- No combat math may live in:
  - GameplayAbilities
  - Actors
  - Components
  - Blueprint graphs
- No uncontrolled randomness.
- No per-ability math forks.

If a rule is not defined here, it does not exist.

---

## 2) Damage Resolution Pipeline

**Canonical order** (cannot be reordered):

1. Base Power (SetByCaller)
2. Source Stat Scaling
3. Target Defense Mitigation
4. Element Multiplier
5. Global Modifiers
6. Clamp / Floor
7. Result Application

All steps occur inside `ExecCalc_Damage`.

---

## 3) Base Damage Formula

BaseDamage =
Power

    SourceAttackScalar

    ElementMultiplier

    GlobalDamageScalar


Where:
- `Power` = `Data.Power` (required SetByCaller)
- `SourceAttackScalar` derives from attacker stats
- `ElementMultiplier` derives from element matchup
- `GlobalDamageScalar` includes buffs/debuffs

No step may be skipped.

---

## 4) Attack vs Defense Mitigation

Mitigation uses a **Pokémon-style diminishing returns curve**.

MitigatedDamage =
BaseDamage * ( Attack / (Attack + Defense) )


Properties:
- Always asymptotic (never reaches 0)
- Stable under large stat values
- No caps unless explicitly added here

This logic lives exclusively in:
- `ExecCalc_Damage`

---

## 5) Element Interaction

- Exactly one **Element tag** per damage instance
- Exactly one **Element tag** per defender

Multiplier lookup:
- Defined by element chart DataAsset
- Resolved inside `ExecCalc_Damage`

Rules:
- No stacking elements
- No dynamic element mutation mid-resolution
- No per-ability overrides

---

## 6) Healing Formula

Healing mirrors damage **without mitigation**.

FinalHealing =
Power

    SourceHealingScalar

    GlobalHealingScalar


- No element interactions
- No defense scaling
- Implemented in `ExecCalc_Healing`

---

## 7) Crits & Randomness

There are **no crits** unless explicitly added here.

- No % rolls
- No bonus variance
- No per-hit randomness

If crits are introduced in the future:
- They must be deterministic
- They must be logged
- This document must be updated first

---

## 8) Speed (Spd) Policies

### Cooldown Scaling
- Spd reduces cooldown multiplicatively
- Scaling is applied via MMC
- Abilities supply **base cooldown only**

Example:

FinalCooldown = BaseCooldown / SpdScalar


No ability may modify its own cooldown beyond base input.

---

### Movement Speed
- Spd contributes to MoveSpeed via MMC
- Actor tick rate must not affect resolution
- No per-ability movement math

---

## 9) Global Modifiers

Global modifiers include:
- Buffs / debuffs
- Trait effects
- Item effects

Rules:
- Applied multiplicatively unless stated otherwise
- Order-independent
- Resolved centrally

No modifier may directly alter final damage outside ExecCalcs.

---

## 10) Clamping & Floors

- Damage floors at `>= 1` unless explicitly specified otherwise
- Healing floors at `>= 0`
- Clamps occur **after** all multipliers

---

## 11) Ownership Summary

| Concern              | Owner               |
|----------------------|---------------------|
| Damage math          | ExecCalc_Damage     |
| Healing math         | ExecCalc_Healing    |
| Defense mitigation   | ExecCalc_Damage     |
| Element multipliers  | ExecCalc_Damage     |
| Cooldown scaling     | MMC                 |
| MoveSpeed scaling    | MMC                 |

If ownership is unclear, stop.

---

## References
- Ability inputs: `brawl-ability-authoring-guide.md`
- Tags: `brawl-gameplay-tags-contract.md`
- Events: `brawl-event-log-schema.md`
- Validation: `brawl-server-validation-checklist.md`