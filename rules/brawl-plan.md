---
trigger: always_on
description: 
globs: 
---

## Implementation plan (no code, but “what gets built when”)

### Status legend
- [x] Complete
- [~] Partial / follow-up needed
- [ ] Pending

### Current parallelizable workflows (next)
- [x] **v3.2 ReplicationGraph:** implement [UBrawlReplicationGraph](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlNet/Private/Net/BrawlReplicationGraph.cpp:15:0-17:1) (per-board relevancy + spectator routing)
- [x] **v3.1 Multi-arena:** pairings + transfer/return generalized to N concurrent arenas + elimination wiring
- [~] **v3.3 Spectator/scouting UX:** board switching + scoreboard, built on RepGraph + ActiveBoard

---

## v1 (foundation / sandbox) — consolidated status

- [x] **Foundations (BrawlCore):** GameplayTags taxonomy, stable IDs, log categories, EventBus + event structs
- [x] **Match lifecycle (BrawlMatch/BrawlNet):** planning → combat → rewards loop, match-ended signal, replay start/stop hooks, JSONL event log export hook
- [x] **Grid + placement (BrawlGrid):** board actor + replicated occupancy + server validation + phase gates
- [x] **Units + GAS baseline (BrawlUnit/BrawlAbilities):** unit actor w/ ASC + AttributeSets, global `ExecCalc_Damage`, cooldown MMC, data-driven ability equip
- [x] **Economy + shop (BrawlEconomy/BrawlMatch):** gold/xp/level, seeded shop + shared pool, purchase/reroll/buyXP/sell validation + events
- [x] **Combining:** star combine + combat deferral + tests + structured events
- [x] **Traits + items v1:** traits apply/remove effects deterministically; items inventory + equip/unequip + tests
- [x] **UI (debug/MVVM):** board/shop/traits/round timer viewmodels + debug overlay subsystem
- [x] **Automation:** LVL_Sandbox latent tests cover key contracts (movement gating, equip gating, shop rollback, arena transfer, combat math, projectile event contract, overflow)

---

## v2 (harden 1v1 + complete economy/traits loop) — consolidated status

### v2.0 Arena transfer orchestration (server authoritative)
- [x] Replace sandbox arena with match-owned transfer + return
- [x] Deterministic transfer order (stable `FBrawlUnitInstanceId`)
- [x] ActiveBoardActor switching during combat and restore after
- [x] Automation: arena transfer round-trip invariants

### v2.1 Combat resolution + rewards semantics
- [x] Combat can resolve early only when all relevant arenas are resolved (no premature phase advance)
- [x] Overtime + deterministic tiebreakers (unit count / total health / deterministic id / damage event ordering)
- [x] Apply `State.Victory` to surviving units after their arena fight ends (tag + behavior)
- [x] Rewards phase gates: placement locked; shop actions rejected

### v2.2 Economy completion
- [x] Bench-full purchase prevention (hard stop; no spend / no consume / no spawn)
- [x] Overflow policy (forced grants):
  - place on bench if possible else field overflow
  - resolve overflow to bench in FIFO order when space opens
  - enforce on Planning → Combat (destroy/refund path as needed)
  - emit `Grid.OverflowSpawned` / `Grid.OverflowResolved`
  - automation coverage
- [x] Sell: server-authoritative refund + event emission; combat bench-only restriction + tests

### v2.3 Traits + items “usable loop”
- [x] Traits thresholds apply/remove effects/abilities/tags deterministically (Active = field-only contributors)
- [x] Items: inventory + equip/unequip + replicate equipped item id + tests

### v2.4 Spectate + artifacts + polish gates
- [x] Replay + match event log export hooks exist
- [x] Artifact staging directory for GameLift-collected logs (replay + JSONL)
- [x] Event-log invariants tests (projectile spawned → exactly one impacted; impact before damage)
- [~] Performance audit pass (per `brawl-performance-budgets.md`)

---

## v3 (8-player scaling) — next

### v3.0 Pairings + elimination
- [x] Deterministic seeded pairings (seeded by `(MatchId, RoundIndex)`), implemented so it can be swapped later to Swiss/round-robin
- [x] TFT-like elimination: players have Life, eliminate at `Life <= 0`, match ends when 1 player remains

### v3.1 Multi-arena orchestration
- [~] 8 home boards in planning; up to 4 concurrent arena host boards in combat (**implemented**, automation stabilization in progress)
- [~] Generalize transfer/return to N pairings simultaneously; inactive home boards while away (**implemented**, automation stabilization in progress)
- [~] Combat purchases spawn onto the player’s ActiveBoard bench row (**implemented**, automation stabilization in progress)

