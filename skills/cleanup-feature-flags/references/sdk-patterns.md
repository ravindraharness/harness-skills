# FME / Split SDK search patterns

Search the **application** repo for the flag **key string** and these evaluation APIs. Wrapper methods and constants often hide the SDK — search the key even when the SDK call is abstracted.

Harness platform code may use a `FeatureName` / `FMEFeatureName` enum whose string value is the flag key. Search both the enum constant and the string.

## Java / Kotlin

```java
splitClient.getTreatment(key, "flag-key");
splitClient.getTreatment(key, "flag-key", attributes);
splitClient.getTreatments(key, Arrays.asList("flag-key"));
splitClient.getTreatmentWithConfig(key, "flag-key");
splitClient.getTreatmentsWithConfig(key, names);
```

Also: `SplitClient.getTreatment`, factory `SplitFactoryBuilder`, `FeatureFlagName`, `FMEFeatureName`.

## JavaScript / TypeScript (Node)

Server SDK (`@splitsoftware/splitio`). The **key is the first argument** — same shape as Java. Confirmed against [Harness Node.js SDK](https://developer.harness.io/docs/feature-management-experimentation/sdks-and-infrastructure/server-side-sdks/nodejs-sdk/) (`client.getTreatment('key', 'FEATURE_FLAG_NAME')`). Do **not** use the one-arg browser form below for Node.

```javascript
client.getTreatment(key, "flag-key");
client.getTreatment(key, "flag-key", attributes);
client.getTreatments(key, ["flag-key"]);
client.getTreatmentWithConfig(key, "flag-key");
client.getTreatmentsWithConfig(key, ["flag-key"]);
```

Also: `SplitFactory`, `factory.client()`, `require("@splitsoftware/splitio")`, `splitClient.getTreatment`.

## Browser / React (client JavaScript)

Key is bound when the factory/client is created, not on each eval. Confirmed against [Harness JavaScript SDK](https://developer.harness.io/docs/feature-management-experimentation/sdks-and-infrastructure/client-side-sdks/javascript-sdk/) (`client.getTreatment('FEATURE_FLAG_NAME')`). Troubleshooting also documents `getTreatment(split_name, attributes)` for iOS, Android, and JavaScript.

```javascript
client.getTreatment("flag-key");
client.getTreatment("flag-key", attributes);
client.getTreatments(["flag-key"]);
client.getTreatmentWithConfig("flag-key");
client.getTreatmentsWithConfig(["flag-key"]);
useSplitTreatments({ names: ["flag-key"] });
<SplitTreatments names={["flag-key"]}>
```

## Go

```go
client.Treatment(key, "flag-key", attributes)
client.Treatments(key, []string{"flag-key"}, attributes)
client.TreatmentWithConfig(key, "flag-key", attributes)
```

## Python

```python
client.get_treatment(key, "flag-key")
client.get_treatments(key, ["flag-key"])
client.get_treatment_with_config(key, "flag-key")
```

## .NET

```csharp
client.GetTreatment(key, "flag-key");
client.GetTreatments(key, new List<string> { "flag-key" });
client.GetTreatmentWithConfig(key, "flag-key");
```

## Ruby

```ruby
split_client.get_treatment(key, "flag-key")
split_client.get_treatments(key, ["flag-key"])
```

## Also search

- Config, fixtures, and tests that pin the key
- YAML/JSON feature-flag lists
- Comments and docs that name the flag
- Dynamic construction (`"prefix-" + id`, `` `flag-${id}` ``) — if found, stop automated removal

## After removal

Re-run the key search. Remaining hits should be unrelated homonyms or other repos.
