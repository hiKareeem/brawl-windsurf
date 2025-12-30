---
trigger: always_on
description: 
globs: 
---

# Gameplay Tags Contract (Namespaces & Authority)

## Purpose
This document defines:
- the **authoritative GameplayTag namespaces**
- what each namespace means
- who is allowed to create or reference tags within them

GameplayTags are a shared language.  
Uncontrolled creation breaks determinism, balance, and tooling.

---

## 1) Hard Rules (Non-Negotiable)

- All tags must belong to a defined namespace.
- Namespaces are owned; ownership is exclusive.
- Do not invent new namespaces without updating this document.
- Tags describe **state or intent**, not behavior.
- Tags must be stable across builds and replays.

If a required concept does not fit an existing namespace: STOP.

---

## 2) Canonical Namespaces

### `Unit.*`
Describes unit identity and classification.

Examples:
- `Unit.Class.Melee`
- `Unit.Size.Large`

Owned by:
- Unit data authoring

Must NOT be added dynamically.

---

### `Ability.*`
Describes ability identity and category.

Examples:
- `Ability.Basic`
- `Ability.Ultimate`

Owned by:
- Ability data

Used for:
- filtering
- gating
- UI

---

### `DamageClass.*`
Describes damage semantics.

Examples:
- `DamageClass.Physical`
- `DamageClass.Special`

Owned by:
- Combat math contract

Must map cleanly into ExecCalcs.

---

### `Element.*`
Describes elemental typing.

Examples:
- `Element.Fire`
- `Element.Water`

Rules:
- Exactly one per damage instance
- Exactly one per defender

Owned by:
- Combat math + data model

---

### `Trait.*`
Describes trait membership.

Examples:
- `Trait.Assassin`
- `Trait.Mage`

Owned by:
- Trait data

Traits must never be inferred implicitly.

---

### `Item.*`
Describes item identity and category.

Examples:
- `Item.Weapon`
- `Item.Support`

Owned by:
- Item data

---

### `State.*`
Describes transient runtime state.

Examples:
- `State.Stunned`
- `State.Invulnerable`

Rules:
- Applied dynamically
- Must have clear lifetime
- Must be removable deterministically

Owned by:
- Runtime systems

---

### `Event.*`
Describes logged events.

Examples:
- `Event.Combat.Damage`
- `Event.Unit.Death`

Owned by:
- Event log schema

Used only for logging and replay.

---

## 3) Reserved / Restricted Tags

These tags (and subtrees) are **reserved**:
- `Internal.*`
- `Debug.*`
- `Temp.*`

They must never affect gameplay logic.

---

## 4) Ownership vs Consumption

- Owning a namespace means:
  - defining tags
  - approving additions
- Consuming a namespace means:
  - querying tags
  - reacting to tags

Most systems **consume**, few **own**.

---

## 5) Forbidden Patterns

- Creating tags at runtime
- Encoding logic in tag names
- Using tags as numeric values
- Using tags to bypass contracts
- Cross-namespace overloading

---

## 6) Adding New Tags (Required Procedure)

To add a new tag:
1. Identify existing namespace
2. Verify owner
3. Add tag to central tag list
4. Update references if required

STOP if:
- a new namespace seems necessary
- a tag replaces a stat
- a tag implies new behavior

---

## References
- Combat math: `brawl-combat-math-contract.md`
- Ability authoring: `brawl-ability-authoring-guide.md`
- Data model: `brawl-data-model-contract.md`
- Event logging: `brawl-event-log-schema.md`