### v3.2 ReplicationGraph (required)
- [x] Implement [UBrawlReplicationGraph](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlNet/Private/Net/BrawlReplicationGraph.cpp:15:0-17:1) routing buckets (match-global vs per-board) + spectator routing

### v3.3 Spectator/scouting UX (partial until we decide + implement replication policy changes)
- [~] Board switching + scoreboard; decide spectator economy visibility policy and wire via RepGraph
- [~] **Player avatar + camera integration (GASP + GameplayCameras):**
  - Replicated GASP-derived player avatar
  - Auto-switch to Board camera on `ActiveBoardActor` changes and teleport avatar (TFT-style)
  - Scoreboard click-to-scout forces Board camera and teleports avatar to scouted board
  - Spec: `brawl-player-avatar-camera-contract.md`

### v3.4 Dedicated server readiness + GameLift integration (deferred unless needed)
- [x] Artifact staging directory for GameLift-collected logs (replay + JSONL)
- [x] RPC validation/rate limiting sweep

---

## v4 (content ramp + balance + UX “playable loop”) — after v3 is stable

### v4.0 Content authoring pipeline validation (“add content without code”)
- [ ] Lock/confirm content directories + AssetManager scanning stays canonical:
  - `/Game/Brawl/Data/Units`, `/Abilities`, `/Items`, `/Traits`
- [ ] Add a lightweight “content validity” gate:
  - load/resolve all Unit/Ability/Item/Trait PrimaryAssets
  - assert required fields exist (tags, curves, ability options, etc.)
  - fail fast in editor/automation if invalid
- [ ] Create a small “vertical slice” roster (example: 8–12 units):
  - UnitData: tags + curves + ability options
  - AbilityData: targeting policy + projectile policy where needed
  - TraitData/ItemData: enough to validate trait/item loops at scale
- [ ] Add 1–2 headless/automation stress tests:
  - spawn ~160 units worst-case approximation and run N seconds
  - assert: no NaNs, no runaway memory, stable event log invariants
- [ ] Allow seeing mutliple battles simulatneously (if performance allows)

### v4.0 Season 0 gating (real feature; pool composition control)
- [ ] Add a Season DataAsset (PrimaryDataAsset) that allowlists Unit PrimaryAssetIds
  - Season 0 = initial curated allowlist
- [ ] Server selects ActiveSeason (default Season 0) and SharedPool composition is derived from:
  - ActiveSeason allowlist (not “all scanned units”)
  - Copies-per-unit from economy tuning
- [ ] Add automation coverage:
  - with Season 0 allowlist = {A,B,C}, shop rolls never produce units outside {A,B,C}
  - shared pool counts only exist for allowlisted units
- [ ] Add minimal debug visibility:
  - log ActiveSeason at match start + log allowlist size

### v4.1 Gameplay breadth (still rules-driven, minimal new systems)
- [ ] Expand targeting policies (data-driven selection modes + deterministic tie-breaks)
- [ ] Expand trait tiers/effects using existing TraitData semantics
- [ ] Add creep rounds / PvE round types (RoundSet-driven)

### v4.2 UX polish (still no gameplay rules in UI)
- [ ] Move from debug-first overlay to “playable” UI flows:
  - shop + board + traits + round timer + scoreboard
- [ ] Spectator/scouting UX polish (board switching, inspect unit loadouts, etc.)

### v4.3 Balance pass (data-only where possible)
- [ ] Establish baseline tuning assets:
  - economy tuning, shop odds, damage tuning, cooldown tuning
- [ ] Establish a repeatable “balance workflow” (recorded seeds + test scenarios)
- [ ] Odd player count rule (TFT-like): one unpaired player fights a deterministic “ghost roster” snapshot of another actively-fighting player (rule definition + tests pending)

---

## v5 (productionization + live-ops hooks + scale hardening)

### v5.0 Dedicated server readiness
- [ ] Artifact staging + GameLift integration
- [ ] Match start/end metadata + graceful shutdown sequencing
- [ ] RPC abuse mitigation sweep (rate limiting + validation checklist compliance)

### v5.1 Performance + replication scaling gates
- [ ] Profile 8p worst-case and close top offenders
- [ ] RepGraph refinement (bandwidth priorities + spectator policy finalized)
- [ ] Reduce/centralize ticking (avoid per-unit tick where possible)