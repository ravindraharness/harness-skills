# Flag removal readiness

Compose this from Harness MCP v2 + local grep. Do not call tools that do not exist.

Default impression stale threshold: **30 days**. Ask before using a different window.

## Public v4 fields (native MCP)

Use `org_id` + `project_id`. Prefer these over `workspace_id`.

| Field | Resource | Use |
|-------|----------|-----|
| `createdAt` | `fme_feature_flag`, `fme_feature_flag_definition` | Age |
| `status` `ACTIVE` / `ARCHIVED` | `fme_feature_flag` | Skip archived |
| `rolloutStatus` `{id, name}` | `fme_feature_flag` | Reference on the flag — names like Permanent → blocked or caution. Full catalog (with `description`) via `fme_rollout_status` list |
| `isKilled`, `defaultTreatment`, `baselineTreatment`, `defaultRule`, `trafficAllocation`, `rules`, treatment `keys` / `segments` / `largeSegments` / `ruleBasedSegments` | `fme_feature_flag_definition` | Forward treatment and targeting (`trafficAllocation` is experiment participation 0–100, not treatment split — use `defaultRule` for %) |
| Matcher type `IN_SPLIT` | definition `rules` | Dependent flags → blocked |
| `impressions.lastImpressionAt` | definition | Primary staleness (last SDK evaluation) |
| List definitions by `feature_flag_name` only | `harness_list` `fme_feature_flag_definition` | All environments in one call (`org_id`+`project_id` only; paginate `offset`/`limit`, max 100) |
| `harness_get` one definition | `fme_feature_flag_definition` | Requires `environment_id` — list is env-agnostic; get is per-env |

## Not on public v4 (do not require)

| Field | Reality |
|-------|---------|
| Flag metadata `lastUpdateTime` / `updatedAt` | Exists on admin `TestMetadataDTO` (`lastEntityUpdateTime`). **Not** mapped on v4 `FeatureFlag`. |
| Definition `lastUpdateTime` | On v2 `SplitExternal`. **Dropped** on v4 definition DTO. |
| `lastTrafficReceivedAt` | v2 only. v4 uses `lastImpressionAt`. |

If the user asks to sort by “last edited”, say that field is not on native MCP yet. Fall back to `createdAt` + `lastImpressionAt`.

## Verdicts

| Verdict | When |
|---------|------|
| **blocked** | Critical envs disagree on `isKilled` or winning treatment; prod-like env still has `rules` or any non-empty treatment targeting list (`keys`, `segments`, `largeSegments`, `ruleBasedSegments`); `IN_SPLIT` dependents; rollout status is permanent (or clearly “do not remove”) |
| **caution** | `lastImpressionAt` is null or newer than the threshold; `createdAt` is recent (default: under 14 days); only some envs are 100% one treatment; killed in prod but live elsewhere; this repo has no refs (other repos unknown) |
| **safe** | All critical envs share the same winning treatment (or all killed with the same `defaultTreatment`); empty `rules` and all four treatment targeting lists empty in those envs; `lastImpressionAt` older than threshold; local code refs found in this repo |

Null `lastImpressionAt` means never evaluated **or** impressions lookup failed on safe reads. That is **caution**, not **safe**. The `impressions` object is always present; only `lastImpressionAt` inside it may be null. A failed impressions lookup on **get/list** may return an API error rather than null — treat errors as **caution**, not proof of zero traffic.

Verdicts assess FME and local-repo readiness only. Present `safe` / `caution` / `blocked` in audit mode before user confirmation. Do not edit code or archive until the user explicitly confirms.

## Forward treatment (FME is source of truth)

1. If `isKilled` is true in every critical env: use `defaultTreatment` (must match across those envs).
2. Else if every critical env has empty `rules`, all treatment targeting lists empty (`keys`, `segments`, `largeSegments`, `ruleBasedSegments`), and `defaultRule` is 100% one treatment: use that treatment.
3. Else: **NOT SAFE**.

Never use the SDK default in application code as the forward value.

## Audit ranking (highest priority first)

1. `blocked` last
2. Archived already — skip
3. `caution` with old `lastImpressionAt` and no local code refs (archive-only candidate — other repos may still evaluate)
4. `safe` with local code refs (remove-from-code candidate)
5. `caution` — present with reasons, do not auto-remove
