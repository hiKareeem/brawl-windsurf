---
trigger: always_on
description: 
globs: 
---

# GameplayTags Contract

GameplayTags are the primary rules language. Prefer tags + tag queries over enums except where performance is proven critical.

---

## 1) Naming conventions
- Use dot-separated namespaces: `State.CC.Stun`, `Trait.Ranger`, `Ability.Basic.Fireball`
- Use singular nouns for identities: `Trait.Ranger` not `Traits.Rangers`
- Avoid embedding numbers in tags; thresholds live in TraitData.

---

## 2) Required namespaces (minimum)
### Phases
- `Phase.Planning`
- `Phase.Combat`
- `Phase.Rewards`
- `Phase.Shop` (or other non-combat rounds)

### Unit states
- `State.Alive`
- `State.Dead`

### Crowd control and immunities
- `State.CC.Stun`
- `State.CC.Silence`
- `State.CC.Disarm`
- `State.CC.Root` (if applicable)
- `State.Immune.CC`
- `State.Immune.Damage` (if applicable)
- `State.Cleansed` (optional marker)

### Damage typing
- `DamageClass.Physical`
- `DamageClass.Special`
- `DamageClass.Mixed`
- `DamageClass.True`

Elements:
- `Element.Fire`
- `Element.Water`
- `Element.Nature`
- `Element.Ice`
- `Element.Toxic`
- `Element.Earth`
- `Element.Nitro`
- `Element.Light`
- `Element.Void`
- `Element.Psychic`
- `Element.Electric`
- `Element.Wind`

#### Element naming note
- `Element.Nature` is the canonical plant element tag. Do not use `Element.Grass`.

### Identity tags
- `Faction.Player`
- `Faction.Creep`
- `Faction.Summon`

### Content identity
- `Trait.<Name>`
- `Item.<Name>`
- `Ability.Basic.<Name>`
- `Ability.Ultimate.<Name>`

### Data (SetByCaller keys)
`Data.*` tags are reserved as **numeric SetByCaller magnitudes** passed into GameplayEffect specs.
They are not “identity” tags and must not be repurposed for other semantics.

Required keys:
- `Data.Power`
  - Base ability power input for `ExecCalc_Damage` (>= 0)
- `Data.Mod`
  - Multiplicative modifier term for damage (defaults to 1.0 when absent)
- `Data.Flat`
  - Flat add/sub modifier term for damage (defaults to 0.0 when absent)
- `Data.CooldownBaseSeconds`
  - Base cooldown seconds input for the cooldown duration calculation (MMC)

Policy:
- Treat missing `Data.*` keys as the documented defaults.
- If you introduce a new `Data.*` key, update this contract + the owning central policy (`ExecCalc_*` / `MMC_*`) so behavior stays auditable and consistent.

---

## 3) Ownership rules (who can add/remove tags)
- Combat state tags (`State.*`) are applied/removed only by:
  - GameplayEffects / Abilities (server authority)
  - Unit lifecycle code (server authority)
- Phase tags are owned by Match/Phase system (GameState/RoundManager), not by UI.
- Identity tags (`Trait.*`, `Element.*`, etc.) originate from DataAssets and may be augmented by traits/items.

---

## 4) Reserved tags used by core systems
These tags have hard-coded meaning in systems and must not be repurposed:
- `Phase.*` for phase gating/validation
- `State.Dead` for removal/target invalidation
- `State.CC.*` for targeting/cast restrictions
- `DamageClass.*` for ExecCalc formula branch
- `Faction.*` for team logic and PvE vs PvP handling
- `Data.*` SetByCaller keys for central math policies (damage, cooldown, etc.)
  - At minimum: `Data.Power`, `Data.Mod`, `Data.Flat`, `Data.CooldownBaseSeconds`

---

## 5) Tag query guidelines
- Target selection and cast gating must use TagQueries so designers can extend rules without code edits.
- Example: “can cast ultimate if NOT `State.CC.Silence` AND target NOT `State.Dead`”.

---