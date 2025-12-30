---
trigger: model_decision
description: When working on player character or camera
globs: 
---

# Player Avatar + Camera Contract (GASP + GameplayCameras) — UE 5.7

This document defines the **player avatar**, **camera mode switching**, and **scouting/teleport** behavior.
It exists to prevent gameplay logic from leaking into UI and to keep camera work compatible with **ReplicationGraph** and **server authority**.

---

## 1) Goals / non-goals

### Goals
- Use a **GASP-derived Character** as the **player avatar** for all players in normal matches.
- The avatar is **replicated** (visible to others) but must remain **gameplay-non-authoritative**.
- Support two camera modes:
  - **Board Camera** (top-down / tactical; used for planning/combat viewing and scouting)
  - **Over-the-Shoulder (OTS) Follow Camera** (GASP-style)
- **TFT-style behavior**:
  - When a player’s `ActiveBoardActor` changes (planning ↔ combat arena), the client is forced into **Board Camera**, and the player avatar is **teleported** to that board.
  - When a player scouts another player, it **forces Board Camera** and teleports the avatar to the scouted board.

### Non-goals
- Camera mode selection is **not a replicated gameplay decision**.
- Avatar position does **not** affect combat, placement legality, targeting, or any server outcomes.

---

## 2) Ownership & authority

### Server-authoritative state
- `ABrawlPlayerState::BoardActor` is the stable home board.
- `ABrawlPlayerState::ActiveBoardActor` is the player’s current board (home or arena) and is **server-authored**.
- Scouting target (e.g., `ScoutedPlayerId`, `ScoutedBoardActor`) is **server-authored** from a client request and **validated** server-side.

### Client-only state
- Camera mode state (Board vs OTS), camera blends, and camera rig tuning are **client-only** and must never be relied upon for gameplay.

### UI boundary
- UI (BrawlUI) is read-only and may only issue **requests** (e.g., “RequestScoutPlayer(TargetPlayerId)”).
- The server decides if the request is valid and applies the resulting state.

---

## 3) Avatar specification (normal matches)

### Required
- Player-controlled pawn is a **GASP-derived Character Blueprint** (derived from `CBP_SandboxCharacter`).
- The avatar is **replicated** (CharacterMovement replication as appropriate).
- Movement (walking/jumping) is allowed in all camera modes (board and OTS).
  - Board camera mode: WASD/Space, left click to move to position/follow cursor (TFT-like), right click interact.
  - OTS camera mode: WASD/Space, left click to orient camera, right click can hover units but not interact/place them. 
- The avatar must be configured so it cannot interfere with board interaction:
  - It must not block tile trace channel(s).
  - It must not block unit selection trace channel(s).
  - It must not collide with units in a way that changes gameplay outcomes.

### Notes
- We will also use a GASP-derived approach for `ABrawlUnitCharacter` in the future, but that is separate from this player-avatar contract.

---

## 4) Camera modes (GameplayCameras integration)

We use the **GameplayCameras** plugin (GASP’s camera stack) to implement these modes:

### 4.1 Board Camera mode
- Uses the board’s view target transform for the relevant player:
  - [ABrawlBoardActor::TryGetViewTargetTransformForPlayer(TargetPlayerId, OutTransform)](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlGrid/Private/Actors/BrawlBoardActor.cpp:681:0-705:1)
- Must **ignore controller control rotation** (board camera rotation is board-authored/tuned, not driven by mouse look).

### 4.2 OTS Follow Camera mode
- Uses GASP’s default third-person follow camera behavior.
- Intended for “walk around and inspect” (cosmetic).

### 4.3 Switching policy (hard rules)
- On any local **ActiveBoardActor change**:
  - Force **Board Camera mode**
  - Set board view to **self** (TargetPlayerId = self)
- On a **scout request** (scoreboard click):
  - Force **Board Camera mode**
  - Set board view to the scouted player + board
- Manual switching to OTS is allowed at any time, but:
  - the next ActiveBoardActor change or scout request re-forces Board Camera.

#### Spectator note (no pawn)
- If the local controller has no pawn (spectator), the controller may use a transient `ACameraActor` as the view target when applying board camera views.

---

## 5) Teleport rules (TFT-style “move the little legend”)

