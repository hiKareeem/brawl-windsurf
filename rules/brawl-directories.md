---
trigger: always_on
description: 
globs: 
---

# Directory + Class Map (Brawl, UE 5.7)
## 0) Source directory layout (per module)
Each module follows:
- `Source/<ModuleName>/Public/...`
- `Source/<ModuleName>/Private/...`

Recommended subfolders (use only what you need):
- `Actors/`
- `Components/`
- `Data/` (DataAssets, table row structs)
- `Interfaces/`
- `Subsystems/`
- `GAS/` (Abilities, Effects helpers, AttributeSets, ExecCalcs, MMCs)
- `AI/`
- `Net/`
- `UI/` (Viewmodels)
---
## 1) Module: BrawlCore
Purpose: shared types, tags, logging, stable IDs, event bus, interfaces. No game rules.

### Files / Classes
**Logging**
- `Public/BrawlLog.h`
  - Declares log categories:
    - `LogBrawlMatch`, `LogBrawlEconomy`, `LogBrawlCombat`, `LogBrawlAI`, `LogBrawlGrid`, `LogBrawlNet`, `LogBrawlUI`

**GameplayTags**
- `Public/BrawlGameplayTags.h`
- `Private/BrawlGameplayTags.cpp`

**Stable IDs**
- `Public/Types/BrawlIds.h`
  - `FBrawlMatchId` (Guid wrapper or FString)
  - `FBrawlUnitInstanceId` (uint32/uint64, server-issued, never reused within match)

**Event structs**
- `Public/Events/BrawlEvents.h`
  - `FBrawlEventBase` + required derived event structs (economy/grid/combat)

**Event bus**
- `Public/Subsystems/BrawlEventBusSubsystem.h`
- `Private/Subsystems/BrawlEventBusSubsystem.cpp`
  - `UBrawlEventBusSubsystem : UWorldSubsystem`
  - `PublishEvent(const FBrawlEventBase&)`

**Interfaces**
- `Public/Interfaces/BrawlCombatantInterface.h` (ASC + UnitInstanceId + Alive)
- `Public/Interfaces/BrawlTeamAgentInterface.h` (TeamId / OwnerPlayerId)
- `Public/Interfaces/BrawlGridOccupantInterface.h` (GridCoord accessors)
- `Public/Interfaces/BrawlMatchContextInterface.h` (MatchId/RoundIndex/PhaseTag/ServerTimeSeconds for event logging; implemented by `ABrawlGameState`)
---
## 2) Module: BrawlGrid
Purpose: board/bench representation + placement rules + tile mods.

### Files / Classes
**Grid types**
- `Public/Types/BrawlGridTypes.h`
  - `FBrawlGridCoord { int32 X, Y; bool bIsBench; }`

**Board actor**
- `Public/Actors/BrawlBoardActor.h`
- `Private/Actors/BrawlBoardActor.cpp`
  - `ABrawlBoardActor : AActor`
  - Owns the grid/bench components for one player board
  - Replicates occupancy mapping needed for UI/spectate/replay
    - A single `ABrawlBoardActor` may contain **both** players’ units (host half + guest half)
    - Both players may request bench shuffles for **their owned units** even if those units currently live on the opponent’s board actor (guest bench row on host board)
    - Field is locked in combat; bench-only movement is allowed (host bench row for host player, guest bench row for guest player)

**Grid + bench comps**
- `Public/Components/BrawlGridComponent.h`
- `Public/Components/BrawlBenchComponent.h`
  - Server-only mutation APIs:
    - `ServerMoveUnit(UnitId, ToCoord)`
    - swap-on-drop
  - Replication strategy: FastArray or per-unit coord replication (choose one; do not mix)

**Placement rules**
- `Public/Data/BrawlPlacementRuleset.h`
  - `UBrawlPlacementRuleset : UDataAsset`
  - Policy: what moves are legal per phase (Planning vs Combat)

**Special tiles (optional early, but reserved)**
- `Public/Data/BrawlTileModifierData.h`
- `Public/Components/BrawlTileModifierComponent.h`
  - Applies GameplayEffects to units occupying tagged tiles

