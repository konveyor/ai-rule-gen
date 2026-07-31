# Java Valid Skip Reasons

When a manifest entry cannot produce a pattern, use one of these reasons in the accountability output. Each reason must include a `detail` field explaining the specific situation.

| Reason | When to use | Example detail |
|---|---|---|
| `method_unchanged` | The API exists in both source and target versions with the same signature and behavior | "RetryHandler interface has identical signature in both HttpClient 4 and 5" |
| `internal_api` | The artifact is an internal/private API not intended for user code | "InternalHttpClient is package-private, not part of public API" |
| `no_code_footprint` | The change has genuinely zero presence in code, config, build files, or scripts | "JVM garbage collector heuristic change — no user-configurable setting" |
| `duplicate` | Another pattern in this batch already covers this artifact | "Already covered by pattern for org.apache.http.HttpResponse (same package rename)" |
| `covered_by_package_rule` | A PACKAGE-level rule already detects this artifact via package rename | "Package org.apache.http renamed — PACKAGE rule covers all classes in this package" |

## Invalid reasons (will be flagged)

- `informational` — too vague. Use `no_code_footprint` with a specific detail.
- `not_detectable` — too vague. Use `no_code_footprint` with a specific detail.
- Empty reason or missing detail.
