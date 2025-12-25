---
trigger: always_on
description: 
globs: 
---

## Implementation plan (no code, but “what gets built when”)

---

### Parallelization (2+ engineers)
Once GameplayTag taxonomy is locked (Milestone 0), the following workstreams can proceed in parallel with low merge-conflict risk if each stays within its module.

#### Workstream A/B (Milestone 3: Combat loop v1 + BrawlAI start) (Next)
- Add a BrawlCore match-context interface (implemented by `ABrawlGameState`) so Abilities/AI can emit EventLog base fields without depending on BrawlMatch
- BrawlAbilities: real TargetingPolicy + TargetResult types; ProjectileActor; GA bases emit structured combat events via EventBus
- Start BrawlAI module (AIController + threat model + debug), focused only on combat decisions

---

v1 (in repo foundation/sandbox)

### Milestone 0: Foundations
- Establish GameplayTag taxonomy (see “rules file” section below)
- Define DataAssets:
  - `UnitData` (cost, rarity, base tags, visuals, ability options, curve refs)
  - `AbilityData` (cooldown, power, element/type tags, targeting mode, projectile params)
  - `ItemData` (1-slot item effects/ability grants)
  - `TraitData` (thresholds + effects)
  - `RoundData` (phase durations, round type, creep wave, shop-only rounds, etc.)
- Create log categories + “combat log event” struct format

### Milestone 1: Grid + planning interactions (server authoritative)
- Board/bench representation, swap rules, placement validation
- Server-confirmed drag/drop actions
- Bench-full purchase prevention
- Overflow policy: spawn at (0,0), enforce capacity at end of planning, destroy+refund

#### Note (Milestone 1 dependency)
Some Milestone 1 items depend on Economy scaffolding (unit cost/refund, purchase pipeline):
- Bench-full purchase prevention
- Overflow destroy+refund

If Economy is not implemented yet, it is acceptable to complete placement-only rules in Milestone 1 and defer purchase/overflow enforcement to Milestone 4 once unit ownership + cost/refund are authoritative.

### Milestone 2: Units + GAS baseline
- Unit actor with ASC + attribute sets:
  - Combat stats: HP/Atk/SpAtk/Def/SpDef/Spd, Energy, etc.
- “Grant 1 basic + 1 ultimate” from selected loadout
- Ability toggling via replicated “equipped ability IDs” (UI writes to server, server grants/removes)

### Milestone 3: Combat loop (1v1 sandbox)
- Phase manager: planning → combat → rewards
- Combat start: lock field units, allow bench-only movement
- AI:
  - basic “acquire target + cast on cooldown”
  - targeting rules per ability (retarget/fizzle/last-known-position)
- Projectile travel time affects hit timing
- Trigger ordering policy: **OnHit → OnDamaged → OnDeath** (enforced consistently)

### Milestone 4: Economy + shop + shared pool + combining
- Gold/interest/streaks + XP/level
- Shop offers (Blueprint-editable odds tables)
- Shared unit pool consumption/return
- 1–3 star combining rules

### Milestone 5: Traits + items v1
- Threshold-based traits with designer-authored effects:
  - apply GameplayEffects
  - grant abilities
  - apply global player/team modifiers
- 1-slot items, unique items, optional consumables

### Milestone 6: Spectate + replay + debug tooling
- Spectator can view all boards (initially 1v1 arena)
- Server records replays (DemoNetDriver)
- Combat log viewer (in-editor and/or in-game dev panel)
- AI debug overlays: current target, threat, decision traces
- (UI) Implement `BrawlUI` MVVM ViewModels for debug/spectate:
  - Board/Occupancy, RoundTimer, DamageRecap (later)
  - No gameplay rules in UI (presentation only)
  - Prefer CommonUI for eventual controller support if possible

### Milestone 7: Scale to 8 players
- Arena layout strategy:
  - easiest: multiple arenas placed far apart in one world + ReplicationGraph relevancy filtering
  - pairing system + “copy opponent team into arena” approach (TFT-like)
