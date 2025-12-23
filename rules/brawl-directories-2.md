---
trigger: always_on
description: 
globs: 
---

## 7) Module: BrawlMatch
Purpose: phase/round state machine, creep rounds, orchestration, match event log buffering (persistence/export deferred).

### Files / Classes
**Game framework**
- `Public/Game/BrawlGameMode.h`  -> `ABrawlGameMode : AGameModeBase` (server rules entry)
- `Public/Game/BrawlGameState.h` -> `ABrawlGameState : AGameStateBase`
  - Replicates: round index, phase tag, time remaining
- `Public/Game/BrawlPlayerState.h` -> `ABrawlPlayerState : APlayerState`
  - Owns Economy/Shop/Traits/Progression components, roster refs
- [Public/Game/BrawlPlayerController.h](cci:7://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Game/BrawlPlayerController.h:0:0-0:0) -> `ABrawlPlayerController : APlayerController`
  - Sends server requests (placement, shop, equip)
  - Placement RPC: [ServerRequestMoveUnit(FBrawlUnitInstanceId UnitId, FBrawlGridCoord ToCoord)](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Game/BrawlPlayerController.h:18:4-18:85)
    - Server derives `PhaseTag` from [ABrawlGameState](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Game/BrawlGameState.cpp:5:0-8:1) (never trust client)
    - Server derives allowed bench row from Board host/guest player ids

**Round manager**
- `Public/Components/BrawlRoundManagerComponent.h`
  - Planning/Combat/Rewards/Shop round types
  - Reads round definitions from `UBrawlRoundSetData`

**Combat driver (centralized combat updates)**
- [Public/Components/BrawlCombatManagerComponent.h](cci:7://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Components/BrawlCombatManagerComponent.h:0:0-0:0) -> `UBrawlCombatManagerComponent : UActorComponent`
  - Owned by [ABrawlGameState](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Game/BrawlGameState.cpp:14:0-19:1)
  - Server-only tick; enabled only while `PhaseTag == Phase.Combat` (via [ABrawlGameState::SetPhaseTag](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Game/BrawlGameState.h:33:4-33:46))
  - Enumerates combat units via [ABrawlBoardActor::GetRegisteredUnitIds(...)](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlGrid/Public/Actors/BrawlBoardActor.h:93:1-93:75) and processes them deterministically (sort by `FBrawlUnitInstanceId.Value`)

**Round data**
- `Public/Data/BrawlRoundData.h`
- `Public/Data/BrawlRoundSetData.h`

**Creeps**
- `Public/Components/BrawlCreepSpawnerComponent.h` (or Actor if preferred)

**Event log**
- [Public/Subsystems/BrawlMatchEventLogSubsystem.h](cci:7://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Subsystems/BrawlMatchEventLogSubsystem.h:0:0-0:0)
  - Subscribes to EventBus; buffers recent events in-memory (base `FBrawlEventBase` fields + event `UScriptStruct` + deep-copied payload bytes) for debug/testing. Persistence/export is deferred.
---
## 8) Module: BrawlNet
Purpose: replication scaling + replay recording hooks.

### Files / Classes
- `Public/Net/BrawlReplicationGraph.h`
  - `UBrawlReplicationGraph : UReplicationGraph`
- `Public/Subsystems/BrawlReplaySubsystem.h`
  - `UBrawlReplaySubsystem : UGameInstanceSubsystem`
  - Start/stop DemoNetDriver recording; attach metadata

**Status note (2025-12):** `BrawlNet` is planned but not yet present in [Source/](cci:7://file:///E:/Unreal/BrawlFinal/Brawl/Source:0:0-0:0) in this repo. Do not reference it from other modules until it is created (Milestone 6+).
---
## 9) Module: BrawlUI
Purpose: MVVM viewmodels + widget binding. No gameplay rules.

### Files / Classes (ViewModels)
- `Public/UI/BrawlVM_Shop.h`
- `Public/UI/BrawlVM_Board.h`
- `Public/UI/BrawlVM_UnitPanel.h` (ability toggles)
- `Public/UI/BrawlVM_Traits.h` (Active(Potential)/Threshold)
- `Public/UI/BrawlVM_RoundTimer.h`
- `Public/UI/BrawlVM_DamageRecap.h`
- `Public/UI/BrawlVM_DPSMeter.h`

Widgets live in Content; they bind to these VMs.
---
## 10) MVP creation order (do not skip)
1) **BrawlCore:** tags + IDs + logging + EventBus + event structs
2) **BrawlMatch:** GameMode/GameState/PlayerState/PlayerController + RoundManager scaffolding
3) **BrawlGrid:** BoardActor + Grid/Bench components + placement rules
4) **BrawlUnit:** UnitCharacter + Definition + Loadout (equipped IDs replicate)
5) **BrawlAbilities:** ASC + AttributeSets + ExecCalc_Damage + Cooldown MMC + GA bases + ProjectileActor
6) **BrawlEconomy:** Economy + Shop + SharedPool + Traits (Active vs Potential)
7) **BrawlNet:** ReplicationGraph + ReplaySubsystem (record server replays)
8) **BrawlAI:** AIController + targeting utilities + debug
9) **BrawlUI:** viewmodels + widgets (MVVM)
---
## 11) Ownership quick map
- **RoundManager (Match)** owns: Phase tag + timers (replicated via GameState)
- **Economy/Shop (PlayerState components)** own: gold/xp/offers/traits (replicated)
- **Board/Grid (BoardActor/components)** own: placement occupancy state (replicated)
- **Unit (UnitCharacter)** owns: ASC + replicated identity + loadout state
- **Abilities (GAS)** own: combat effects and tag-based state
- **Net (ReplicationGraph)** owns: relevancy/bandwidth rules
- **EventBus/EventLog** observe and record; they do not author gameplay state