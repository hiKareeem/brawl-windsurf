---
trigger: model_decision
description: Cosmetics contract (avatar skins, board skins, emotes) — authority, replication, non-interactivity, data model.
globs: 
---

# Cosmetics Contract (Avatars, Boards, Emotes) — UE 5.7

This document defines the **cosmetics system surface area** that is visible during matches (and therefore interacts with replication/replay).
It exists to ensure cosmetics remain **presentation-only** and never interfere with **server-authoritative gameplay**, placement input, or determinism expectations.

---

## 1) Scope & release gating (informative, non-authoritative)

Target roadmap:
- 0.3: core gameplay loop stabilization
- 0.5: content expansion (units/abilities/traits)
- 0.7: public test (Season 0)
- 0.9: final balance pass
- 1.0: account progression + monetized cosmetics store

In-scope pre-0.7:
- Avatar skins (must support immediate Masculine/Feminine option)
- At least one board skin to validate non-interactivity
- Emotes are required by 0.7

Out-of-scope until ~1.0:
- Paid entitlements, premium currency, free currency vault rotation, store pricing rules
- Backend purchase flows and platform MTX rail decisions (Steam vs AWS)
- Battle pass and long-term account progression systems (separate contract)

---

## 2) Goals / non-goals

### Goals
- Cosmetics are **presentation-only** and must never affect gameplay outcomes.
- Cosmetic selection is **pre-match only**.
- The **server is authoritative** over what cosmetics are active for a player in a match.
- Everyone should be able to see everyone else’s cosmetics (players + spectators), subject to ReplicationGraph relevancy.

### Non-goals
- Cosmetics do not change:
  - placement legality
  - targeting
  - board layout dimensions
  - authoritative trace channels for input
  - damage, stats, or combat simulation
- Cosmetics are not required for match correctness or determinism.

---

## 3) Cosmetic types (v0)

### 3.1 Avatar skins
- A player selects an `AvatarSkinId` (PrimaryAssetId).
- Must support at least:
  - Masculine default
  - Feminine default
- Avatar skins may change mesh/materials/FX, but must not modify collision/traces in ways that affect board interaction.

### 3.2 Board skins
- A player selects a `BoardSkinId` (PrimaryAssetId).
- Any “arena presentation differences” must be handled purely by presentation (Blueprint/material/FX logic) and must not introduce new gameplay interaction surfaces.

### 3.3 Emotes (required by 0.7)
- Emotes are player-triggered expressions visible to others.
- Emotes are never gameplay-authoritative.

---

## 4) Ownership & authority

### Server-authoritative
- The final selected cosmetic IDs for a player during a match.
- Any emote activation decisions (validation + rate limiting).

### Client-only
- Cosmetic preview UI (selection menus, loadout screen).
- Local-only preview effects.

### UI boundary
- UI (BrawlUI) is read-only and may only issue **requests**:
  - RequestSetAvatarSkin(AvatarSkinId)
  - RequestSetBoardSkin(BoardSkinId)
  - RequestPlayEmote(EmoteId)
- The server validates requests and commits state.

---

## 5) Replication + replay

### 5.1 Selection replication policy
- Cosmetic IDs must replicate so:
  - other players can see them,
  - spectators can see them,
  - replays reproduce the chosen cosmetics.
- Because selection is **pre-match only**, cosmetic selection replication should be **InitialOnly** or otherwise “no mid-match changes”.

### 5.2 Emote replication policy
- Emotes are server-authorized and must replicate to viewers (board-relevant connections).
- Emotes should be implemented in a way that is captured by network replay (DemoNetDriver):
  - either replicated state with a monotonic counter,
  - or server multicast RPC routed via RepGraph relevance rules.

---

## 6) Non-interactivity & safety rules (hard rules)

Cosmetics must never interfere with board interaction or authoritative gameplay:

### 6.1 Trace/collision constraints
- Cosmetic meshes/components must not block:
  - tile interaction trace channel(s)
  - unit selection/hover trace channel(s)
- Cosmetics must not change unit/board collision in ways that influence gameplay-relevant traces.

### 6.2 Board layout invariants
- Cosmetics must not modify authoritative board layout:
  - TileSpacing/TileSize/BenchTileSpacing/BenchTileSize/BenchFieldGap
  - FieldWidth/FieldHeight/BenchWidth
- Cosmetics must not mutate occupancy state or placement rules.

### 6.3 Performance constraints
- Prefer instancing and static materials over per-frame tick logic.
- Avoid heavy runtime spawning during match unless budgeted and tested.

---

## 7) Data model & content directories

### 7.1 Asset identity
- Cosmetics use `FPrimaryAssetId` for stable references (replication/replay friendly).

### 7.2 Proposed PrimaryDataAsset types (C++ classes)
Recommended to live in **BrawlCore** (so BrawlGrid can remain “Core-only dependency”):
- `UBrawlAvatarSkinData : UPrimaryDataAsset`
- `UBrawlBoardSkinData : UPrimaryDataAsset`
- `UBrawlEmoteData : UPrimaryDataAsset`

### 7.3 Proposed content directories
- `/Game/Brawl/Data/Cosmetics/Avatars`
- `/Game/Brawl/Data/Cosmetics/Boards`
- `/Game/Brawl/Data/Cosmetics/Emotes`

AssetManager scanning must be configured so these assets resolve at runtime on server and clients.

---

## 8) GameplayTags (reserved namespaces)

If identity tags are used for cosmetics, reserve:
- `Cosmetic.AvatarSkin.*`
- `Cosmetic.BoardSkin.*`
- `Cosmetic.Emote.*`

Account progression tags are deferred to a separate contract.

---

## 9) Backend/persistence notes (deferred)

Planned direction:
- AWS GameLift + DynamoDB (or equivalent) for durable account state
- Authentication tied to Steam (EOS optional later)

Dev iteration (early):
- Cosmetics may be treated as “unlocked” and chosen via local save/config.
- Server still validates that requested PrimaryAssetIds exist and are allowable for the current build/season.

---

## 10) Cross-references
- Module boundaries: [brawl-implementation.md](cci:7://file:///C:/Users/hi/.codeium/.windsurf/rules/brawl-implementation.md:0:0-0:0) + [brawl-directories.md](cci:7://file:///C:/Users/hi/.codeium/.windsurf/rules/brawl-directories.md:0:0-0:0)
- Replication rules: [brawl-replication-contract.md](cci:7://file:///C:/Users/hi/.codeium/.windsurf/rules/brawl-replication-contract.md:0:0-0:0)
- Tag naming: [brawl-gameplay-tags-contract.md](cci:7://file:///C:/Users/hi/.codeium/.windsurf/rules/brawl-gameplay-tags-contract.md:0:0-0:0)
- Asset model: [brawl-data-model-contract.md](cci:7://file:///C:/Users/hi/.codeium/.windsurf/rules/brawl-data-model-contract.md:0:0-0:0)
- Avatar/camera behavior: `brawl-player-avatar-camera-contract.md`