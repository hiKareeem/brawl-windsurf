---
trigger: always_on
description: 
globs: 
---

# Style_and_Workflow.md (Brawl)

This file is a style/workflow + assistant-behavior guide. It is NOT the source of truth for gameplay rules.

## Precedence
If anything here conflicts with other docs, follow them in this order:
1) `rules.md`
2) `brawl-directories.md`
3) `brawl-replication-contract.md`
4) `brawl-gameplay-tags-contract.md`, `brawl-data-model-contract.md`, `brawl-combat-math-contract.md`, `brawl-event-log-schema.md`
5) This file

---

## Assistant behavior rules
- **No build/run/launch:** Do not attempt to compile, run, package, or launch the project. I will do it.
- **Rule citations (lightweight):** When making an architectural decision, rejecting an approach, or implementing something security/replication-sensitive, include a short **“Rules Applied:”** line citing the relevant document section(s) (by file name + heading).
    - Do NOT list rules for trivial edits (typos, renames, formatting).
- **Changes must be explicit:** When generating implementation output, always include:
    - Files added/modified (paths)
    - New classes/structs/assets introduced (names + module)
    - Replication impact summary (if any)
- **No invented architecture:** Do not create new modules or rename established ones unless explicitly directed. Follow `brawl-directories.md`.
- **Confirm:** When in doubt, ask for confirmation or clarification. I would rather have you ask than make a bad decision.
- **Revise:** If my prompts deviate from what is mentioned in the docs, confirm the change with me, then suggest any necessary changes to the docs/rules you'd like me to make to ensure clarity moving forward. I will then edit the rules for you.
- **This is early in development:** we don't need to worry about breaking existing Blueprints or other functionality, there is no original client to maintain yet, we're developing it now and I would rather have things break temporarily and fix them then leaving unused, dirty, legacy code in the codebase.

---

## Unreal C++ conventions
- Use Unreal reflection macros correctly: `UCLASS`, `USTRUCT`, `UENUM`, `UPROPERTY`, `UFUNCTION`.
- Prefer **composition** (components/subobjects) over deep inheritance.
- Prefer event-driven logic over Tick; if ticking is unavoidable, centralize and keep it minimal.
- Use forward declarations in headers where possible; keep headers minimal; put heavy includes in `.cpp`.
- Use `TObjectPtr<>` for `UObject` UPROPERTY references where appropriate (UE5 style).
- Use `const` correctness and pass heavy structs by `const&`.
- Use UPROPERTY macros to categorize Blueprint readable properties under a "Brawl" header (Category = "Brawl|x")

---

## Naming conventions (UE-friendly)
- **Classes/Structs/Enums:** PascalCase (`ABrawlUnitCharacter`, `FBrawlGridCoord`)
- **Functions:** PascalCase (`RecomputeTraits`, `ServerMoveUnit`)
- **UPROPERTY members:** PascalCase (consistent with reflection/Blueprint exposure)
- **Private non-UPROPERTY members + locals:** PascalCase
- **Booleans:** `bDead`, `bPendingDestroy` (no "is" for boolean names if possible, IsDead? is a question and reserved for bool returning functions)
- **GameplayTags:** follow `brawl-gameplay-tags-contract.md` naming (e.g., `Phase.Planning`, `State.CC.Stun`)
- **Blueprint assets:**
    - Actors: `BP_...`
    - Widgets: `WBP_...`
    - DataAssets: `DA_...`
    - Curves/Tables: `CT_...`, `DT_...`

---

## Formatting
- Braces on next line for functions and class bodies:
  ```cpp
  void Foo()
  {
      ...
  }
- Indentation:  Tabs preferred, do not fret over spaces if found in any files.
- Keep functions short and readable; extract helpers instead of long monoliths.
- One blank line between function definitions.
- Comments: only for non-obvious intent; avoid narrating the code.

---

## Module and folder usage
- All source code must live under the module folders defined in `brawl-directories.md`.
- Do not reintroduce a monolithic `Public/AbilitySystem/...` layout inside a single module.
- Public headers = types meant to be used by other modules.
- Private headers = internal-only; prefer `.cpp`-local helpers when possible.

---

## GAS authoring rules (high level)
(Details live in `brawl-ability-authoring-guide.md` + `[brawl-combat-math-contract.md](cci:7://file:///C:/Users/hi/.codeium/.windsurf/rules/brawl-combat-math-contract.md:0:0-0:0)`.)
- Abilities use base classes (`GA_*Base`) and data from `AbilityData`.
- Damage/healing numbers are computed in ExecCalcs (global math), not hand-rolled per ability.
- Targeting logic is NOT done in ExecCalcs; it lives in targeting policies/ability tasks/AI utilities.
- GameplayCues are presentation-only (no gameplay logic).

---

## Replication and security rules (high level)
(Details live in `brawl-replication-contract.md` + `brawl-server-validation-checklist.md`.)
- Dedicated server authoritative only.
- Client actions are requests; server validates and commits state.
- Never trust client-supplied outcomes (damage, RNG, legality).
- Prefer replication-efficient containers (FastArray) for lists (shop offers, board occupancy) as specified in the contract docs.

---

## Logging and diagnostics
- Use dedicated log categories defined in `BrawlLog.h`:
    - `LogBrawlMatch`, `LogBrawlEconomy`, `LogBrawlCombat`, `LogBrawlAI`, `LogBrawlGrid`, `LogBrawlNet`, `LogBrawlUI`
- Log with context: stable IDs (UnitInstanceId), player/team IDs, phase/round index when relevant.
- Prefer structured event logging via the EventBus/EventLog when behavior must be auditable or testable (see `brawl-event-log-schema.md`).

---

## Data-driven content workflow
- Systems in C++; content in DataAssets/GAS assets/Blueprint.
- Adding new units should primarily be “fill out a DataAsset + set BP visuals”, not code changes (see `brawl-data-model-contract.md`).
- Designer-editable tuning must be exposed via DataAssets/Blueprint-editable fields where required by the contracts.

---

## UI (MVVM) rules
- Widgets are presentation only.
- ViewModels are C++ and are the only place widgets bind to game data.
- Do not place gameplay rules/validation inside widgets.

---

## Localization readiness
- All player-facing strings must use LOCTEXT/NSLOCTEXT (or equivalent UE localization macros).
- Avoid hard-coded FString UI labels in gameplay code.

---

## Testing expectations
(Details live in `brawl-testing-matrix.md`.)
- Add automation tests for core invariants when introducing new systems:
    - damage math sanity, cooldown scaling, trait counts, overflow rules, trigger ordering
- Prefer headless tests for combat/economy logic when possible.

---

## Git and workflow conventions
- Commit prefixes:
    - `feat:`, `fix:`, `perf:`, `docs:`, `style:`, `refactor:`, `test:`, `chore:`
- Branch naming:
    - `feature/<name>`, `fix/<name>`, `refactor/<name>`
- Keep commits focused (one subsystem change per commit where feasible).
- Update docs (`docs:`) when introducing new contracts or altering an existing one.

---