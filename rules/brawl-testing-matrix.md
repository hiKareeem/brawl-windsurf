---
trigger: model_decision
description: Testing matrix for automation and headless combat
globs: 
---

# Testing Matrix (Automation + Headless Combat)

Goal: keep combat/economy stable as content grows (58+ units).

---

## 1) Required automation tests (minimum set)
### Combat math
- Damage sanity:
  - physical vs special uses correct stats
  - clamping prevents negative/NaN
- element multiplier table applies
- STAB applies
- Mixed/True semantics covered
- Cooldown scaling:
  - higher Spd reduces cooldown per CombatMath contract
  - clamp behavior at extremes

### Core rules
- Trigger ordering:
  - OnHit events logged before OnDamaged before OnDeath for a controlled scenario
- Placement rules:
  - swap-on-drop in planning
  - field locked in combat
  - move/swap snap + phase lock
- Overflow:
  - overflow spawn at (0,0)
  - end of planning destroy+refund prevents combat participation
- Traits:
  - ActiveCounts (field) vs PotentialCounts (field+bench) computed correctly
  - UI format can show `Active(Potential)/Threshold`

### Economy
- bench full prevents shop purchase
- shared pool consume/return invariants

---

## 2) Headless simulation tests
- Spawn N units per side (including worst-case approximations)
- Run for 60 seconds
- Assert:
  - no NaNs
  - no runaway memory growth
  - expected deaths/casts occur (within tolerance)
- Any RNG must be seeded and recorded.

---

## 3) Test maps (editor)
- LVL_Sandbox map: combined sandbox for early placement + combat testing (used by automation tests).
- (Optional later) CombatSandbox map: quick pit comps against each other, show debug overlays, run time-scale controls.
- (Optional later) EconomySandbox map: shop/bench/placement flows with deterministic offers.

---