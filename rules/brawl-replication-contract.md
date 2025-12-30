---
trigger: always_on
description: 
globs: 
---

# Replication Contract (Authoritative State & Replay)

## Purpose
This document defines:
- what state is replicated
- who owns it
- how replay and spectate are supported

If state is not explicitly allowed to replicate here, it must not replicate.

---

## 1) Hard Rules (Non-Negotiable)

- The server is the sole authority for gameplay state.
- Clients send **requests only**, never outcomes.
- Derived state is **never replicated**.
- Replay correctness does **not** rely on re-simulation.
- Replication shape must be deterministic and stable.

If a system cannot meet these rules, it must emit events instead.

---

## 2) Replication Categories

Every piece of gameplay state must belong to **exactly one** category.

### A) Replicated Authoritative State
- Required for live gameplay
- Required for UI
- Required for spectate

### B) Derived / Local State
- Computed from authoritative inputs
- Never replicated
- Recomputed client-side

### C) Event-Only State
- Logged once
- Replayed from EventLog
- Not re-simulated

---

## 3) Canonical Replicated State

### Match-Level
Owner: `ABrawlGameState`

Replicates:
- match phase
- round index
- timers (authoritative)
- player readiness

Does NOT replicate:
- transient countdowns
- UI timers
- derived phase flags

---

### Player-Level
Owner: `ABrawlPlayerState`

Replicates:
- gold
- XP
- level
- bench capacity
- cosmetic selections

Does NOT replicate:
- shop RNG rolls
- pending purchase intents

---

### Unit-Level
Owner: replicated unit runtime structs

Replicates:
- unit instance ID
- current HP
- energy
- grid coordinate
- alive/dead state

Does NOT replicate:
- cooldown timers (derive from timestamps)
- targeting decisions (event-driven)
- damage breakdowns

---

### Board / Grid
Owner: grid subsystem authoritative state

Replicates:
- unit → grid mapping
- bench vs field location

Does NOT replicate:
- world transforms
- interpolation state

---

## 4) Gameplay Ability System (GAS)

Rules:
- Attributes replicate only if required by UI
- Cooldowns are derived from:
  - server timestamps
  - base cooldown data
- GameplayEffects replicate only when:
  - long-lived
  - player-visible

No per-ability replication exceptions.

---

## 5) EventLog vs Replication

### Must be Replicated
- State required to make decisions
- State required for UI correctness

### Must be Logged (EventLog)
- Damage dealt
- Healing applied
- Target chosen
- Ability cast
- Projectile spawned / impacted
- Unit death

### Must NOT be Both
A value is either:
- replicated **or**
- logged

Never both.

---

## 6) Replay & Spectate

Replay is driven by:
- replicated state snapshots
- ordered EventLog entries

Rules:
- No combat re-simulation
- No RNG replay
- No client-side inference

Spectate uses the same pipeline.

---

## 7) Replication Graph Requirements

- ReplicationGraph is mandatory
- Spatial culling must respect grid ownership
- Event-only data must not enter replication channels

---

## 8) Forbidden Patterns

- Replicating damage numbers
- Replicating RNG seeds to clients
- Client-authored attributes
- Tick-driven replication for combat
- Re-simulating combat for replay

---

## 9) Ownership Summary

| State Type        | Owner              | Transport |
|-------------------|--------------------|-----------|
| Match flow        | GameState          | Replicate |
| Economy           | PlayerState        | Replicate |
| Unit core state   | Runtime structs    | Replicate |
| Combat outcomes   | Combat system      | EventLog  |
| Targeting         | Combat system      | EventLog  |
| Cooldowns         | GAS/MMC            | Derived   |

---

## References
- Event schema: `brawl-event-log-schema.md`
- Validation: `brawl-server-validation-checklist.md`
- Performance: `brawl-performance-budgets.md`