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

### PlayerState (replicated, owner + potentially spectators depending on design)
Replicate (minimum):
- Gold, XP, Level, Win/Loss HP (if you have it), streak state
- Shop offers visible to that player (and to spectators if spectating includes economy visibility)
- Trait counts: Active + Potential
- Roster references (unit instance IDs owned by player)

Never client-authored:
- gold changes, purchases, rolls, pool counts

---

## 5) Unit replication
### ABrawlUnitCharacter (replicated actor)
Replicate:
- Stable UnitInstanceId (server-assigned)
- Owning PlayerId / TeamId
- Star level (1–3)
- Equipped ability IDs (1 basic + 1 ultimate) as replicated state (for UI + replay)
- Alive/dead state and death time marker (if needed)
- Transform state 
- `UnitDataId` (`FPrimaryAssetId`) so clients/replays/tools can resolve unit content identity (portrait/tags/stat curves/etc).

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

---

## 6) Board / grid replication
Board/Bench state must support:
- planning-phase placement visualization
- spectator viewing
- replay playback

Replicate either:
- (A) per-unit replicated “CurrentGridCoord + IsOnBench”, OR
- (B) a FastArray of grid slots -> UnitInstanceId mapping

### Placement Authority: Board Occupancy is Canonical (Option A)

- The authoritative placement state is `ABrawlBoardActor::Occupancy` (replicated slot/coord → `FBrawlUnitInstanceId`).
- Unit actor transforms are **derived** from occupancy:
  - On the server, when occupancy changes (move/swap/spawn), the unit actor is snapped/teleported to the coord’s authoritative world transform.
- Clients never author placement transforms. Clients only send placement requests; the server validates and updates occupancy.
- During `Phase.Combat`, field units may move freely via CharacterMovement/navmesh; occupancy remains the authoritative state for:
  - bench positions (bench-only movement rules),
  - combat start positions / planning placement history,
  - UI/spectate/replay decoding of “who was on what tile”.

Hard rule:
- Only the server changes placement state. Client sends placement request; server validates; server updates replicated state.

Additional requirements:
- Board occupancy replication must support the arena transfer model:
  - 8 planning boards (owner-only units on host half)
  - up to 4 combat arena boards (host + guest halves populated after transfer)
- Layout must be data-driven (no hardcoded dimensions in C++). Clients must be able to render placement correctly using replicated occupancy + layout config.
- If layout parameters are configurable per board instance/match (e.g., `FieldWidth`, `FieldHalfHeight`, `BenchWidth`, `TileSpacing`, `TileSize`, `BenchTileSpacing`, `BenchTileSize`, `BenchFieldGap`, and any gameplay-relevant `Coord -> World` spacing/gap), replicate them `InitialOnly` so clients/replays can decode occupancy deterministically.

### Arena terminology (clarification)
An “arena board” is not a special actor type. It refers to an existing /BrawlGrid/Public/Actors/BrawlBoardActor.h that the Match designates as the host board for a combat pairing for the current round.
Opponent units (and their bench) are transferred onto the host board’s guest half. The guest player’s home board remains spawned but is treated as inactive/empty for that round.

### Sandbox note (pre-ReplicationGraph / scout camera)
Until ReplicationGraph-based scouting relevancy (and a scout camera) are implemented, board replication/relevancy is mostly distance-based.
For sandbox testing, keep spawned boards physically close together so all boards remain network-relevant and visible without special camera movement.

### Implementation note: Occupancy (`ABrawlBoardActor::Occupancy`) is canonical; **unit actor transforms are derived** from occupancy changes (server snaps/teleports on commit)

### Input tracing policy (Tiles vs Units)
- **Tile coordinate picking**:
  - The board uses UBrawlBoardInteractionTilesComponent::GetInteractionTraceChannel() (currently `ECC_GameTraceChannel1`) for tile hit-testing.
  - Only the tile interaction ISM components should **block** this channel.
  - Unit actors (capsule/mesh) must **ignore** this channel so units do not “steal” tile hits.
- **Unit selection / hover / pickup**:
  - Use a **separate dedicated trace channel** (recommend: project-defined “UnitInteract”, e.g. `ECC_GameTraceChannel2`).
  - Units should **block** UnitInteract; tiles should **ignore** UnitInteract.
  - Controller policy: do a **unit trace first**, and if a unit is hit, resolve selection via **UnitId -> ABrawlBoardActor::TryFindUnitCoord(UnitId)** (do not infer coords from actor transforms). If no unit is hit, fall back to tile trace for `Coord`.
- **Authority**:
  - Client hit-testing is input-only. The server remains authoritative: it validates requested moves and commits occupancy updates; transforms are derived from occupancy on the server.

---

## 7) Economy / shop replication
- Shop offers are replicated as explicit offer entries (unit type ids + price + any flags).
- Any offer generation randomness must be server-only (seeded and logged).
- Clients never “roll locally”.

---

## 8) RPC contract (requests)
All request RPCs must be:
- Reliable only if needed; otherwise use Unreliable + server correction
- Rate-limited (basic abuse mitigation)

Required server validations:
- Phase legality (e.g., cannot purchase during combat unless round type allows it)
- Resource sufficiency (gold, bench space, etc.)
- Placement legality (grid bounds, swap rules, combat lock rules)
- Ability equip legality (must be one of the unit’s defined options)

---

## 9) “Do not replicate” list (default)
- Derived stats that can be recomputed locally for UI (unless needed for spectators)
- Internal AI decision state (except via Event Log/debug toggles)
- Per-tick transient values (e.g., “current desire score”)
- Any client-side predicted placement before server confirmation (optional cosmetic only)

---