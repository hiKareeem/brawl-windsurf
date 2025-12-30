---
trigger: model_decision
description: When working on BrawlEconomy or economy related tasks
globs: 
---

# Economy Contract (Deterministic & Server-Authoritative)

## Purpose
This document defines all economy rules:
- gold
- XP
- shop generation
- shared pools
- purchase resolution

All economy outcomes are **server-authored and deterministic**.

---

## 1) Hard Rules (Non-Negotiable)

- All economy state is owned by the server.
- Clients submit **requests only**.
- All randomness is:
  - server-only
  - seeded
  - reproducible
- Rejected actions must not spend resources.

---

## 2) Economy State Ownership

Owner: `ABrawlPlayerState`

Replicates:
- gold
- XP
- level
- bench capacity

Does NOT replicate:
- shop rolls
- pending actions
- RNG state

---

## 3) Gold Rules

- Gold is integer-only.
- Gold may only change via:
  - round income
  - interest
  - unit sales
  - rewards explicitly defined here

No floating-point accumulation.

---

## 4) XP & Leveling

- XP thresholds are fixed per level.
- XP overflow carries forward.
- Level-up is atomic:
  - XP deducted
  - level incremented
  - capacity updated

Partial level state must not exist.

---

## 5) Shop Generation

### Generation
- Shop rolls occur server-side only.
- RNG is seeded per-player per-round.
- Seed must be stable across reconnects.

### Pools
- Units are drawn from shared pools.
- Pool depletion is authoritative.
- Pool returns occur only on unit sale.

No local prediction of shop contents.

---

## 6) Purchase Validation

Before committing a purchase, the server must validate:
- sufficient gold
- available bench space
- unit availability in pool
- phase legality

If any check fails:
- purchase is rejected
- no gold is spent
- no pool mutation occurs

---

## 7) Commit Semantics

A purchase commit performs, in order:
1. deduct gold
2. mutate pool
3. create unit runtime instance
4. place on bench

This sequence is atomic.

---

## 8) Rerolls & Locking

- Rerolls consume gold only on success.
- Locked shops preserve the previous roll exactly.
- Lock state is authoritative and replicated.

---

## 9) Sales & Refunds

- Selling a unit:
  - returns gold
  - returns unit to pool
- Refund values are deterministic and fixed.

No partial refunds.

---

## 10) Forbidden Patterns

- Client-side shop generation
- Floating-point gold
- Predictive pool mutation
- UI-driven economy logic
- GAS-based economy state

---

## 11) Ownership Summary

| Concern          | Owner            |
|------------------|------------------|
| Gold / XP        | PlayerState      |
| Shop RNG         | Economy system   |
| Pools            | Economy system   |
| Validation       | Server           |
| Commit           | Server           |

---

## References
- Replication: `brawl-replication-contract.md`
- Validation: `brawl-server-validation-checklist.md`
- Testing: `brawl-testing-matrix.md`