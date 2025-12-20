---
trigger: model_decision
description: Combat math
globs: 
---

# Combat Math Contract (ExecCalcs + Global Policies)

This document defines the v1 canonical formulas. Designers may tune constants via DataAssets, but logic lives in central ExecCalcs/MMCs.

## Implementation status (as of 2025-12-16)

Current shipped behavior in `ExecCalc_Damage`:
- Supports `DamageClass.Physical`, `DamageClass.Special`, `DamageClass.Mixed`, `DamageClass.True`.
  - Physical/Special select Atk/Def vs SpAtk/SpDef.
  - Mixed splits 50/50 between Physical and Special ratio branches, combines deterministically, then clamps once.
  - TrueDamage = Power (Ratio = 1.0; ignores offense/defense). True currently ignores STAB and element effectiveness.
- Applies canonical Ratio clamp (`MinRatio`/`MaxRatio`) and final damage clamp (`MinDamage`/`MaxDamage`).
- Applies STAB from tuning (`StabMultiplier`) when source has the same `Element.*` tag as the ability’s element tag.
- Applies element effectiveness from tuning rules (`ElementEffectivenessRules`) keyed by ability element vs target element.
- Honors `State.Immune.Damage` (final damage becomes 0).

Also implemented:
- SetByCaller damage modifiers:
  - `Data.Mod` (optional, multiplicative, defaults to 1.0; non-finite treated as 1.0)
  - `Data.Flat` (optional, additive, defaults to 0.0; non-finite treated as 0.0)

Ordering (v1):
- For Physical/Special/Mixed:
  - `Raw = (BaseDamage * ElemMult * StabMult * Mod) + Flat`
- For True:
  - `Raw = (Power * Mod) + Flat` (True still ignores STAB and element effectiveness)

---

## 1) Stats schema (authoritative)
- HP, MaxHP
- Atk, SpAtk
- Def, SpDef
- Spd
- Energy, MaxEnergy

Resource pools are represented as current + max attributes (e.g., HP/MaxHP, Energy/MaxEnergy).

---

## 2) Damage formula ownership
- All damage is computed by `ExecCalc_Damage`.
- Abilities provide:
  - BasePower (numeric)
  - DamageClass tag: Physical/Special/Mixed/True (`DamageClass.*`)
  - Element tag (`Element.*`)
  - Optional modifiers via SetByCaller magnitudes and tags

### Standard SetByCaller keys (canonical)
These tags are the canonical numeric inputs used by central math policies.

Damage (`ExecCalc_Damage`):
- `Data.Power` (required): base power input
- `Data.Mod` (optional): multiplicative modifier, defaults to 1.0
- `Data.Flat` (optional): flat modifier, defaults to 0.0

Cooldown scaling (MMC):
- `Data.CooldownBaseSeconds` (required): base cooldown seconds before Spd scaling

---

## 3) Canonical v1 damage formula (explicit)
Let:
- A = (Atk if Physical else SpAtk)
- D = (Def if Physical else SpDef)
- Power = ability base power (>= 0)
- ElemMult = element effectiveness multiplier (defaults to 1.0 unless a rule table says otherwise)
- Mod = product of all multiplicative modifiers from tags/effects (defaults to 1.0)
- Flat = sum of flat add/sub modifiers (defaults to 0)
- StabMult = STAB multiplier (1.0 by default; `StabMultiplier` when source shares the ability’s `Element.*` tag)

2) Raw = (Power * Ratio * ElemMult * StabMult * Mod) + Flat

Compute:
1) Ratio = clamp(A / max(D, 1), MinRatio, MaxRatio)
2) Raw = (Power * Ratio * ElemMult * Mod) + Flat
3) FinalDamage = clamp(Raw, MinDamage, MaxDamage)

All clamps are tuning constants (DataAsset):
- MinRatio (default 0.25), MaxRatio (default 4.0)
- MinDamage (default 1), MaxDamage (default large)

## 3.1) Damage class semantics (v1)
Damage class is determined by a `DamageClass.*` tag on the damage GameplayEffect spec.

- `DamageClass.Physical`:
  - A = Atk, D = Def
- `DamageClass.Special`:
  - A = SpAtk, D = SpDef
- `DamageClass.Mixed`:
  - Split 50/50 between Physical and Special branches (compute both branches using the canonical formula and combine deterministically).
- `DamageClass.True`:
  - TrueDamage = Power (Ratio = 1.0; ignores Atk/SpAtk and Def/SpDef).
  - True currently ignores STAB and element effectiveness.

## 3.2) STAB (Same Type Attack Bonus)
- STAB is a multiplicative bonus applied when the source has the same `Element.*` tag as the ability’s element tag.
- Default STAB multiplier is designer-tunable (recommended default: 1.5).
- STAB must be applied centrally (ExecCalc), not in individual abilities.

Note:
- If you later add armor-pen/resists, do it inside ExecCalc_Damage using tags and SetByCaller values.

---

## 4) Element effectiveness
- Element rules are data-driven (table keyed by source `Element.*` vs target `Element.*` -> multiplier).
- Default multiplier is 1.0 when no rule exists.

---

## 5) Speed policy (global)
Spd affects movement and cooldowns with a single shared reference constant `SpdRef` (default 100).

### Move speed
- MoveSpeedMultiplier = clamp(Spd / SpdRef, MinMoveMult, MaxMoveMult)
- CharacterMovement MaxWalkSpeed = BaseMoveSpeed * MoveSpeedMultiplier

### Cooldown scaling
Canonical v1 semantics: higher Spd = faster cooldowns.
- CooldownMultiplier = clamp(SpdRef / max(Spd, 1), MinCDMult, MaxCDMult)
- EffectiveCooldownSeconds = BaseCooldownSeconds * CooldownMultiplier

Tuning clamps (DataAsset):
- MinMoveMult, MaxMoveMult
- MinCDMult, MaxCDMult (example: 0.25..4.0)

Cooldown scaling must be implemented centrally (MMC or a single shared function), not per ability.

---

## 6) Energy rules (v1)
- Basics generate energy on cast (and optionally on hit) via designer-authored values.
- Ultimate casts when Energy >= MaxEnergy (default behavior).
- Exact energy gain values are authored in AbilityData/GE, but the trigger rule is centralized.

Energy spend/gain application (v1):
- On successful cast, apply both cost and gain on the server:
  - `NewEnergy = clamp(CurrentEnergy - max(EnergyCost, 0) + max(EnergyGained, 0), 0..MaxEnergy)`
- Ultimate default trigger remains:
  - cast when `CurrentEnergy >= MaxEnergy` (then cost/gain are applied as above).

---