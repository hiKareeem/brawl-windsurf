---
trigger: always_on
description: 
globs: 
---

# Directories & Module Ownership (Authoritative)

## Purpose
This document defines the **only allowed module, directory, and class ownership structure** for Brawl.

If something is not listed here, it must not be created without updating this document first.

This document is **higher precedence** than all feature-level guides.

---

## 1) Hard Rules (Non-Negotiable)

- Do not create new modules.
- Do not rename modules or public classes.
- Do not move classes across modules.
- Do not introduce cross-module dependencies outside those listed.
- If unsure where something lives: STOP.

---

## 2) Module Overview

| Module                | Responsibility                                  |
|-----------------------|-------------------------------------------------|
| `BrawlCore`           | Shared primitives, math, IDs, tags              |
| `BrawlGameplay`       | Match rules, phases, combat flow                |
| `BrawlUnits`          | Units, traits, items, abilities (data + logic) |
| `BrawlGrid`           | Grid, placement, mirroring, legality            |
| `BrawlEconomy`        | Shop, gold, XP, pools                           |
| `BrawlUI`             | UI only (no gameplay logic)                     |
| `BrawlCosmetics`      | Non-interactive cosmetics                       |
| `BrawlReplay`         | Replay, spectate, event ingestion               |

No other gameplay modules are permitted.

---

## 3) Core Types (`BrawlCore`)

Owns:
- `FBrawlUnitInstanceId`
- `FBrawlGridCoord`
- combat enums
- tag constants
- shared math helpers

Does NOT own:
- gameplay rules
- replication state
- match flow

---

## 4) Gameplay Flow (`BrawlGameplay`)

Owns:
- match phases
- combat orchestration
- round resolution
- authoritative sequencing

Key classes:
- `ABrawlGameState`
- `UBrawlPhaseSubsystem`
- `UBrawlCombatDriver`

Does NOT own:
- unit stats
- targeting logic
- economy math

---

## 5) Units & Abilities (`BrawlUnits`)

Owns:
- unit runtime state
- abilities
- traits
- items
- GAS integration

Key bases:
- `UBrawlGA_BasicBase`
- `UBrawlGA_UltimateBase`
- `UBrawlAbilityData`
- `UBrawlUnitData`

Does NOT own:
- grid legality
- shop RNG
- replay systems

---

## 6) Grid & Placement (`BrawlGrid`)

Owns:
- canonical grid coordinates
- placement legality
- mirroring rules
- bench vs field

Key types:
- `FBrawlGridCoord`
- `UBrawlGridSubsystem`

Rules:
- Grid coords are authoritative
- Actor transforms are secondary
- All legality checks live here

---

## 7) Economy (`BrawlEconomy`)

Owns:
- gold
- XP
- shop generation
- shared pools

Key systems:
- shop RNG (seeded)
- purchase validation
- overflow handling

Does NOT own:
- unit stats
- combat math
- GAS logic

---

## 8) UI (`BrawlUI`)

Owns:
- widgets
- HUD
- presentation state

Rules:
- UI never commits gameplay state
- UI reads replicated state only
- No gameplay authority

---

## 9) Cosmetics (`BrawlCosmetics`)

Owns:
- avatar skins
- boards
- emotes
- VFX/SFX swaps

Rules:
- No gameplay impact
- No stat changes
- No targeting or visibility changes

---

## 10) Replay & Event Ingestion (`BrawlReplay`)

Owns:
- event ingestion
- replay playback
- spectate support

Rules:
- Replays do not re-simulate combat
- Events are authoritative
- Ordering is preserved

---

## 11) Dependency Rules

Allowed dependencies (one-way):

Core → (all)
Gameplay → Core
Units → Core
Grid → Core
Economy → Core
UI → Gameplay/Core
Cosmetics → Core
Replay → Gameplay/Core


Reverse dependencies are forbidden.

---

## 12) Forbidden Patterns

- Feature-specific submodules
- “Helper” modules
- Circular dependencies
- Copying logic across modules
- One-off systems inside feature folders

---

## References
- Replication: `brawl-replication-contract.md`
- Data models: `brawl-data-model-contract.md`
- Combat math: `brawl-combat-math-contract.md`