**Board coordinate system**
- Canonical coordinate system is auth and uses `FBrawlGridCoord`.
- Layout (field width, field half-height, bench width) must be **data-driven** (DataAsset / BP config), with a default of 7x4 per side + 9x1 bench, but tunable for experiments (e.g., 7x4). Bench rows are fixed at 2 (host + guest).
- Guest half is 180° rotated relative to host for transfer/mirroring; UI/view mapping is separate and not authoritative.
- World-space layout (`FBrawlGridCoord -> WorldTransform`) is gameplay-relevant and must be owned by the auth gameplay board actor (or BrawlGrid component).
  - Layout supports independent bench vs field tile spacing/size (`BenchTileSpacing`/`BenchTileSize` overriding `TileSpacing`/`TileSize`).
  - `BenchFieldGap` is an edge-to-edge gap between bench tiles and field tiles (not a center-to-center spacing rule).
- Cosmetic meshes must be decorative only and must not affect collision/placement.
- Board lifecycle (TFT-like):
  - Planning: 8 boards active; each board hosts only its owner’s units on the host half; guest half forbidden.
  - Combat: up to 4 arena boards active; opponent units are transferred onto the arena host board’s guest half; both benches exist and are interactable for assigned players

Ownership: Board/Grid never decides economy/phase; it asks Match for current `Phase.*` tag if needed.
---
## 3) Module: BrawlUnit
Purpose: unit actor + ASC ownership + lifecycle + star combine + loadout state (not ability logic).

### Files / Classes
**Unit data**
- `Public/Data/BrawlUnitData.h`
  - `UBrawlUnitData : UPrimaryDataAsset`

**Unit character**
- `Public/Actors/BrawlUnitCharacter.h`
- `Private/Actors/BrawlUnitCharacter.cpp`
  - `ABrawlUnitCharacter : ACharacter, IAbilitySystemInterface`
  - Replicates:
    - `FBrawlUnitInstanceId UnitId`
    - `int32 StarLevel`
    - `FPrimaryAssetId EquippedBasicAbilityId`
    - `FPrimaryAssetId EquippedUltimateAbilityId`
    - Team/owner refs as needed
    - `FPrimaryAssetId EquippedItemId`

**Definition/init**
- `Public/Components/BrawlUnitDefinitionComponent.h`
  - Applies UnitData tags, initializes base state, applies star scaling hooks (actual attribute math is in Abilities module)

**Loadout**
- `Public/Components/BrawlUnitLoadoutComponent.h`
  - Server grants/removes GAS abilities based on equipped IDs
  - Enforces “1 basic + 1 ultimate equipped”