### 5.1 Active board teleport
When the server changes a player’s `ActiveBoardActor`:
- The server must teleport that player’s avatar to the new board.
- The client must also switch into Board Camera mode.

### 5.2 Scouting teleport
When a client requests to scout `TargetPlayerId`:
- The server validates the request, resolves the target’s current `ActiveBoardActor`,
- Sets the viewer’s scouted selection state,
- Teleports the viewer’s avatar to the resolved board,
- The client switches into Board Camera mode.
### Scouted board state updates are idempotent

[ABrawlPlayerState::SetScoutedBoardActor(PlayerId, BoardActor)](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Game/BrawlPlayerState.cpp:312:0-335:1) must be idempotent:
- If `(ScoutedPlayerId, ScoutedBoardActor)` already equals `(PlayerId, BoardActor)`, the call is a no-op.

Rationale:
- Prevents redundant delegate broadcasts (`OnScoutedBoardActorChanged`) and avoids spamming server-authored teleports/camera triggers that are driven by those state changes.

### Scouting invalidation (server-authored)
- If the scouted target becomes invalid (becomes spectator, `Life <= 0`, or leaves the match), the server clears the viewer’s scouted state:
  - `ScoutedPlayerId = INDEX_NONE`
  - `ScoutedBoardActor = nullptr`
- When scouted state clears, the client returns to auto-following their own `ActiveBoardActor` (Board Camera mode).

### Implementation notes: server-authored teleports on board-selection state
- Board/presence teleports are **server-authored** and triggered by [ABrawlPlayerState](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Game/BrawlPlayerState.cpp:66:0-77:1) state changes:
  - [SetActiveBoardActor(...)](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Game/BrawlPlayerState.h:103:4-103:61) teleports the owning player avatar to the new board presence anchor.
  - [SetScoutedBoardActor(TargetPlayerId, BoardActor)](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Game/BrawlPlayerState.h:113:4-113:80) teleports the owning player avatar to the scouted board presence anchor for `TargetPlayerId`.
- Scouting selection state (`ScoutedPlayerId`, `ScoutedBoardActor`) is replicated **owner-only** (viewer-only), since it is presentation state for that viewer’s camera/pawn.
- When a player becomes a spectator (including elimination), that player’s own scouting selection is cleared server-side.

### 5.3 Teleport destination
- The board provides a camera view transform already (host/guest offsets).
- The avatar teleport destination should be a **board-defined “presence anchor”** near the relevant side of the board.
- The anchor must be cosmetic and must not impact gameplay. Exact anchor authoring is content/iteration-driven.

#### Current default (C++ implementation)
- `ABrawlBoardActor::TryGetPresenceAnchorTransformForPlayer(PlayerId, ...)`:
  - Uses a field coord anchor (not bench): `X = FieldWidth - 1`
  - `Y = 0` for host, `Y = FieldHeightTotal - 1` for guest
  - Facing: host yaw 180, guest yaw 0
- This remains cosmetic-only and may be replaced by data-driven anchors later.

---

## 6) ReplicationGraph considerations

- Avatars are replicated actors and must be routed so that:
  - a connection viewing a board receives the avatars currently on that board.
  - spectators who “see all boards” also see all avatars.
- Camera state itself is not replicated.

---

## 7) Validation & abuse mitigation (server)

- Scouting is a client request; the server must validate:
  - target player exists
  - request rate limit
  - any spectator-only rules (if/when added)
- Teleport is server-authored. Clients never author their own teleport destination.

---

## 8) Documentation cross-references
- Replication rules: [brawl-replication-contract.md](cci:7://file:///C:/Users/hi/.codeium/.windsurf/rules/brawl-replication-contract.md:0:0-0:0)
- Module ownership: [brawl-directories.md](cci:7://file:///C:/Users/hi/.codeium/.windsurf/rules/brawl-directories.md:0:0-0:0) + [brawl-directories-2.md](cci:7://file:///C:/Users/hi/.codeium/.windsurf/rules/brawl-directories-2.md:0:0-0:0)
- Current work plan: [brawl-plan.md](cci:7://file:///C:/Users/hi/.codeium/.windsurf/rules/brawl-plan.md:0:0-0:0)