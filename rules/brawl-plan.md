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
- [x] 8 home boards in planning; up to 4 concurrent arena host boards in combat (**implemented**, automation stabilized)
- [x] Generalize transfer/return to N pairings simultaneously; inactive home boards while away (**implemented**, automation stabilized)
- [x] Combat purchases spawn onto the player’s ActiveBoard bench row (**implemented**)
- [x] Automation: `Brawl.Match.MultiArena.Orchestration.4P_2Arenas.LVL_Sandbox`

### v3.2 ReplicationGraph (required)
- [x] Implement [UBrawlReplicationGraph](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlNet/Private/Net/BrawlReplicationGraph.cpp:15:0-17:1) routing buckets (match-global vs per-board) + spectator routing

### v3.3 Spectator/scouting UX (partial until we decide + implement replication policy changes)
- [x] Board switching + scoreboard (UI/UX polish pending)
  - [x] Spectator visibility policy implemented: spectators see all boards + all shops (shop offers replicate via `COND_NetGroup` + `Brawl.Spectator` net condition group membership)
  - [x] Automation: `Brawl.Net.ReplicationVisibility.SpectatorSeesAllBoardsAndShops.LVL_Sandbox`
- [~] Player avatar + camera integration (GASP + GameplayCameras): (pending 2 weeks)
  - Server-auth scouting state + RepGraph scouted-board routing (**implemented**)
  - Server-auth teleport hooks + BP [ForceBoardCameraMode()](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Game/BrawlPlayerController.h:33:1-33:29) triggers (**implemented**)
  - Camera rig/UX polish remains pending (content/BP work)

  - Scouting invalidation auto-clear (target spectator / `Life <= 0` / removal) + automation (**implemented**)
  - Automation:
    - `Brawl.Match.Scouting.ServerAuth.StateAndFollowActiveBoard.LVL_Sandbox`
    - `Brawl.Match.Scouting.ServerAuth.RejectionsAndRateLimit.LVL_Sandbox`

  - Scouting RPC hardening (requester-side):
  - Reject spectator and eliminated requesters from changing scouting state
  - Log rejections via existing LogRejectedRequest path
  - Automation: `Brawl.Match.Scouting.ServerAuth.RejectionsAndRateLimit.LVL_Sandbox` (expanded)
- Scouting lifecycle improvements:
  - Made SetScoutedBoardActor idempotent to avoid redundant teleports/delegate spam
  - Centralized “clear viewers scouting X” logic into ABrawlGameState::ServerClearScoutingForViewersOfPlayerId
  - Called from SetIsSpectator(true) and RemovePlayerState

### v3.4 Dedicated server readiness + GameLift integration (deferred unless needed)
- [x] Artifact staging directory for GameLift-collected logs (replay + JSONL)
- [x] RPC validation/rate limiting sweep
- [~] Automation: EndMatch cleanup regressions (arena transfer + ghost roster)
  - Ensures EndMatch restores `ActiveBoardActor`, clears arena `GuestPlayerId`, returns transfer plan units, and destroys/unregisters ghost roster units.
---

## v4 (content ramp + balance + UX “playable loop”) — after v3 is stable

### v4.0 Content authoring pipeline validity gate
- [x] AssetManager scanning canonical dirs includes: `/Game/Brawl/Data/Units`, `/Abilities`, `/Items`, `/Traits`, `/Seasons`
- [x] Automation: `Brawl.Content.ValidityGate.AllAssets.LVL_Sandbox`
- [ ] Create a small “vertical slice” roster (example: 8–12 units)
- [ ] Add 1–2 headless/automation stress tests
- [ ] Allow seeing mutliple battles simulatneously (if performance allows)

### v4.0 Season 0 gating (pool composition control)
- [x] Season DataAsset exists: `UBrawlSeasonData` (PrimaryAssetType `Season`) with `AllowedUnits`
- [x] Match selects ActiveSeason: `ABrawlGameState::ActiveSeasonId` (replicated) + [ABrawlGameMode](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Private/Game/BrawlGameMode.cpp:61:0-66:1) sets `DefaultActiveSeasonId`
- [x] SharedPool comp uses allowlist: `UBrawlSharedPoolSubsystem` filters scanned Units
- [x] Automation: `Brawl.Economy.Shop.SeasonGating.OffersAreAllowlisted.LVL_Sandbox`

### v4.3 Balance pass
- [ ] Establish baseline tuning assets
- [ ] Establish a repeatable “balance workflow” (recorded seeds + test scenarios)
- [x] Odd player count rule: unpaired player fights a deterministic ghost roster snapshot of another actively-fighting player
  - Automation: `Brawl.Match.GhostRoster.ArenaResolvedEvent.LVL_Sandbox`
- [x] Ghost roster snapshot cleanup (defensive)
  - Ghost roster unit tracking (`GhostUnitIds`) is best-effort. Cleanup must also remove any unit actors registered on the ghost arena board that are owned by `GhostSourcePlayerId` (fallback).
  - Cleanup order must be deterministic (sort by `FBrawlUnitInstanceId.Value`).
  - After cleanup, clear the ghost arena board guest assignment (`GuestPlayerId = INDEX_NONE`) when appropriate to avoid leaving an arena board in an “assigned guest” state.

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