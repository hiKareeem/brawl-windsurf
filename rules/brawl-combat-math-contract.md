---
trigger: model_decision
description: Combat math
globs: 
---

# Combat Math Contract (ExecCalcs + Global Policies)

This document defines the v1 canonical formulas. Designers may tune constants via DataAssets, but logic lives in central ExecCalcs/MMCs.

## Implementation status (as of 2025-12-16)

Current shipped behavior in `ExecCalc_Damage`:
- Supports `DamageClass.Physical` vs `DamageClass.Special` stat selection (Physical is the default when Special is not explicitly present).
- Applies canonical Ratio clamp (`MinRatio`/`MaxRatio`) and final damage clamp (`MinDamage`/`MaxDamage`).
- Honors `State.Immune.Damage` (final damage becomes 0).

Not implemented yet (contract remains the target design):
- Element effectiveness multiplier table (`ElemMult`)
- STAB multiplier
- `DamageClass.Mixed` semantics
- `DamageClass.True` semantics
- Additional multiplicative (`Mod`) / flat (`Flat`) modifiers

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

---

## 3) Canonical v1 damage formula (explicit)
Let:
- A = (Atk if Physical else SpAtk)
- D = (Def if Physical else SpDef)
- Power = ability base power (>= 0)
- ElemMult = element effectiveness multiplier (defaults to 1.0 unless a rule table says otherwise)
- Mod = product of all multiplicative modifiers from tags/effects (defaults to 1.0)
- Flat = sum of flat add/sub modifiers (defaults to 0)

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
  - Ignores defensive mitigation from Def/SpDef (treat the mitigation term as neutral for v1).
  - Element effectiveness/STAB behavior is policy-driven; default is “no element effectiveness” (multiplier 1.0) unless explicitly enabled by tuning.

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

---