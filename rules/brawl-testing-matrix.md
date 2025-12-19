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
- Mod/Flat:
  - `Data.Mod` multiplies damage and `Data.Flat` adds after multipliers
  - Mixed applies Mod/Flat once (not per-branch)
  - True applies Mod/Flat but still ignores STAB/element
- element multiplier table applies
- STAB applies
- Mixed/True semantics covered
- Cooldown scaling:
  - higher Spd reduces cooldown per CombatMath contract
  - clamp behavior at extremes

### Core rules
- Trigger ordering:
  - OnHit events logged before OnDamaged before OnDeath for a controlled scenario
- Projectile event lifecycle:
  - For any spawned projectile: assert exactly one `Combat.ProjectileImpacted` exists with the same `projectileId` (within the same `MatchId`).
  - Assert `ImpactTimeSeconds >= SpawnTimeSeconds` and that both are in the same timebase as `ServerTimeSeconds`.
  - Assert miss/expiry still emits `ProjectileImpacted` with invalid/unset `impactedTargetId` (but a valid `impactPos` when possible).
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

### Match / RPC validation
- EquipAbilities RPC ([ServerRequestEquipUnitAbilities](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Game/BrawlPlayerController.h:37:1-37:134)):
  - Allowed phases: `Phase.Planning`, `Phase.ItemShop`
  - Rejected phases: `Phase.Combat`, `Phase.Rewards`
  - Reject invalid `UnitInstanceId`, non-option ability IDs, and wrong-kind ability IDs
  - Assert equipped ability IDs update on success and remain unchanged on rejection
  - Assert ASC ability specs grant/clear behavior is correct (no duplication across re-equip)

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

### Automation authoring constraints (Dedicated Server)
- Automation tests must be compatible with dedicated-server execution.
- Do not use `UGameplayStatics::CreatePlayer` in automation tests (dedicated servers cannot have local players).
- If a test needs a PlayerController/PlayerState, spawn the server-side [ABrawlPlayerController](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Game/BrawlPlayerController.cpp:20:0-23:1), call `InitPlayerState()`, and manually wire only the minimal required match state (e.g., BoardActor/Unit ownership) for the scenario.

---