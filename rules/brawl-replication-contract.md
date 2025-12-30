---
trigger: always_on
description: 
globs: 
---

# Replication Contract (Dedicated Server, UE 5.7)

This document specifies what replicates, who owns it, and what must never be client-authored.
Goal: competitive integrity + spectate + replay recording (DemoNetDriver) + scalable to 8 players.

---

## 1) Authority model
- Dedicated server is authoritative for all gameplay state.
- Clients may only send **requests** (RPCs). Server validates and executes. Clients never compute outcomes.

---

## 2) Replay requirement
- Server records network replays (DemoNetDriver).
- Replays must include enough replicated state/events to reconstruct:
  - purchases/sells/rerolls/XP buys
  - unit placement changes (bench/field, swaps)
  - equipped ability selections
  - combat outcomes (ability usage, damage, deaths)
- Combat does NOT need deterministic re-simulation on playback.

---

## 3) ReplicationGraph (required)
- Use ReplicationGraph for relevancy/bandwidth control.
- Relevancy rules (baseline):
  - A player is relevant to: their board + opponent board (+ shared UI state like round/phase).
  - Spectator may opt into “see all boards”.
  - Current implementation uses APlayerState::IsSpectator() in RepGraph
- Do not rely on “replicate everything to everyone”.

---

## 4) Replicated game framework objects
### GameState (replicated to all relevant clients)
Replicate:
- Current round index / round type identifier
- Current phase tag (`Phase.Planning`, `Phase.Combat`, etc.)
- Phase time remaining (server time reference)
- Match-level tuning hash/version (for debugging)

Server-only:
- pairing logic (later 8-player)
- event log file export triggers

### PlayerState (replicated to all players; spectators see all)
Replicate (v3/v4 visibility policy):
- To **all players** (and spectators):
  - Player name / identifier
  - Life
  - Gold
  - XP + Level
  - Win/Loss streak state
  - **Player-centric roster list** (owned unit instance ids + content ids needed for UI)
  - **Player-centric bench list / “away-safe” unit list**:
    - The scoreboard must remain correct even if the unit actors are currently away on a different arena board.
    - Bench/field membership for a given board is still derived from canonical `ABrawlBoardActor::Occupancy`, but the “what units does this player own” list is player-centric and not tied to actor relevancy.

- To **spectators** (in addition to the above):
  - All shop offers for all players
  - Visibility of all boards’ replicated occupancy (grid slot -> UnitInstanceId mapping)

Unit actor visibility policy (RepGraph):
- At minimum: a connection should receive unit actors for the viewer’s **currently selected/scouted board** (home/active/opponent being scouted).
- Optional (performance permitting): keep multiple/all concurrent battles relevant so the player can look around and see multiple fights.
- Do not rely on distance-only replication for this; enforce via ReplicationGraph routing.

Never client-authored:
- gold changes, purchases, rolls, pool counts

Implementation note:
- Use ReplicationGraph relevancy/routing to control bandwidth; do not rely on distance-only replication.

---

## 5) Player avatar + camera replication (GASP)

- Player avatar is a replicated Character (cosmetic / presentation-only).
- Camera state (board vs OTS mode, camera rigs, control rotation behavior) is **client-only** and must not be replicated or used for gameplay authority.
- Teleports to boards (ActiveBoard changes or scouting) are **server-authored**; clients send requests only.
- ReplicationGraph should route avatars to the same relevancy buckets as boards so viewers of a board receive the avatars on that board.
- Contract: `brawl-player-avatar-camera-contract.md`

### Player pawn/avatar RepGraph routing (implementation note)
- Player-controlled pawns/avatars are treated as **board-scoped actors** for relevancy.
- Routing is assigned in [UBrawlReplicationGraph::RefreshBoardScopedRouting()](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlNet/Private/Net/BrawlReplicationGraph.cpp:215:0-396:1):
  - Prefer the pawn’s owning `APlayerState` via `IBrawlBoardPresenceInterface`:
    - `ActiveBoardActor`, else `HomeBoardActor`.
  - Fallback: nearest `ABrawlBoardActor` by distance (deterministic tie-break by board name).
- Spectators (who gather all board nodes) will receive all pawns/avatars as part of those board nodes.

---

## 6) Unit replication
### ABrawlUnitCharacter (replicated actor)
Replicate:
- Stable UnitInstanceId (server-assigned)
- Owning PlayerId / TeamId
- Star level (1–3)
- Equipped ability IDs (1 basic + 1 ultimate) as replicated state (for UI + replay)
- Alive/dead state and death time marker (if needed)
- Transform state 
- `UnitDataId` (`FPrimaryAssetId`) so clients/replays/tools can resolve unit content identity (portrait/tags/stat curves/etc).
- Equipped item ID(s) (e.g., `FPrimaryAssetId EquippedItemId`) as replicated state (UI + replay friendly)

Movement:
- Units move via CharacterMovement; combat navmesh determinism is not required.
- Keep movement replication settings conservative; let RepGraph relevance handle bandwidth.

GAS:
- ASC replicates:
  - GameplayTags (states, CC, immunities)
  - Attributes necessary for UI/combat (as per GAS replication mode)
- AttributeSets:
  - For attributes that must replicate to clients (UI/spectate/debug), use standard GAS attribute replication (`ReplicatedUsing` + `OnRep_*` + `GAMEPLAYATTRIBUTE_REPNOTIFY`).
  - During early scaffolding/plumbing, AttributeSets may exist without attribute replication; do not assume UI updates from attributes yet.
- Prefer Minimal replication mode where feasible; avoid replicating transient internal variables.