**Items**
- [Public/Data/BrawlItemData.h](cci:7://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlUnit/Public/Data/BrawlItemData.h:0:0-0:0)
  - `UBrawlItemData : UPrimaryDataAsset`
- [Public/Components/BrawlUnitItemComponent.h](cci:7://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlUnit/Public/Components/BrawlUnitItemComponent.h:0:0-0:0)
  - `UBrawlUnitItemComponent : UActorComponent`
  - Applies/removes item-authored GameplayEffects / ability grants / loose tags to the unit ASC (server authority)

**Star combining**
- `Public/Components/BrawlStarCombineComponent.h`
  - Combines 1–3 star; emits events to EventBus
  - Leaves stat scaling to Abilities (attributes) using curve refs from UnitData

**Lifecycle/state**
- `Public/Components/BrawlUnitStateComponent.h` (optional but recommended)
  - Alive/dead transitions, despawn hooks, combat lock flags

Ownership: Unit owns ASC, but ability content/framework lives in BrawlAbilities.
---
## 4) Module: BrawlAbilities
Purpose: GAS framework, AttributeSets, ExecCalcs, MMCs, base abilities, targeting policies, projectile gameplay.

### Files / Classes
**ASC**
- `Public/GAS/BrawlAbilitySystemComponent.h`
  - `UBrawlAbilitySystemComponent : UAbilitySystemComponent`

**AttributeSets**
- `Public/GAS/Attributes/BrawlAttrSet_Combat.h` (HP/Atk/SpAtk/Def/SpDef)
- `Public/GAS/Attributes/BrawlAttrSet_Speed.h` (Spd + derived knobs if used)
- `Public/GAS/Attributes/BrawlAttrSet_Energy.h` (Energy/MaxEnergy)

**ExecCalcs**
- `Public/GAS/ExecCalc/BrawlExecCalc_Damage.h`
- `Private/GAS/ExecCalc/BrawlExecCalc_Damage.cpp`
  - Implements CombatMath contract (no per-ability bespoke math)

**Cooldown scaling**
- `Public/GAS/MMC/BrawlMMC_CooldownDuration.h`
  - Central cooldown scaling policy from Spd (single source of truth)

**Ability bases**
- `Public/GAS/Abilities/BrawlGA_Base.h`
- `Public/GAS/Abilities/BrawlGA_BasicBase.h`
- `Public/GAS/Abilities/BrawlGA_UltimateBase.h`
  - Must publish key events (cast, target chosen, etc.) to EventBus

**Ability data**
- `Public/Data/BrawlAbilityData.h`
  - `UBrawlAbilityData : UPrimaryDataAsset`

**Targeting**
- `Public/Data/BrawlTargetingPolicy.h`
  - `UBrawlTargetingPolicy : UDataAsset`
  - Supports retarget/fizzle/last-position
- `Public/Types/BrawlTargetingTypes.h`
  - `FBrawlTargetResult`

**Projectiles**
- `Public/Actors/BrawlProjectileActor.h`
- `Private/Actors/BrawlProjectileActor.cpp`
  - Travel time affects gameplay
  - Emits `Combat.ProjectileSpawned/Impacted` events
---
## 5) Module: BrawlAI
Purpose: smart targeting + BT/StateTree AI + debug traces.

### Files / Classes
- `Public/AI/BrawlAIController.h`
  - `ABrawlAIController : AAIController`
- `Public/AI/BrawlThreatModel.h`
  - `UBrawlThreatModel : UObject` scoring utilities
- `Public/Components/BrawlAIDebugComponent.h`
  - Debug draw toggles and trace export
- BehaviorTree/StateTree assets live in Content; C++ tasks/services if needed:
  - `BTService_BrawlUpdateTarget`
  - `BTTask_BrawlTryCastAbility`

AI must not mutate economy/phase; only unit combat behavior.
---
## 6) Module: BrawlEconomy
Purpose: TFT-like economy/shop/shared pool + trait counting (active vs potential).

### Files / Classes
**Economy**
- `Public/Components/BrawlEconomyComponent.h`
  - Gold, interest, streaks, rewards application

**Progression**
- `Public/Components/BrawlProgressionComponent.h`
  - XP/Level, team-size cap (max 10)

**Shop**
- `Public/Components/BrawlShopComponent.h`
  - Offer list replication (FastArray recommended)
  - Purchase/reroll APIs (server-only commit)

**Shared pool**
- `Public/Subsystems/BrawlSharedPoolSubsystem.h`
  - `UBrawlSharedPoolSubsystem : UWorldSubsystem` (server authority)

**Traits**
- `Public/Components/BrawlTraitComponent.h`
  - Maintains:
    - `ActiveCounts` (field-only, effects applied)
    - `PotentialCounts` (field+bench, UI only)
  - Recompute triggers on roster/placement changes

**Items**
- [Public/Components/BrawlItemInventoryComponent.h](cci:7://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlEconomy/Public/Components/BrawlItemInventoryComponent.h:0:0-0:0)
  - [UBrawlItemInventoryComponent](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlEconomy/Private/Components/BrawlItemInventoryComponent.cpp:35:0-40:1)
  - Owner-only replicated inventory list of `{ ItemId, Count }`

**Data**
- `Public/Data/BrawlEconomyTuningData.h`
- `Public/Data/BrawlShopOddsData.h`
- `Public/Data/BrawlTraitData.h`