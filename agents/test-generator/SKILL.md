---
name: test-generator
description: Generates compilable test source code that triggers Konveyor analyzer rule patterns. Use when test data is needed for kantra rule validation.
---

# Test Generator

You generate test application source code that triggers Konveyor analyzer rules. The test code must be compilable and must contain code that EXACTLY matches each rule's `when` condition pattern.

## Inputs

- `rules_dir` — Directory containing rule YAML files
- `tests_dir` — Directory containing scaffolded test structure
- `groups` — (optional) List of groups to generate. If provided, skip scaffold and manifest steps. If omitted, run scaffold first and read manifest.json to get the full group list. Each group has:
  - `name` — Group name
  - `data_dir` — Path to group's data directory
  - `rule_ids` — Rule IDs in this group
  - `files` — Files to generate (path + purpose)

## Returns

- `groups_completed` — Number of groups processed
- `files_written` — Number of files written
- `suspected_kantra_limitations` — (optional) List of objects `{rule_id, reason}` where Maven Central confirmed no plain-semver version exists for the artifact. The rule is still written as `java.dependency`; this signals the orchestrator to pre-classify these rules for the validator.

## Permissions

| Operation | Pattern | Purpose |
|-----------|---------|---------|
| shell | `go run ./cmd/scaffold *` | Create test directories (standalone invocation) |
| shell | `go mod *` | Resolve Go module dependencies |
| shell | `go doc *` | Look up Go API signatures |
| shell | `curl -s "https://search.maven.org/solrsearch/select*"` | Look up real artifact versions on Maven Central for java.dependency test data |
| read | `output/**` | Read rule YAML and scaffold manifest |
| read | `agents/test-generator/references/**` | Read test data guide |
| read | `agents/test-generator/references/languages/**` | Read language-specific test data guide |
| write | `output/**` | Write test source files |
| edit | `output/**` | Fix test files during compilation |

**Use only Go commands (e.g. `go run`, `go mod`, `go doc`) and `curl` as listed in the permissions table — do NOT use `python`, `python3`, `node`, or any scripting language runtime.** This is a Go project. Every unnecessary shell command triggers a permission prompt that blocks the autonomous pipeline.

## References

Read these before starting:
- [`references/test-data-common.md`](references/test-data-common.md) — Shared test data contract: goal, requirements, source API rule, output format, manifest.json structure, XML sanitization
- `references/languages/<language>/test-data-guide.md` — Language-specific project structure, condition matching table, dependency resolution, compilation check
- `references/templates/<language>/` — Minimal build/source templates. Start from templates, then inject rule snippets.

## Workflow

### 1. Scaffold (skip if `groups` provided)

If `groups` was provided in the inputs, skip to step 3 — scaffold and manifest were already handled.

Otherwise, run the CLI to create test structure and manifest:

```bash
go run ./cmd/scaffold --rules <rules-dir> --output <tests-dir>
```

This creates:
- `.test.yaml` files (kantra test definitions)
- Data directories for each test group
- `manifest.json` describing what source files to generate

### 2. Read manifest.json

The manifest tells you exactly what files to generate. All paths are **relative to `tests_dir`**:

```json
{
  "language": "java",
  "groups": [
    {
      "name": "web",
      "data_dir": "data/web",
      "test_file": "web.test.yaml",
      "rule_count": 3,
      "providers": ["java"],
      "files": [
        {"path": "data/web/pom.xml", "file_type": "xml", "purpose": "build"},
        {"path": "data/web/src/main/java/com/example/Application.java", "file_type": "java", "purpose": "source"}
      ],
      "rule_ids": ["rule-00010", "rule-00020", "rule-00030"]
    }
  ]
}
```

**Always join `tests_dir` with each `files[].path` when writing:** `<tests_dir>/<files[].path>`. Do NOT write files relative to the current working directory — they will land at the repo root instead of inside `output/<migration>/tests/`.

### 3. Generate source code for each group

For each group:

1. Read the rules referenced by `rule_ids` from the rules directory
2. Look at each rule's `when` condition to understand what pattern must be matched
3. Copy the minimal language template files from `references/templates/<language>/` and write them to `<tests_dir>/<files[].path>`
4. Inject rule-specific snippets into template placeholders (for example `{{RULE_SNIPPETS}}`)
5. Write the **build file** (purpose: `build`) to `<tests_dir>/<path>` and **source file** (purpose: `source`) to `<tests_dir>/<path>`

**Source code requirements:**
- The project must be COMPLETE and COMPILABLE
- For EACH rule, include code that EXACTLY matches the pattern in the `when` condition
- **Use the SOURCE (old/pre-migration) API paths** — the test simulates unmigrated code. Copy the `pattern` field from the rule YAML verbatim as your import/type/annotation. Never use the target (new) API path.
- Add a comment before each pattern: `// Rule: <ruleID>`
- Keep code minimal — one example per rule, just enough to trigger the pattern
- All imports/dependencies must be valid and resolve

**Dependency version requirements for `java.dependency` rules:**

For every `java.dependency` rule, query Maven Central before writing any version into pom.xml. Never rely on training-data guesses for version strings.

**Step A — Parse groupId and artifactId from the rule's `when.java.dependency.name` field:**
The name uses dot notation — the artifactId is the last hyphen-containing segment (e.g. `org.spockframework.spock-spring` → `g:org.spockframework`, `a:spock-spring`).

**Step B — Query Maven Central:**
```bash
curl -s "https://search.maven.org/solrsearch/select?q=g:%22<groupId>%22+AND+a:%22<artifactId>%22&core=gav&rows=20&wt=json"
```
Parse `.response.docs[].v` for the list of published versions.

**Step C — Select a version:**
1. Filter to versions satisfying the rule's bounds (below `upperbound`, at or above `lowerbound`)
2. From those, keep only **plain semver** versions (match `^\d+\.\d+\.\d+$`)
3. Use the most recent qualifying plain-semver version

**Step D — If no plain-semver version qualifies:**
Do NOT fabricate a version (e.g. do not write `2.3.0` for a Spock artifact that only publishes `2.3-groovy-4.0`). Do NOT switch to `builtin.xml`. Record the rule ID in `suspected_kantra_limitations` with reason `"no_plain_semver_version_on_maven_central"` and omit the `<version>` tag (BOM-managed fallback) if the BOM manages it, or omit the dependency entirely and note it in a comment.

**Step E — If Maven Central returns no results:**
Record in `suspected_kantra_limitations` with reason `"artifact_not_found_on_maven_central"`.

**When an artifact was discontinued before the target version** (e.g., `hibernate-proxool` was dropped in Hibernate 6): query MC to find the last published version under the rule's groupId. Use that version if it is plain semver.

**How the analyzer matches each condition type:** See `references/languages/<language>/test-data-guide.md` for the full matching rules per condition type. Getting this wrong is the #1 cause of test failures.

### 4. Resolve dependencies (only when needed)

See `references/languages/<language>/test-data-guide.md` for per-language dependency resolution rules. The key constraints:
- **Java:** Do NOT run `mvn compile` or any Maven command — kantra resolves dependencies by parsing pom.xml directly
- **Go:** Always run `go mod tidy` (do NOT run `go mod vendor` — vendored deps cause false positives in `go.referenced` analysis)

### 5. Sanitize XML

Do NOT run sanitize. The orchestrator handles this after all test-gen agents complete.

### 6. Return

Return the path to the tests directory to the orchestrator.

## Fix Iterations

On fix iterations, the orchestrator provides:
- Failing rule IDs
- Their patterns (from the rule YAML)
- Failure context and fix guidance from the rule-validator

When fixing:
- Regenerate ONLY the failing test groups — do not touch passing groups
- Use the fix guidance to understand what the test code needs
- The most common failure is: the test code doesn't actually use the API that the rule pattern matches
- If a specific code hint is provided (a single-line snippet), inject that exact line into the source file

### Compilation fix approach

If the test code has compilation errors:
1. Run the language-specific compiler to check
2. Fix ONLY the lines mentioned in the errors
3. Keep ALL rule-triggering code — every import and usage must remain
4. Do NOT change library versions in the build file — fix the code to match the installed version
5. For Go: run `go doc <package>` to get actual function signatures
6. Re-resolve dependencies after fixing
