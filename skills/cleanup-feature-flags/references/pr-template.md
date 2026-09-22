# Pull request template (flag cleanup)

Copy into the application-repo PR. Fill every section.

```markdown
## Summary

Remove feature flag `<flag_key>` from this repository and hardcode treatment `<forward_treatment>`.

## Why this is safe

- FME org/project: `<org>` / `<project>`
- Critical environments checked: `<env names/ids>`
- Readiness verdict: safe | caution | blocked
- Forward treatment source: `isKilled` / `defaultTreatment` / `defaultRule` (not SDK default)
- `impressions.lastImpressionAt`: `<timestamp or null>` (threshold: 30d unless noted)
- Rollout status: `<name or none>`

## Code changes

- Kept: `<files/behavior matching forward treatment>`
- Removed: `<dead branches, tests, constants>`

## FME follow-up

- [ ] Archive via `harness_execute` `fme_feature_flag` `archive` after merge/deploy
- [ ] Do not delete unless explicitly requested after archive
- [ ] Other repos may still evaluate this flag: `<unknown | listed>`

## Test plan

- [ ] Flag key grep is clean in this repo
- [ ] Existing unit/integration tests updated and passing
```
