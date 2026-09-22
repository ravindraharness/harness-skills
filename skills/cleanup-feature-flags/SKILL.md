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
  version: 1.0.0
  mcp-server: harness-mcp-v2
license: Apache-2.0
compatibility: Requires Harness MCP v2 server (harness-mcp-v2)
---

# Cleanup Feature Flags

Audit Harness FME flags and remove a launched flag from application code while preserving the treatment FME currently serves. Creating, killing, or toggling flags is `/manage-feature-flags`.

## Instructions

Follow ordered phases. Load [references/readiness.md](references/readiness.md) before any verdict and [references/sdk-patterns.md](references/sdk-patterns.md) before searching code.

**Do not invent MCP tools.** Compose `harness_list`, `harness_get`, and `harness_execute` plus local search.

**MCP resource types** (native FME — use `org_id` + `project_id`; for workspace-scoped flows see `/manage-feature-flags`):

| Tool | `resource_type` | When |
|------|-----------------|------|
| `harness_list` | `fme_environment` | Phase 1 — discover envs |
| `harness_list` | `fme_feature_flag` | Phase 3 — list flags |
| `harness_list` | `fme_feature_flag_definition` | Phase 3/5 — definitions across envs (pass `feature_flag_name`) |
| `harness_get` | `fme_feature_flag` | Phase 5 — flag metadata |
| `harness_get` | `fme_feature_flag_definition` | Phase 5 — one env (pass `environment_id`) |
| `harness_execute` | `fme_feature_flag` | Phase 9 — `action: "archive"` |

**Never guess the forward treatment** from SDK defaults in source. Query FME definitions.

**Stop before mutating.** Do not edit application code, archive, or delete until the user explicitly confirms the cleanup plan.

Prefer **archive** over delete. Do not `harness_delete` an ACTIVE flag. “Delete” / “I insist” means archive first, then wait for a second explicit confirm after archive.

### Phase 1: Establish scope

Reuse [scope-establishment.md](../../references/scope-establishment.md). Ask for `org_id` and `project_id` if missing. Native FME uses org/project scope; `/manage-feature-flags` covers workspace-scoped listing and CRUD when needed.

```
Call MCP tool: harness_list
Parameters:
  resource_type: "fme_environment"
  org_id: "<org>"
  project_id: "<project>"
```

Identify **critical environments** with the user (Production-like first). Restate scope before proceeding.

### Phase 2: Choose mode

- **Audit** — inventory / flag debt. Run Phase 3. Do **not** edit code.
- **Remove** — named flag cleanup. Start at Phase 4 unless Phase 3 was already done.

### Phase 3: Audit candidates (audit mode)

```
Call MCP tool: harness_list
Parameters:
  resource_type: "fme_feature_flag"
  org_id: "<org>"
  project_id: "<project>"
```

Skip archived flags. For each candidate:

```
Call MCP tool: harness_list
Parameters:
  resource_type: "fme_feature_flag_definition"
  org_id: "<org>"
  project_id: "<project>"
  feature_flag_name: "<flag_name>"
```

Rank with [readiness.md](references/readiness.md) and grep the application repo per [sdk-patterns.md](references/sdk-patterns.md).

Present: flag, verdict (`safe` / `caution` / `blocked`), winning treatment per critical env, code-ref count. Ask which flag to remove, if any.

### Phase 4: Explore code (remove mode)

Identify the SDK family (see [sdk-patterns.md](references/sdk-patterns.md)). Search the flag key and relevant eval patterns in the **application** repo.

For each hit: file:line, which branch runs, side effects. Dynamic keys → stop (incomplete automation).

### Phase 5: Readiness and forward treatment

```
Call MCP tool: harness_get
Parameters:
  resource_type: "fme_feature_flag"
  org_id: "<org>"
  project_id: "<project>"
  feature_flag_name: "<flag_name>"
```

List definitions across **every** critical environment (not a single-env get alone):

```
Call MCP tool: harness_list
Parameters:
  resource_type: "fme_feature_flag_definition"
  org_id: "<org>"
  project_id: "<project>"
  feature_flag_name: "<flag_name>"
```

Apply [readiness.md](references/readiness.md). If **blocked**, stop.

Forward treatment comes from FME only — all critical envs must agree. Active targeting in prod-like envs → not safe.

### Phase 6: Present the plan and wait

Show: forward treatment and why, code refs, planned keep vs delete, verdict and warnings, archive-after-merge (not delete).

**Do not proceed until the user explicitly confirms.**

### Phase 7: Remove from application code

Only after confirmation:

- Keep the branch matching forward treatment; remove the other
- Remove flag-only imports, constants, wrappers, tests, docs
- Do not refactor unrelated code

Match the app’s SDK dialect ([sdk-patterns.md](references/sdk-patterns.md)).

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
// Node.js — Before
const treatment = splitClient.getTreatment(key, "new-checkout-flow");
return treatment === "on" ? renderNewCheckout() : renderOldCheckout();

// After
return renderNewCheckout();
```

### Phase 8: Verify

1. Re-search for the flag key — no leftovers
2. Run existing build/test/lint
3. Open a PR using [references/pr-template.md](references/pr-template.md)

Archive in FME only after merge/deploy (or archive-only if code is already gone).

### Phase 9: Archive in FME

```
Call MCP tool: harness_execute
Parameters:
  resource_type: "fme_feature_flag"
  action: "archive"
  org_id: "<org>"
  project_id: "<project>"
  feature_flag_name: "<flag_name>"
```

If archive is blocked by governance, show the error — do not delete to bypass it.

Delete only after archive and a second explicit user confirm.

## What NOT to do

- Guess forward treatment from code
- Kill a flag as “cleanup”
- `harness_delete` an ACTIVE flag
- Edit code or archive before confirmation
- Create, kill, or restore flags (use `/manage-feature-flags`)

## Examples

- "Clean up stale FME flags" — Audit mode; rank and stop.
- "Remove `new-checkout-flow` from this repo" — Remove mode; plan, wait, code, verify, archive after merge.
- "Create a dark-mode flag" — Use `/manage-feature-flags`.

## Troubleshooting

| Issue | Action |
|-------|--------|
| Environments disagree | Do not pick a treatment; ask user to align or narrow critical envs |
| Missing usage data | **Caution** — ask before proceeding |
| Archive blocked | Show error; do not delete around governance |
| User insists on delete | Archive while ACTIVE; delete only on later confirm |
| Flag not found | Confirm org, project, exact flag name |
| No code refs | Other repos or dynamic keys may still evaluate — do not archive as “unused” without user OK |
| Skill not in MCP tool list | Load this skill via `@` in chat; MCP exposes `harness_*` tools only |

## References

- [readiness.md](references/readiness.md) — verdict rules
- [sdk-patterns.md](references/sdk-patterns.md) — code search
- [pr-template.md](references/pr-template.md) — PR body
- `/manage-feature-flags` — create, kill, restore