- Performance pass for 160 units worst case

---

## v1 (in-repo foundation / sandbox)
- Core: tags + stable IDs + EventBus + event structs
- Match: phase/round loop, combat tick gating, match-ended signal, replay recording hook, JSONL event log export
- Grid: board actor + occupancy FastArray replication + server placement validation
- Unit/Abilities: Unit actor w/ ASC, global damage ExecCalc, cooldown MMC, projectile + targeting policies
- Economy: gold + shop offers + shared pool + trait counts (active vs potential)
- UI: MVVM ViewModels + debug overlay subsystem
- Net: Replay subsystem exists; ReplicationGraph stub exists

---

## v2 (next) — harden 1v1 + complete economy/traits loop

### v2.0 Arena transfer orchestration (server authoritative)
- Replace sandbox “fake arena” with match-owned transfer + return.
- Deterministic transfer order: sort by `FBrawlUnitInstanceId.Value`.
- Preserve/restore original `(BoardActor, FBrawlGridCoord)` per transferred unit.
- During combat, both players’ `ActiveBoardActor` points to the arena host board; after combat, restore to home boards.
- Add automation test(s): transfer → combat → return invariants.

### v2.1 Combat resolution + rewards semantics
- Combat can end early only when ALL arena fights are complete (no premature phase advance while any fight is ongoing).
- When an arena fight ends, surviving units get `State.Victory` to disable AI/targeting and allow victory presentation.
- Tie rules:
  - Sudden death overtime (TFT-like, designer-defined; e.g., limited overtime with escalating pressure).
  - Backup deterministic tiebreak (if needed): more units alive → more total unit health → stable UnitId/metric.
- Rewards phase:
  - placement locked
  - shop actions not allowed (teardown + rewards distribution + next-round setup)

### v2.2 Economy completion
- Bench-full purchase prevention (server authoritative; hard stop).
- Overflow policy:
  - spawn overflow at canonical field `(0,0)` when needed
  - enforce capacity at end of planning: destroy excess + refund
- Sell: refund policy + shared pool return semantics.
- Automation tests: bench-full, overflow destroy+refund, pool accounting.

### v2.3 Traits + items “usable loop”
- Traits: thresholds apply gameplay effects/abilities/tags deterministically, based on Active (field-only) contributors.
- Items: consume from inventory, apply/remove item-authored effects, replicate equipped item id.
- Tests for trait activation and item equip.

### v2.4 Spectate + artifact staging (optional)
- Add artifact staging directory for GameLift-collected server logs (replay + JSONL).
- Minimal spectator/debug flows without ReplicationGraph (sandbox world layout keeps boards close).

### Deferred (until v3)
- ReplicationGraph relevancy (required for 8-player scaling).
- Full scouting/spectator relevance policies.

### v3.0 Pairings + elimination
- Define pairing schedule and avoidance rules.
- Define elimination / player HP / match end conditions (recommended: last player standing).

### v3.1 Multi-arena orchestration
- 8 home boards in planning.
- Up to 4 concurrent arena host boards in combat.
- Generalize transfer/return to multiple simultaneous pairings.

### v3.2 ReplicationGraph (required)
- Implement `UBrawlReplicationGraph` routing:
  - per-board actors
  - match-global actors
- Per-connection relevance:
  - player sees their active arena (and optionally their home board)
  - spectator can opt into all boards

### v3.3 Spectator/scouting UX
- Spectator camera + board switching.
- Decide economy visibility for spectator/scouting.
- Each board has a host camera and a 180°-rotated guest camera for scouting/combat viewing.
- Spectators see everything (including shop offers) and the scoreboard can show: level/xp/gold, roster, items, trait synergies.

---

# 2) Proposed v2 trajectory (expanded, promptable)

