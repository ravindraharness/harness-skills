# Code search patterns

Always search the flag **key string** first. Wrappers and enums often hide the SDK call.

Search the **application** repo (not this skills repo).

## Find the SDK family

Check dependencies and imports (`package.json`, `go.mod`, `pom.xml`, `build.gradle`):

| Signal | Grep for |
|--------|----------|
| FME / Split server | `getTreatment`, `@splitsoftware/splitio` |
| FME / Split browser | `getTreatment("`, `useSplitTreatments` |
| OpenFeature | `getBooleanValue`, `getStringValue`, `@openfeature/` |
| Harness FF SDK | `@harnessio/ff-`, `boolVariation`, `variation(`, `useFeatureFlag` |
| Custom wrapper | flag key + `isEnabled`, `getFlag`, `FeatureFlagService` |

**Node vs browser Split:** server uses `getTreatment(key, "flag-key")`; browser uses `getTreatment("flag-key")` only.

**Classic FF vs FME:** this skill’s readiness and archive use FME. If the repo only uses `@harnessio/ff-*` and FME has no such flag, stop.

## Also search

- Tests, config, fixtures, comments
- Dynamic keys (`"prefix-" + id`, `` `flag-${id}` ``) — if found, stop automated removal

## After removal

Re-run the key search. Remaining hits should be homonyms or other repos.
