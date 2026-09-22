---
name: cleanup-feature-flags
description: >-
  Audit Harness FME feature flags and safely remove a flag from code while
  preserving production treatment. Use when asked to clean up stale flags,
  remove a feature flag from the codebase, hardcode the winning treatment after
  rollout, or archive a launched flag. Do not use for creating or toggling flags
  (use manage-feature-flags). Trigger phrases: flag cleanup, stale flags, remove
  feature flag, archive flag, hardcode treatment, flag debt, FME cleanup.
metadata:
  author: Harness
  version: 1.0.3
  mcp-server: harness-mcp-v2
license: Apache-2.0
compatibility: Requires Harness MCP v2 server (harness-mcp-v2)
---

# Cleanup Feature Flags

Audit Harness FME flags and remove a launched flag from application code while preserving the treatment FME currently serves. Creating, killing, or toggling flags is `/manage-feature-flags`.

## Instructions

Follow ordered phases. Load [references/readiness.md](references/readiness.md) before any verdict and [references/sdk-patterns.md](references/sdk-patterns.md) before searching code.

**Do not invent MCP tools.** There is no `find-stale-flags` or `check-removal-readiness`. Compose `harness_list` / `harness_get` / `harness_execute` plus local search.

**Never guess the forward treatment** from SDK defaults in source. Query FME definitions.

**Stop before mutating.** Do not edit application code, archive, or delete until the user explicitly confirms the cleanup plan.

Prefer **archive** over delete. Do not `harness_delete` until the user confirms delete **after** the flag is archived. “Delete” / “I insist” on an ACTIVE flag means archive, then wait.

### Phase 1: Establish scope

Reuse [scope-establishment.md](../../references/scope-establishment.md). Ask for `org_id` and `project_id` if missing. Do not require `workspace_id`.

```
Call MCP tool: harness_list
Parameters:
  resource_type: "fme_environment"
  org_id: "<org>"
  project_id: "<project>"
```

Identify **critical environments** with the user. Recommend Production-like first. Restate: `Working in org=..., project=..., critical envs=...`

Optional: `harness_describe` with `resource_type: "fme_feature_flag"` if the payload shape is unclear.

### Phase 2: Choose mode

- **Audit** — “what can we clean up?”, flag debt, stale flags, inventory. Run Phase 3. Do **not** edit code. Stop after ranking candidates.
- **Remove** — a named flag to clean up from code. Skip Phase 3 and start at Phase 4 (explore code). If the user picked a flag from a completed audit, Phase 3 is already done — continue at Phase 4. Run Phase 3 only if they also want the landscape.

### Phase 3: Audit candidates (audit mode)

```
Call MCP tool: harness_list
Parameters:
  resource_type: "fme_feature_flag"
  org_id: "<org>"
  project_id: "<project>"
```

Paginate with `offset` / `size` (max 50). Skip `status: ARCHIVED`. Note `createdAt` and `rolloutStatus`.

For each promising candidate (or the user’s shortlist):

```
Call MCP tool: harness_list
Parameters:
  resource_type: "fme_feature_flag_definition"
  org_id: "<org>"
  project_id: "<project>"
  feature_flag_name: "<flag_name>"
```

Do **not** pass `environment_id` on this list — native mode only (`org_id` + `project_id`). It returns definitions across environments. Paginate with `offset` / `limit` (max 100). For a single env, use `harness_get` with `environment_id`.

Rank using [readiness.md](references/readiness.md): `lastImpressionAt` vs 30-day default, kill/default agreement, empty `rules`, no identity/segment includes, `IN_SPLIT` dependents, rollout status, and local code ref count (`safe` requires refs in the application repo).

Grep the **application** workspace (not this skills repo) for each candidate’s flag name using [sdk-patterns.md](references/sdk-patterns.md) before assigning a verdict — FME-only heuristics cannot produce `safe`.

Present a table: flag, verdict (`safe` / `caution` / `blocked`), `lastImpressionAt`, winning treatment per critical env, code-ref count in the application repo. **Do not edit code.** Ask which flag to remove, if any.

### Phase 4: Explore code (remove mode)

Search the **application** workspace (not this skills repo) for the flag key and every pattern in [sdk-patterns.md](references/sdk-patterns.md).

For each hit, record file:line, which treatment branch runs, and side effects.

If keys are built dynamically (`flag-${id}`), stop: automated removal is incomplete.

### Phase 5: Readiness and forward treatment

```
Call MCP tool: harness_get
Parameters:
  resource_type: "fme_feature_flag"
  org_id: "<org>"
  project_id: "<project>"
  feature_flag_name: "<flag_name>"
```

**Always** list definitions across critical environments (even in remove mode if Phase 3 was skipped). Forward treatment requires agreement in **every** critical env — a single-env get is not enough:

```
Call MCP tool: harness_list
Parameters:
  resource_type: "fme_feature_flag_definition"
  org_id: "<org>"
  project_id: "<project>"
  feature_flag_name: "<flag_name>"
```

Paginate with `offset` / `limit` (max 100). Optional drill-down for one env:

```
Call MCP tool: harness_get
Parameters:
  resource_type: "fme_feature_flag_definition"
  org_id: "<org>"
  project_id: "<project>"
  feature_flag_name: "<flag_name>"
  environment_id: "<environment_id>"
```

Apply [readiness.md](references/readiness.md). If **blocked**, stop and list blockers.

**Forward treatment** (FME only):

| Scenario | Forward treatment |
|----------|-------------------|
| All critical envs killed, same `defaultTreatment` | That default treatment |
| All critical envs not killed, same 100% `defaultRule` treatment, empty `rules`, no treatment targeting lists (`keys`, `segments`, `largeSegments`, `ruleBasedSegments`) | That treatment |
| Critical envs disagree on kill or treatment | **NOT SAFE** — stop |
| Prod still has rules or includes | **NOT SAFE** — stop |