This is a “v2 roadmap” you can paste into [brawl-plan.md](cci:7://file:///C:/Users/hi/.codeium/.windsurf/rules/brawl-plan.md:0:0-0:0) and also slice into implementation prompts.

## v2.0 — Arena transfer + combat correctness (server authoritative)
- **v2.0.1**: Complete B2 (transfer + return + active board switching)
- **v2.0.2**: Arena transfer automation test(s)
- **v2.0.3**: Define combat end conditions (timer vs early-out) and winner detection
- **v2.0.4**: Rewards-phase rules for what’s allowed (placement locked; purchases allowed? optional)

## v2.1 — Economy completion (rules-driven, server validation)
Focus on finishing the contract items you already cite in docs:
- **v2.1.1**: Bench-full purchase prevention (hard stop)
- **v2.1.2**: Overflow policy implementation (spawn at `(0,0)` field, then destroy+refund at end of planning if still over cap)
- **v2.1.3**: Selling returns unit to pool + refunds gold (confirm exact refund policy)
- **v2.1.4**: Automation tests:
    - bench-full purchase is denied
    - overflow destroy+refund happens deterministically
    - pool accounting is correct on buy/sell/overflow

## v2.2 — Traits/items “usable loop” (designer-authored knobs)
You have trait counting scaffolding; this step is “make it actually affect gameplay”.
- **v2.2.1**: Trait threshold tiers (data-driven) + apply/remove effects deterministically
- **v2.2.2**: Item equip consumes inventory + applies item-authored effects (and removes on swap)
- **v2.2.3**: Tests:
    - trait effects apply only for active contributors (field-only)
    - item effects apply and replicate equipped item id

## v2.3 — Spectate + artifacts + debug ergonomics
Without going full RepGraph yet:
- **v2.3.1**: spectator visibility policy (even if “everything relevant” for now)
- **v2.3.2**: GameLift artifact staging directory (optional but useful early)
- **v2.3.3**: minimal in-game debug overlay flows (board switching, selected unit details)

## v2.4 — “Polish gates” before v3 scaling
- **v2.4.1**: event log invariants tests (projectile spawned → exactly one impacted, damage ordering)
- **v2.4.2**: performance audit on 2 boards / 2 teams (no obvious per-tick heavy scans beyond combat driver)

---

# 3) Proposed v3 plan (8-player scaling) + sequencing

v3 is where RepGraph stops being optional.

## v3.0 — 8-player match loop (pairings + board lifecycle)
- **v3.0.1**: Define pairing system + scheduling
    - swiss-like? random? round-robin-ish?
    - avoid repeats? (optional)
- **v3.0.2**: Define elimination / player HP / match end conditions
    - TFT-like player HP damage per loss?
    - or “roundset exhaustion” only? (probably insufficient for 8p)

## v3.1 — Multi-arena orchestration
- **v3.1.1**: Create 4 concurrent arena host boards per round
- **v3.1.2**: Transfer/return generalized to N pairings simultaneously
- **v3.1.3**: “Inactive home boards” behavior
    - empty/non-interactive while player is away in combat
    - purchases during combat/rewards must spawn onto the *current arena board* bench row (per your contract) — decide if v3-only

## v3.2 — ReplicationGraph (no longer deferred)
- **v3.2.1**: Wire RepGraph enablement (INI)
- **v3.2.2**: Routing buckets:
    - per-board actors (board + occupants + projectiles)
    - match-global (GameState, maybe RoundManager state)
- **v3.2.3**: Per-connection relevance:
    - player sees: own active arena + (optionally) own home board
    - spectator sees: selectable subset or all

## v3.3 — Spectator/scouting UX (depends on RepGraph)
- **v3.3.1**: spectator camera + “switch board” controls
- **v3.3.2**: economy visibility policy for spectators (yes/no; partial)

## v3.4 — Dedicated server readiness + GameLift integration
- **v3.4.1**: artifact staging + log upload model (GameLift-collected logs)
- **v3.4.2**: match start/end metadata + graceful shutdown sequencing
- **v3.4.3**: abuse mitigation/rate limiting for RPCs (validation checklist sweep)