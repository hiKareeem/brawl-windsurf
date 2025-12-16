---
trigger: model_decision
description: Performance budgets 60 FPS target
globs: 
---

# Performance and Replication Budgets (60 FPS Target)

Worst-case target: 8 players, level 10, full benches => ~160 unit actors in match.

---

## 1) High-level performance rules
- Avoid per-unit Tick where possible.
  - Prefer ability tasks/timers and centralized schedulers for AI decisions.
- Avoid blueprint tick logic in core combat systems.
- Use stable IDs and stable ordering; do not sort every frame.

---

## 2) Replication rules of thumb
- Use ReplicationGraph to limit relevancy.
- Prefer FastArray replication for:
  - board occupancy maps
  - shop offer lists
  - trait counts summaries
- Do not replicate derived values every time they change unless UI requires it.
- GAS replication:
  - keep tag/attribute replication modes conservative
  - avoid replicating transient cosmetic data through GAS

---

## 3) Bandwidth priorities
Highest priority to replicate:
- phase/round state (GameState)
- placement state changes (planning)
- ability casts that matter to readability (if not already inferred)
- death events / alive state

Lower priority:
- fine-grained movement for off-screen boards (handled by relevancy)
- debug-only AI decisions (use Event Log + dev toggles)

---

## 4) Debug tooling without perf collapse
- Debug overlays (threat/target traces) must be behind cvars and default off.
- Event log can be buffered; file export should be optional.

---