### Phase 6: Present the plan and wait

Before any code change, show:

1. Forward treatment and why (definition fields, not code defaults)
2. Code references (file:line)
3. Planned keep vs delete per reference
4. Readiness verdict and warnings (`lastImpressionAt` null, other repos unknown)
5. FME action: archive after merge (not delete)

**Do not proceed until the user explicitly confirms.**

### Phase 7: Remove from application code

Only after confirmation, and only in the user’s application repo:

- Keep the branch that matches the forward treatment; delete the other branch
- Remove flag-only imports, constants, wrappers, tests, and docs
- Do not refactor unrelated code or restyle untouched files

Example (boolean flag, forward treatment `on`). Match the **application** SDK dialect in [sdk-patterns.md](references/sdk-patterns.md): Node/Java pass `(key, flagName)`; browser JS passes `(flagName)` only.

```java
// Before
if ("on".equals(splitClient.getTreatment(key, "new-checkout-flow"))) {
  return renderNewCheckout();
}
return renderOldCheckout();

// After
return renderNewCheckout();
```

```javascript
// Node.js server SDK — Before
function renderCheckout(key) {
  const treatment = splitClient.getTreatment(key, "new-checkout-flow");
  if (treatment === "on") {
    return "new-checkout";
  }
  return "old-checkout";
}

// After
function renderCheckout(key) {
  return "new-checkout";
}
```

### Phase 8: Verify

1. Search again for the flag key and SDK patterns — no leftovers
2. Run the project’s existing build/test/lint if present
3. Open a PR using [references/pr-template.md](references/pr-template.md)

Do **not** archive in FME until the user confirms the change is merged/deployed, unless they asked for archive-only (flag already gone from code).

### Phase 9: Archive in FME (after merge, or archive-only)

Restate scope. Then:

```
Call MCP tool: harness_execute
Parameters:
  resource_type: "fme_feature_flag"
  action: "archive"
  org_id: "<org>"
  project_id: "<project>"
  feature_flag_name: "<flag_name>"
```

If archive returns 409, treat as OPA/governance — do not delete to bypass it.

If the user asks to delete (including “I insist”): archive with the call above while the flag is ACTIVE, then stop. `harness_delete` needs a later message that confirms delete of an already archived flag.

## What NOT to do

- Guess the forward treatment from code
- Call non-existent tools (`find-stale-flags`, `check-removal-readiness`)
- Require `workspace_id` when `org_id` + `project_id` work
- Kill a flag as “cleanup”
- `harness_delete` an ACTIVE flag (even if the user insists)
- Edit code or archive before confirmation
- Create, kill, or restore flags (use `/manage-feature-flags`)
- Change files unrelated to the flag
- Put a README inside this skill folder

## Examples

- "Clean up stale FME flags" — Audit mode: list flags, list definitions, rank, stop.
- "Remove `new-checkout-flow` from this repo" — Remove mode: readiness, plan, wait, then code.
- "Is `dark_mode` safe to archive?" — Readiness only; no code edits unless they confirm.
- "Create a dark-mode flag" — Do **not** use this skill; use `/manage-feature-flags`.
- "Delete `dark_mode` now, I insist" — Archive. Do not `harness_delete` until a second confirm after archive.

## Performance Notes

- Native definition **list** is one call per flag across environments. Prefer it over N gets during audit.
- Flag **list** max page size is 50; paginate.
- `lastImpressionAt` null is not proof of zero traffic (lookup can fail). Default to **caution**.
- Public v4 flag metadata has `createdAt` only — not entity `lastUpdateTime`. Do not invent an updated-at field.
- Keep SKILL.md focused; load references on demand.

## Troubleshooting

### Blocked: environments disagree
Do not pick a treatment. Ask the user to align targeting or exclude an environment from “critical”.

### `lastImpressionAt` is null
Caution, not safe. Ask whether to proceed or wait. Never treat null as “never used” without user agreement.

### Archive returns 409
OPA or dependents. Show the error. Do not delete to work around governance.

### User insists on delete
Archive if ACTIVE. Do not `harness_delete` in the same turn. A second message after archive is required for delete.

### Flag not found
Confirm org/project and exact `feature_flag_name` (case-sensitive). List flags with no name filter.

### No code references but still targeted
Code may live in another repo or a dynamic key. Do not archive as “unused” unless the user accepts that risk.

### MCP auth or empty list
Check Harness MCP v2 is connected and the PAT can read FME in that project.

### Skill not listed in MCP tools
This skill is **not** an MCP tool. MCP only exposes `harness_list` / `harness_get` / `harness_execute`. Load this `SKILL.md` plus `readiness.md` and `sdk-patterns.md` in the agent chat (for example with `@`), then prompt. `/cleanup-feature-flags` auto-completes only if the skill is installed in that workspace; the Cursor Harness plugin may ship `/manage-feature-flags` without this skill.

### Wrong `getTreatment` arity
Node/Java: `getTreatment(key, flagName)`. Browser JS: `getTreatment(flagName)` (key at factory init). Search both; do not treat one-arg JS as Node. See [sdk-patterns.md](references/sdk-patterns.md).

## References

- [readiness.md](references/readiness.md) — verdict table and API fields
- [sdk-patterns.md](references/sdk-patterns.md) — Split/FME evaluation search patterns
- [pr-template.md](references/pr-template.md) — PR body for flag removal
- `/manage-feature-flags` — create, kill, restore, CRUD
