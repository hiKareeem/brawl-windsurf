---
trigger: model_decision
description: When working on player character or camera
globs: 
---

# Player Avatar & Camera Contract (Non-Gameplay Authority)

## Purpose
This document defines:
- the player avatar (GASP-based)
- camera systems (GameplayCameras)
- how players view and navigate the board

These systems are **non-authoritative** and must not affect gameplay outcomes.

---

## 1) Hard Rules (Non-Negotiable)

- Player avatars have **no gameplay authority**.
- Cameras never determine legality, targeting, or outcomes.
- All gameplay decisions remain server-authored.
- Visual state must never gate input validity.

If a camera or avatar affects gameplay logic: STOP.

---

## 2) Player Avatar (GASP)

The player avatar exists for:
- presence
- identity
- cosmetics
- navigation

Rules:
- Uses GASP locomotion
- Cannot block units or projectiles
- Cannot affect visibility or targeting
- Cannot interact with grid legality

Avatar state is cosmetic-only.

---

## 3) Avatar Replication

Replicates:
- transform (for presence)
- cosmetic selections

Does NOT replicate:
- collision authority
- gameplay-relevant state

Avatar replication is excluded from replay logic.

---

## 4) Camera Systems

Uses:
- GameplayCameras

Camera modes may include:
- board overview
- player-follow
- scouting view

Rules:
- Camera transitions are client-side
- Camera selection does not affect game phase
- No camera-driven input restrictions

---

## 5) Board View & Scouting

Scouting:
- is a **view-only** action
- does not grant hidden information
- respects fog-of-war rules (if any)

Board mirroring:
- is visual-only
- does not alter canonical grid coordinates

---

## 6) Teleportation & Navigation

Teleporting the avatar:
- is cosmetic
- does not imply player intent
- does not trigger gameplay events

No teleport may:
- move units
- lock targets
- trigger abilities

---

## 7) Input Semantics

Player inputs related to:
- movement
- camera control
- emotes

Are processed locally.

Gameplay inputs (placement, purchase, etc.) are:
- explicit requests
- validated server-side
- unrelated to camera state

---

## 8) Forbidden Patterns

- Camera-based targeting
- Avatar collision affecting units
- Hidden gameplay info via camera
- Camera-driven phase logic

---

## References
- Grid rules: `brawl-grid` (via `brawl-directories.md`)
- Replication: `brawl-replication-contract.md`
- Cosmetics: `brawl-cosmetics-contract.md`