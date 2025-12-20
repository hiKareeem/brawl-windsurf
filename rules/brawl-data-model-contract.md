---
trigger: model_decision
description: Data model
globs: 
---

# Data Model Contract (Assets, Tables, and Runtime Mapping)

Systems in C++; content authored in DataAssets / GAS assets / Blueprint. This document defines required asset schemas and how they map to runtime.

---

## 1) Asset identity strategy
- Use PrimaryAssetIds for Units/Abilities/Items/Traits so references are stable and serializable.
- Runtime objects refer to content by ID, not by raw pointers when possible (supports replay/logging/debug).
- Project config must enable AssetManager scanning for these PrimaryDataAssets so `FPrimaryAssetId` can be resolved/loaded at runtime (e.g., `[/Script/Engine.AssetManagerSettings]` + `PrimaryAssetTypesToScan` entries).

---

## 2) Required DataAssets

Implementation requirement:
- `UBrawlUnitData` and `UBrawlAbilityData` must derive from `UPrimaryDataAsset`.
- Runtime/replication should refer to these assets by `FPrimaryAssetId` (stable + serializable).

### UnitData (UBrawlUnitData)
Required fields:
- Id (PrimaryAssetId)
- Cost (1–5), Rarity
- Base gameplay tags:
  - Trait tags (`Trait.*`)
  - Element tag(s) (`Element.*`)
  - Faction tag (`Faction.Player` / `Faction.Creep`)
- Stat curves (CurveTable refs):
  - per-star scaling for HP/Atk/SpAtk/Def/SpDef/Spd
- Ability options:
  - 3 Basic Ability references (AbilityData ids)
  - 3 Ultimate Ability references (AbilityData ids)
- Visuals:
  - Unit BP class / skeletal mesh / anim set (as needed)

### AbilityData (UBrawlAbilityData)
Required fields:
- Id (PrimaryAssetId)
- Ability kind: Basic or Ultimate
- Cooldown base seconds
- Base power (numeric)
- Damage class tag: `DamageClass.Physical`, `DamageClass.Special`, `DamageClass.Mixed`, or `DamageClass.True`
- Element tag (`Element.*`, from the GameplayTags Contract list)
- Targeting policy reference (TargetingPolicy DataAsset)
- Projectile policy:
  - None or ProjectileData reference (travel time impacts gameplay)
- Energy:
  - Energy gained (Basic) and/or energy cost (Ultimate) (design-owned)

### ItemData (UBrawlItemData)
Required fields:
- Id (PrimaryAssetId)
- Unique item flag
- GameplayEffects to apply (to unit and/or team)
- Abilities to grant (optional)
- Tags to add (optional)

### TraitData (UBrawlTraitData)
Required fields:
- Id (PrimaryAssetId)
- Display name/description (localizable)
- Thresholds list:
  - threshold N -> list of effects
Effects may include:
- Apply GameplayEffect(s) to team/units
- Grant ability(s)
- Add/remove tags
- Conditional rules (first cast, after X seconds, etc.) via a standard “Condition” asset/policy

Trait counting rules:
- ActiveCounts = field only (effects applied)
- PotentialCounts = field + bench (UI only, shown like `2(3)/3`)

### RoundData / RoundSetData
Required fields:
- Round index / identifier
- Round type (CombatPvP / CombatPvE / Shop / ItemShop, etc.)
- Planning duration seconds (BP editable)
- Combat duration seconds (BP editable; may be 0)
- Reward definition reference
- Creep wave reference (for PvE rounds)

### Economy tuning
- Interest table
- Streak reward table
- XP per buy and level thresholds
- Roll odds by level (designer-owned)

Shared pool:
- Pool counts per unit type and rules for consume/return

---

## 3) Runtime mapping rules
- Unit spawn:
  - The server initializes base combat/energy/speed attributes immediately after `ASC->InitAbilityActorInfo(...)` once `UnitData` is known.
  - Base values are derived from UnitData stat curves using the unit’s current `StarLevel`.
  - Pooled defaults (v1):
    - `CurrentHealth` starts at `MaxHealth`.
    - `CurrentEnergy` starts at `0` (unless explicitly overridden by a round/trait/effect).
  - Implementation note:
    - Prefer setting ASC base values directly (e.g., `SetNumericAttributeBase`) over authoring per-unit “default attribute” GameplayEffects, to keep “add a new unit” authoring lightweight and consistent.
- Traits:
  - Trait component recomputes counts on roster changes and applies effects for ActiveCounts only
- Economy:
  - Shop offers are explicit replicated entries; offer generation is server-only
- Ability execution:
  - AbilityData.BasePower is passed into damage effects via SetByCaller `Data.Power`.
  - AbilityData.CooldownBaseSeconds is passed into cooldown effects via SetByCaller `Data.CooldownBaseSeconds` (then scaled by the global cooldown MMC).
  - Optional standardized modifiers may be passed via `Data.Mod` and `Data.Flat` (see Combat Math + GameplayTags contracts).

---

## 4) “Add a new unit” checklist (must remain simple)
- Create UnitData
- Fill tags, curves, ability options
- Assign a unit BP visual (derived from ABrawlUnitCharacter)
- Ensure pool entry exists (counts)
- Optional: add trait/item interactions via tags (no code)

---