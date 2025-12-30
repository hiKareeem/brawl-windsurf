---
trigger: always_on
description: 
globs: 
---

## 7) Board / grid replication
Board/Bench state must support:
- planning-phase placement visualization
- spectator viewing
- replay playback

Replicate:
- a FastArray of grid slots -> UnitInstanceId mapping

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

## 8) Economy / shop replication
- Shop offers are replicated as explicit offer entries (unit type ids + price + any flags).
- Any offer generation randomness must be server-only (seeded and logged).
- Clients never “roll locally”.

---

## 9) RPC contract (requests)
All request RPCs must be:
- Reliable only if needed; otherwise use Unreliable + server correction
- Rate-limited (basic abuse mitigation)

Required server validations:
- Phase legality (e.g., cannot purchase during combat unless round type allows it)
- Resource sufficiency (gold, bench space, etc.)
- Placement legality (grid bounds, swap rules, combat lock rules)
- Ability equip legality (must be one of the unit’s defined options)

---

## 10) “Do not replicate” list (default)
- Derived stats that can be recomputed locally for UI (unless needed for spectators)
- Internal AI decision state (except via Event Log/debug toggles)
- Per-tick transient values (e.g., “current desire score”)
- Any client-side predicted placement before server confirmation (optional cosmetic only)

---

## 11) Cosmetics replication (match-visible)

- Cosmetic selections (AvatarSkinId, BoardSkinId) are server-authored and must replicate to relevant viewers for spectate and replay capture.
- Selection is pre-match only; prefer InitialOnly replication.
- Emotes are client-requested but server-authorized (validate + rate limit) and must replicate (or multicast) in a replay-captured way.
- Contract: `brawl-cosmetics-contract.md`

---

## Visibility policy (v3)

### Scoreboard visibility (all players + spectators)
Replicate to all connections (COND_None):
- Player name (PlayerState base)
- Life, win/loss streak
- Gold
- XP / Level
- Roster summary (including bench vs field placement summary)

Rationale:
- Scoreboard must remain correct even when a player’s units/board actors are not relevant to a given connection.

### Spectator-only visibility
Spectators additionally receive:
- All boards’ occupancy (via ReplicationGraph routing)
- All players’ shop offers (server-gated)

### Shop offer replication gating (server-side)
- Shop offers are replicated via [UBrawlShopComponent](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlEconomy/Private/Components/BrawlShopComponent.cpp:28:0-33:1) as a subobject.
- The shop component replication condition is `COND_NetGroup`.
- The component is registered into net condition groups:
  - `UE::Net::NetGroupOwner` (owning player)
  - `Brawl.Spectator` (spectators)
- A spectator `APlayerController` must be included in net condition group `Brawl.Spectator` for shop replication to occur.

### ReplicationGraph board routing
- Spectators gather all board nodes (see all boards/occupancy).
- Non-spectators gather:
  - their active board
  - plus an additional scouted board (if any), as exposed via `IBrawlBoardPresenceInterface`.