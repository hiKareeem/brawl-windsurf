---
trigger: model_decision
description: Server validation checklist for competitive integrity
globs: 
---

# Server Validation Checklist (Authoritative Gate)

## Purpose
This document defines **mandatory server-side validation** for all client-authored requests.

Clients may request actions.  
The server decides whether they occur.

If validation fails, the request is rejected with **no side effects**.

---

## 1) Hard Rules (Non-Negotiable)

- Every client request is validated server-side.
- Validation occurs **before** any state mutation.
- Failed validation must not:
  - spend resources
  - mutate pools
  - alter state
- Validation logic is deterministic and auditable.

---

## 2) Request Categories

Every request must be categorized as one of:

- Placement / movement
- Economy (purchase, reroll, sell)
- Ability activation
- Phase readiness
- Cosmetic / emote (non-gameplay)

Unknown categories are rejected.

---

## 3) Universal Validation (All Requests)

The server must validate:
- request originates from owning player
- match is in a valid phase
- request payload is well-formed
- referenced IDs exist and are owned
- request is not duplicated or stale

---

## 4) Placement & Grid Requests

Validate:
- phase allows placement
- grid coordinates are legal
- destination is unoccupied
- unit belongs to requesting player
- bench/field rules are respected

Reject if any check fails.

---

## 5) Economy Requests

Validate:
- sufficient gold
- bench capacity (for purchases)
- pool availability
- phase legality

On rejection:
- gold is unchanged
- pools are unchanged

---

## 6) Ability Activation Requests

Validate:
- unit exists and is alive
- ability is off cooldown
- unit has sufficient energy (if applicable)
- phase allows combat actions

Target legality:
- resolved server-side
- client targets are treated as hints only

---

## 7) Phase & Ready Requests

Validate:
- request matches current phase
- player is eligible to signal readiness
- duplicate readiness is ignored

---

## 8) Cosmetic & Emote Requests

Validate:
- request affects no gameplay state
- rate limits are respected

Cosmetic requests must never fail gameplay validation paths.

---

## 9) Anti-Cheat Posture

The server assumes:
- clients are untrusted
- payloads may be malicious
- timing may be manipulated

Therefore:
- no client-provided values are trusted
- no client-side RNG is accepted

---

## 10) Forbidden Patterns

- Mutating state before validation
- Partial commits
- Trusting client cooldowns or energy
- Validation logic in UI or client code

---

## References
- Replication: `brawl-replication-contract.md`
- Economy: `brawl-economy-contract.md`
- Grid: `brawl-directories.md`
- Testing: `brawl-testing-matrix.md`
