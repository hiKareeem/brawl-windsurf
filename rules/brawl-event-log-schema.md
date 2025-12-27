---
trigger: always_on
description: 
globs: 
---

# Event Log Schema (Structured, Server Auth)

Event Log exists alongside network replay to support debugging/testing/analysis.
It is NOT required for replay playback, but must be stable and comprehensive.

[Later / when GameLift is added] you have 2 common options:
- Option A (recommended early): GameLift log upload
  - You write your event log file (and optionally replay .demo) into a directory that GameLift is configured to collect as “server logs”.
  - GameLift can then upload those logs to S3 as part of its fleet log collection flow.
  - Upside: no per-event networking, minimal code, great for “save locally first, cloud later”.
- Option B: Explicit upload to AWS
  - On match end, the server uploads artifacts to S3 / CloudWatch / Kinesis/Firehose.
  - Upside: more control + near-real-time analytics possible.
  - Downside: AWS SDK integration, IAM roles/credentials, retry/backoff, failure modes, and more code surface area.

---

## 1) Global requirements
- Server-only emission; clients may view but never author.
- Every event includes:
  - `SchemaVersion` (int)
  - `MatchId` (string/guid)
  - `ServerTimeSeconds` (float, match time since start)
  - `RoundIndex` (int)
  - `PhaseTag` (GameplayTag)
  - `EventType` (name/tag)
  - `InstigatorUnitId` (optional)
  - `TargetUnitId` (optional)
  - `PlayerId` (optional, for economy/placement)

- Use stable IDs:
  - UnitInstanceId is assigned by server and never reused within a match.

- Timebase (authoritative):
  - `ServerTimeSeconds` is the canonical timebase: **match time since start** (server authority).
  - Any additional timestamp fields inside specific event payloads (e.g. `CastStartTimeSeconds`, `SpawnTimeSeconds`, `ImpactTimeSeconds`, `TimeSeconds`) MUST use the **same timebase** as `ServerTimeSeconds`.
  - For combat events, do not mix `UWorld::GetTimeSeconds()` with match time in event payloads; use the match context time (`IBrawlMatchContextInterface::GetServerTimeSeconds()`).

- ID uniqueness:
  - `ProjectileId` only needs to be unique **within a match**. Treat `(MatchId, ProjectileId)` as the globally-unique key in logs/tools.

---

## 2) Required event types

### Economy
- `Economy.Purchase` (offerId/unitTypeId/cost)
- `Economy.Sell` (unitId/refund)
- `Economy.Reroll` (cost, resulting offer list ids)
- `Economy.BuyXP` (cost, xpGained, newLevel)

### Placement
- `Grid.UnitMoved` (unitId, fromCoord, toCoord, fromBenchFlag, toBenchFlag)
- `Grid.UnitSwapped` (unitAId, unitBId, coordA, coordB)
- `Grid.OverflowSpawned` (unitId, coord)
- `Grid.OverflowResolved` (unitId, destroyedFlag, refund)

### Combat decisions (navmesh drift tolerant)
- `Combat.TargetChosen`
  - abilityId, targetId, policy (Retarget/Fizzle/LastPosition)
  - optional target position snapshot (vector) when using LastPosition
- `Combat.AbilityCast`
  - casterId, abilityId, castStartTime, lockedTargetId (optional)
- `Combat.ProjectileSpawned`
  - `targetId` / `targetPos` represent the **intended** target at spawn time.
  - If using LastPosition, `targetPos` is the **position snapshot taken at target resolution time**.
- `Combat.ProjectileImpacted`
  - `impactedTargetId` is the **actual** impacted unit (if any). It may differ from the intended target, or be unset/invalid on miss/expiry.
- `Combat.DamageApplied`
  - sourceId, targetId, abilityId, amount, damageClassTag, elementTag, finalTagsApplied
  - damageClassTag is a `DamageClass.*` tag (Physical/Special/Mixed/True)
  - elementTag is the ability’s `Element.*` tag (used for effectiveness/STAB when applicable)

