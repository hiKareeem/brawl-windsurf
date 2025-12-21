# Deferred — Season Content Gating (Interface Only)

## Decision (current)
- We will **not** introduce seasonal gating yet.
- We will keep the current behavior:
- Shared pool enumerates all `Unit` PrimaryAssets via the AssetManager (server-only).
- Designers tune progression/odds/pool copies via `UBrawlEconomyTuningData` / `UBrawlShopOddsData`.

## Why deferred
- Until we have enough content, a Season allowlist adds maintenance overhead without immediate gameplay value.
- We do want a clean future hook so we don't hard-wire pool composition into code.

## Implementation Prompt (for later)

### Objective
Add a **Season asset interface** that can optionally restrict which units (and later items/traits/augments) are considered “in the pool”.

### Constraints
- Do **not** create new modules.
- Keep it **data-driven** (DataAsset), server-authoritative, and module-boundary-safe.
- Do not change any existing gameplay behavior unless a Season asset is explicitly assigned.

### Proposed Asset (BrawlCore)
Add a `UPrimaryDataAsset`:
- `UBrawlSeasonData` (PrimaryAssetType: `Season`)

Fields (v0):
- `TArray<FPrimaryAssetId> AllowedUnitDataIds`
- Optional: `bool bRestrictSharedPoolToAllowlist = false` (or treat empty allowlist as “no restriction”).

Later extensions (not in v0):
- Allowed item ids, trait ids, and any per-cost weighting overrides.

### Integration
1) `UBrawlEconomyTuningData`
- Add optional reference:
- `TObjectPtr<UBrawlSeasonData> SeasonData` (or `TSoftObjectPtr` if you want lighter coupling).

2) `UBrawlSharedPoolSubsystem`
- When rebuilding the pool, if `SeasonData` is provided and `AllowedUnitDataIds` is non-empty:
- Only include units whose `FPrimaryAssetId` is in the allowlist.
- Otherwise:
- Keep current “scan all Unit assets” behavior.

3) Logging/debug
- Log the active Season asset id/name at pool init (server log only).
- (Optional later) include SeasonId in match metadata / event log header.

### Acceptance Criteria
- With no Season assigned: behavior is unchanged.
- With a Season assigned: shared pool contains only allowlisted units, and shop rolls can only produce those units.

### Rules Applied
- brawl-implementation.md (module boundaries; server authority)
- brawl-data-model-contract.md (PrimaryAssetId strategy)
