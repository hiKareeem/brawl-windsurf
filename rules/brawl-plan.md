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

### Milestone 7: Scale to 8 players
- Arena layout strategy:
  - easiest: multiple arenas placed far apart in one world + ReplicationGraph relevancy filtering
  - pairing system + “copy opponent team into arena” approach (TFT-like)
- Performance pass for 160 units worst case