### Combat.UnitDied (field additions)
- `KillingDamageAppliedSequenceNumber` (int64)
  - Optional. If known, this is the EventLog `SequenceNumber` of the `Combat.DamageApplied` event that caused this death.
  - If unknown/unavailable, set to `0`.

#### Clarification: PlayerId semantics for combat events
For `Combat.UnitDied`, define whether `PlayerId` refers to:
- the dead unit’s owning player (victim)

### Combat.ArenaResolved
- `HostFinalEliminationDamageSequenceNumber` (int64)
- `GuestFinalEliminationDamageSequenceNumber` (int64)
  - Sequence numbers of the killing `Combat.DamageApplied` events for each side’s final elimination.
  - `0` if the ordering could not be determined.
- `HostPlayerId` (int)
- `GuestPlayerId` (int)
- `WinningPlayerId` (int)
- `LosingPlayerId` (int)
- `Outcome` (string): `HostWin` / `GuestWin`
- `bWentToOvertime` (bool)
- `Tiebreaker` (string): `Elimination` / `UnitCount` / `TotalHealth` / `DeterministicPlayerId`
- `HostAliveUnitCount` (int)
- `GuestAliveUnitCount` (int)
- `HostAliveTotalHealth` (float)
- `GuestAliveTotalHealth` (float)

Emission guarantees:
- Server-only.
- Emitted at most once per arena board per combat phase.
- Used by Rewards application; Rewards must not be applied until all relevant arenas have emitted `Combat.ArenaResolved`.

Field encoding notes (v1):
- `AbilityId` (all combat events):
  - Prefer [UBrawlAbilityData::GetPrimaryAssetId().PrimaryAssetName](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlAbilities/Public/Data/BrawlAbilityData.h:19:1-19:60) (asset name) when AbilityData is available.
  - Fallback (only when no AbilityData exists): the `UGameplayAbility` class name (`Ability->GetClass()->GetFName()`).
- `Combat.TargetChosen.Policy`:
  - Encodes the invalidation policy token: `Retarget` / `Fizzle` / `LastPosition` (not the TargetingPolicy asset name).

---

## 3) Emission points (guarantees)
- Emit `TargetChosen` when a cast locks a target OR when the targeting policy resolves.
- Emit `ProjectileSpawned` and `ProjectileImpacted` for all non-instant projectiles.
- Emit `DamageApplied` for every damage application (including DoTs, on-hit procs).
- Emit placement and economy events immediately after server commits the state change.

- Projectile lifecycle pairing (guarantee):
  - For every emitted `Combat.ProjectileSpawned`, the server MUST emit exactly one terminal `Combat.ProjectileImpacted` for the same `projectileId`.
  - `Combat.ProjectileImpacted` must be emitted even if the projectile:
    - hits nothing (miss),
    - expires (`MaxLifetimeSeconds` / lifespan),
    - is destroyed for cleanup/round transitions.
  - A miss is represented by leaving `impactedTargetId` unset/invalid, while still emitting `impactPos` (best-known final position).

- Projectile impact ordering:
  - If an impact applies damage/effects, emit `Combat.ProjectileImpacted` **before** `Combat.DamageApplied` for that impact.

---

## 4) Ordering and stability
- When multiple events occur “at the same time”, order them deterministically:
  - sort by ServerTimeSeconds, then by stable UnitInstanceId, then by EventType priority.
- Trigger ordering invariant to reflect in logs:
  1) OnHit
  2) OnDamaged
  3) OnDeath

---

## 5) Runtime buffering contract (in-memory ring buffer)

The server maintains an in-memory ring buffer of recently published events via `UBrawlMatchEventLogSubsystem` (BrawlMatch).
This exists to support automation tests and debug tooling. It is not required for replay playback.

