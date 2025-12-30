---
trigger: model_decision
description: Data model
globs: 
---

# Data Model Contract (Authoritative Content Pipeline)

## Purpose
This document defines:
- which **DataAsset and table types are allowed**
- what each one owns
- how static data becomes runtime state

If content cannot be expressed using the models below, it must be escalated before implementation.

---

## 1) Hard Rules (Non-Negotiable)

- All gameplay content is **data-driven**.
- Runtime gameplay state is **never** stored in DataAssets.
- DataAssets are **read-only at runtime**.
- Do not invent new DataAsset types without updating this document.
- Identity is stable and explicit (no name-based lookups).

---

## 2) Canonical DataAsset Types

### `UBrawlUnitData`
Defines a unit archetype.

Owns:
- base stats
- default traits
- allowed ability slots
- visuals (via cosmetic indirection)

Does NOT own:
- runtime HP, energy, cooldowns
- economy cost resolution

---

### `UBrawlAbilityData`
Defines an ability configuration.

Owns:
- base power
- cooldown base
- targeting policy reference
- projectile policy reference
- gameplay tags

Does NOT own:
- damage math
- cooldown scaling
- stat formulas

See:
- `brawl-ability-authoring-guide.md`

---

### `UBrawlItemData`
Defines an equippable item.

Owns:
- granted tags
- granted modifiers
- passive hooks

Does NOT own:
- direct stat mutation logic
- bespoke ability logic

---

### `UBrawlTraitData`
Defines a trait breakpoint system.

Owns:
- thresholds
- granted effects per tier

Does NOT own:
- hardcoded unit or item logic

---

### `UBrawlTargetingPolicy`
Defines target selection behavior.

Owns:
- eligibility queries
- selection mode
- tie-breaking policy

See:
- `brawl-ability-authoring-guide.md`

---

### `UBrawlProjectilePolicy`
Defines projectile behavior.

Owns:
- travel speed
- lifetime
- impact behavior class

Does NOT own:
- damage math
- retargeting logic

---

## 3) Tables & Registries

### Master Registries
All DataAssets must be discoverable via:
- explicit registries
- or primary asset labels

No runtime scanning or soft discovery.

---

### ID Stability
Every gameplay-relevant asset must have:
- a stable ID
- never reused
- never inferred from asset name

---

## 4) Runtime Mapping

### Static → Runtime
At match start:
- DataAssets are read
- Runtime instances are constructed
- Runtime instances own mutable state

Examples:
- `UBrawlUnitData` → `FBrawlUnitRuntime`
- `UBrawlItemData` → `FBrawlItemRuntime`

DataAssets are never mutated or replicated.

---

## 5) Adding New Content (Required Procedure)

When adding **new gameplay content**:

1. Identify the correct existing DataAsset type
2. Add a new instance (not a new class)
3. Reference it via registry
4. Validate against:
   - tags contract
   - combat math contract
   - replication contract

STOP if:
- a new DataAsset type seems required
- a DataAsset needs runtime state
- a DataAsset must “know” about match flow

---

## 6) Forbidden Patterns

- Per-ability DataAssets with bespoke fields
- DataAssets that reference runtime actors
- Storing arrays of live units/items inside assets
- Implicit content discovery via asset paths
- Encoding gameplay logic in Blueprint-only assets

---

## 7) Ownership Summary

| Concern                | Owner                    |
|------------------------|--------------------------|
| Static config          | DataAssets               |
| Runtime state          | Replicated sim structs   |
| Math / scaling         | ExecCalcs / MMCs         |
| Target selection       | TargetingPolicy assets   |
| Projectiles            | ProjectilePolicy assets  |

---

## References
- Tags: `brawl-gameplay-tags-contract.md`
- Combat math: `brawl-combat-math-contract.md`
- Ability authoring: `brawl-ability-authoring-guide.md`
- Replication: `brawl-replication-contract.md`
