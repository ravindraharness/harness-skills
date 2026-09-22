# Flag removal readiness

Use Harness MCP plus local code search. Default stale threshold: **30 days** (ask before changing).

Scope with `org_id` + `project_id`.

## What to check

For each **critical environment**:

- Flag is not archived
- Same winning treatment (or same kill/default behavior) across all critical envs
- No active targeting rules, segments, or dependent-flag rules still in play
- Recent usage: prefer flags with no recent evaluations; treat missing usage data as **caution**, not proof of zero traffic
- Rollout status: permanent or “do not remove” → **blocked** or **caution**

Also grep the application repo for the flag key before **labeling** a flag **safe**.

## Verdicts

| Verdict | When |
|---------|------|
| **blocked** | Critical envs disagree; prod-like env still targeted; dependent flags; clearly permanent rollout |
| **caution** | Missing or recent usage data; young flag; partial rollout; no code refs in this repo (others unknown) |
| **safe** | All critical envs agree on one treatment with no targeting left; usage looks stale; code refs found here |

## Forward treatment

FME definitions are the source of truth — never the SDK default in code.

1. If killed everywhere: use the shared default treatment.
2. Else if every critical env is 100% on one treatment with no rules left: use that treatment.
3. Else: **not safe** — stop.

## Audit ranking

1. Skip archived flags
2. Prefer **safe** with code refs (remove-from-code candidates)
3. **Caution** with reasons — do not auto-remove
4. **Blocked** last