### 5.1 Server-only behavior
- The subsystem is server-only: it is inactive on clients and clients never author buffered events.

### 5.2 Buffered entry representation
Each buffered entry stores:
- `SequenceNumber` (monotonic identifier)
- `EventStructName` (the published event `UScriptStruct` name)
- `EventBase` (the base `FBrawlEventBase` fields for quick filtering)
- `EventStruct` (the event payload `UScriptStruct*`)
- `EventBytes` (a deep-copied, fully-constructed instance of the published event struct)

Important: `EventBytes` is NOT a serialized payload. It is raw storage holding a constructed `UScriptStruct` instance.
- It must be created using `UScriptStruct::InitializeStruct` and copied using `UScriptStruct::CopyScriptStruct`.
- It must be destroyed using `UScriptStruct::DestroyStruct`.
- Do not treat it as `memcpy`/POD bytes.

### 5.3 Ordering and retrieval
- The runtime ring buffer preserves publish order.
- [GetBufferedEvents(...)](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Subsystems/BrawlMatchEventLogSubsystem.h:36:4-36:83) returns entries in order: oldest → newest (publish order).
- Retrieval produces independent copies of entries (callers may safely retain/inspect them without depending on the subsystem’s lifetime).

If/when events are persisted/exported to a file, the persistence layer is responsible for applying any additional deterministic ordering rules described in “## 4) Ordering and stability”.

### 5.4 Type-safe decoding for tests/tools
Recommended pattern:
1) Check `Entry.EventStruct` (or `Entry.EventStructName`) matches the expected type.
2) Interpret/copy the payload using the struct reflection API.

Important:
- Do NOT `reinterpret_cast` `Entry.EventBytes` into an event struct type in tests/tools.
- Always decode by copying via `UScriptStruct::InitializeStruct` + `CopyScriptStruct` (example below). This avoids alignment and non-POD lifetime issues (see also “5.5 Alignment note”).

Example (copy out):
```cpp
const FBrawlMatchBufferedEventEntry& Entry = ...;

if (Entry.EventStruct == FBrawlCombatProjectileSpawnedEvent::StaticStruct())
{
    FBrawlCombatProjectileSpawnedEvent Payload;
    FBrawlCombatProjectileSpawnedEvent::StaticStruct()->InitializeStruct(&Payload);
    FBrawlCombatProjectileSpawnedEvent::StaticStruct()->CopyScriptStruct(&Payload, Entry.EventBytes.GetData());

    // read Payload fields...

    FBrawlCombatProjectileSpawnedEvent::StaticStruct()->DestroyStruct(&Payload);
}```

### 5.5 Alignment note (current implementation)
Event payloads are stored in TArray<uint8>. This relies on engine allocator alignment being sufficient for all event types. If an event type requires an alignment greater than the allocator guarantee (see EventStruct->GetMinAlignment()), the buffer storage must be updated to use aligned allocation.

### 5.6 Sequence number semantics
 - SequenceNumber is monotonic within a world lifetime.
 - Clearing the buffer does not reset the monotonic sequence generator (i.e., new events continue with increasing SequenceNumber).

## 6) Export artifact (v1) — JSONL on match end

### Trigger + output path
- Export is server-only and occurs on match end ([ABrawlGameState::EndMatch()](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Game/BrawlGameState.cpp:36:0-51:1) -> `ABrawlGameMode` hook).
- Output path:
  - `Saved/<MatchEventLogExportSubdir>/MatchEventLog_<MatchId>_<UTC timestamp>.jsonl`
  - `MatchEventLogExportSubdir` comes from `UBrawlNetSettings` (`DefaultEngine.ini`), default `BrawlEventLogs`.

### JSONL line format
- One JSON object per published event, written in publish order (oldest → newest).
- Each line contains all reflected USTRUCT fields of the specific event payload, plus:
  - `SequenceNumber` (string; monotonic within the world lifetime)
  - `EventStructName` (string; published `UScriptStruct` name)