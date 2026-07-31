---
name: generate-rules
description: Generates Konveyor analyzer migration rules from a migration guide (URL, file path, or pasted text). Use when the user wants to create Konveyor rules, migrate an application, or convert a migration guide into analyzer rules.
---

# Generate Konveyor Migration Rules

Generate Konveyor analyzer migration rules from a migration guide.

**Input:** $ARGUMENTS (URL, file path, or pasted text of a migration guide)

If no argument is provided, ask the user for the migration guide source.

**IMPORTANT — Fresh run every time:** Each invocation creates a new timestamped output directory. Do NOT scan `output/` for previous runs, do NOT reuse existing directories, do NOT check for prior results. Start step 1 immediately. The only exception is when the user explicitly provides `migration_dir` with `resume_from`.

## Inputs

- `guide_source` — URL, file path, or pasted text of a migration guide
- `sources` — (optional) Source technology labels as a list, e.g. `["spring-boot3", "spring-boot"]`. Each becomes a `konveyor.io/source=` label. First element is the primary (used for naming). Auto-detected if omitted.
- `targets` — (optional) Target technology labels as a list, e.g. `["spring-boot4", "spring-boot"]`. Each becomes a `konveyor.io/target=` label. First element is the primary (used for naming). Auto-detected if omitted.
- `mode` — (optional) `interactive` (default) or `non_interactive`
- `checkpoint_behavior` — (optional) `ask` (default), `continue`, or `stop_after_extract`
- `resume_from` — (optional) `ingest`, `extract`, `coverage`, `scaffold`, `test`, `report`
- `migration_dir` — (optional) explicit output directory override. If omitted, auto-generated as `output/<primary_source>-to-<primary_target>-<YYYYMMDD-HHMMSS>`. Required when using `resume_from`.

### Command Presets

| Command | checkpoint_behavior | What it does |
|---------|-------------------|--------------|
| `/generate-rules` | `stop_after_extract` | Ingest, extract, coverage, report, summary (no testing) |
| `/generate-rules-with-test` | `continue` | Full pipeline including test generation and validation |

Both commands invoke this skill. The user can override `checkpoint_behavior` explicitly.

## Returns

- `rules_dir` — Path to generated rule YAML files
- `tests_dir` — Path to test data
- `report` — Path to report.yaml
- `summary` — Markdown summary table with:
  - rules_count, passed, failed, pass_rate
  - coverage_report (sections processed/skipped)
  - fix_iterations used

## Execution Modes

- `interactive` mode: ask at checkpoint only when `checkpoint_behavior=ask`
- `non_interactive` mode: never prompt the user; obey `checkpoint_behavior`
  - `continue`: run full pipeline
  - `stop_after_extract`: end after extraction/coverage with untested outputs
  - `ask`: treated as `continue` (no prompt allowed in non-interactive mode)

## References

- [`references/orchestrator-details.md`](references/orchestrator-details.md) — status-line conventions, parallelism defaults, resume preconditions, rebuild behavior

## Permissions

| Operation | Pattern | Purpose |
|-----------|---------|---------|
| shell | `mkdir -p output` | Create output directory |
| shell | `go run ./cmd/ingest *` | Fetch migration guide as markdown |
| shell | `go run ./cmd/sections *` | Index guide sections with classification |
| shell | `go run ./cmd/merge-patterns *` | Merge partial patterns files |
| shell | `go run ./cmd/contract-validate *` | Validate sub-agent payload contracts |
| shell | `go run ./cmd/verify *` | Verify patterns against source artifacts |
| shell | `go run ./cmd/construct *` | Build rule YAML from patterns.json |
| shell | `go run ./cmd/validate *` | Validate rule YAML structure |
| shell | `go run ./cmd/scaffold *` | Create test directories and manifests |
| shell | `go run ./cmd/sanitize *` | Fix XML comments in test data |
| shell | `go run ./cmd/test *` | Run kantra tests |
| shell | `go run ./cmd/report *` | Generate summary report with per-rule status |
| shell | `go run ./cmd/coverage *` | Check guide coverage |
| shell | `go run ./cmd/log *` | Append orchestrator events to pipeline log |
| shell | `ls *` | List files and directories |
| read | `output/**` | Read manifest, test results |
| write | `output/**` | Write migration artifacts |

**Do NOT use `python`, `python3`, `node`, or any scripting language runtime.**
**Do NOT use `grep`, `sed`, `awk`, `wc`, `find`, or other shell text-processing tools to parse data that CLI commands already return.**
Every `go run ./cmd/*` command returns structured JSON. Use that JSON output directly — do not pipe it through shell commands to extract fields.
This is a Go project. Run only commands listed in this permissions table.
If a required action is not permitted, stop and report the limitation — do NOT retry the same operation with different syntax (heredocs, base64 encoding, helper scripts, etc.). A permission denial means the action is blocked, not that the syntax was wrong. Move to the next pipeline step or end the pipeline.

## UX Principles

- Emit short status lines continuously (start + finish for each step).
- Only ask at checkpoint when `mode=interactive` and `checkpoint_behavior=ask`.
- Do not block after testing starts; continue even if some rules remain failing.
- Fix only failing rules, not the full suite.

For pipeline logging conventions, see [`references/orchestrator-details.md`](references/orchestrator-details.md) (Pipeline Transcript section).

For stage timing instructions, see [`references/orchestrator-details.md`](references/orchestrator-details.md) (Timing section).

For sub-agent invocation protocol and contract validation, see [`references/orchestrator-details.md`](references/orchestrator-details.md) (Sub-agent Protocol section).

## Pipeline

**Error handling:** If any CLI command (`ingest`, `construct`, `validate`, `scaffold`, `sanitize`, `test`, `report`) exits non-zero, print `[step] FAILED — <error>` and stop the pipeline. Do not proceed to the next step.

**Partial failures:** If a step produces partial results before failing (e.g., `construct` succeeds for 48 patterns but fails on 2), preserve the partial output and report what succeeded. The user can fix the failing patterns and re-run. Do not discard valid results because of a partial failure.

**Resume behavior:** When `resume_from` is set, the user must also provide `migration_dir` pointing to the specific timestamped directory to resume. Skip prior stages and validate prerequisites first. Fail fast when required artifacts are missing.

Stage prerequisites (paths relative to `<migration_dir>`):
- `extract` requires `guide.md`
- `coverage` requires `patterns.json`
- `scaffold` requires `rules/`
- `test` requires `rules/` and `tests/manifest.json`
- `report` requires source/target metadata plus pass/fail totals and rule ID lists

### 1. Ingest

Generate a timestamp once at pipeline start (`YYYYMMDD-HHMMSS` in local time).

Determine the migration directory path:
- If the user provided `migration_dir`, use that (no timestamp)
- If the user provided `sources` and `targets` (or `source` and `target`), use `output/<primary_source>-to-<primary_target>-<YYYYMMDD-HHMMSS>` where `primary_source` is `sources[0]` and `primary_target` is `targets[0]`
- If auto-detecting: ingest to `output/guide-temp-<YYYYMMDD-HHMMSS>.md` first (use the pipeline timestamp to avoid collisions when multiple invocations run in parallel), read the first ~50 lines to detect source/target/language (lowercase, hyphenated names like `spring-boot-3`), then use `output/<source>-to-<target>-<YYYYMMDD-HHMMSS>`

```bash
mkdir -p output/<source>-to-<target>-<YYYYMMDD-HHMMSS>
```

Every `go run` command should be a standalone invocation with fully-expanded arguments — substitute actual paths and flag values directly into each command. Use one complete, self-contained command per invocation; do not use shell variable assignments (`VAR=...`) or compound `&&` chains, since these don't match the permissions table.

The log flags are: `--log <migration_dir>/pipeline.log --agent orchestrator --model <your_model_id>`. Append these literally to every CLI invocation.

```
[ingest] Fetching guide from <source>...
```

- **URL:** `go run ./cmd/ingest --log <migration_dir>/pipeline.log --agent orchestrator --model <your_model_id> --input <url> --output <migration_dir>/guide.md`
- **File (not markdown):** `go run ./cmd/ingest --log <migration_dir>/pipeline.log --agent orchestrator --model <your_model_id> --input <path> --output <migration_dir>/guide.md`
- **Pasted text or already markdown:** Write directly to `<migration_dir>/guide.md`

If auto-detecting, write `output/guide-temp-<YYYYMMDD-HHMMSS>.md` to `<migration_dir>/guide.md` using the Write tool after the migration directory path is determined.

**Stub detection — follow sub-pages.** After ingesting each URL, check the output for stub characteristics. A page is a stub if it has fewer than 100 non-blank lines of content after `cmd/ingest` processing (`cmd/ingest` already strips `<nav>`, `<footer>`, and `<aside>` elements during HTML-to-markdown conversion, so no additional filtering is needed).

If a stub is detected:

1. Read the ingested markdown and extract all links whose resolved URL shares the same base path as the original URL (same-origin sibling pages). Sibling links are those pointing to `*.html` files under the same directory path as the original URL. Exclude external links, anchor-only links (`#...`), and links to parent directories.
2. For each sibling link, resolve it to a full URL relative to the original URL's base path.
3. Ingest each sub-page with `cmd/ingest` into a separate file (`guide-part-<N>.md`).
4. Concatenate all parts (original + sub-pages) into the final `guide.md`, in the order the links appear in the stub page (document order).
5. If a sub-page fetch fails (404, timeout), log a warning (`[ingest] WARNING — failed to fetch <url>: <error>`) and skip that page. Do not abort the pipeline — partial content is better than none.

This ensures that documentation sites which split content across multiple pages are fully captured. Without this step, the pipeline will only see the stub and miss the bulk of the migration content.

Count lines (`GUIDE_LINES`) and section headings. Print:

```
[ingest] Done — <GUIDE_LINES> lines, <N> sections → <migration_dir>/ (<elapsed>)
```

### 2. Extract (parallel)

Extraction uses parallel agents to process the guide faster. The orchestrator splits the guide into chunks and spawns multiple rule-writer agents.

**2a. Index sections and auto-detect metadata:**

```bash
go run ./cmd/sections <log_flags> --guide <migration_dir>/guide.md
```

This returns JSON with all sections classified as `content` or `header-only`. Filter to content sections only.

```
[extract] Extracting patterns from <content_count> sections (<N> parallel agents)...
```

**2b. Split into chunks and dispatch:**

Split the content sections into **N balanced chunks** (minimum 1, maximum 5 agents). Balance by section count. Assign each chunk a number (1, 2, ..., N).

**Keep related sections together.** Group sections by their top-level heading (`##`) — all subsections (`###`, `####`) under the same `##` must stay in the same chunk. Migration guides pair before/after code examples across subsections within a topic; splitting them across agents means no agent sees both sides of the diff. Balance chunks by top-level-heading groups, not by individual sections.

**Invoke:** `rule-writer` (one per chunk, in parallel)
**Purpose:** Extract migration patterns from assigned sections.
**Inputs per invocation:**
  - guide: <migration_dir>/guide.md
  - sources: {sources array (e.g., ["spring-boot3", "spring-boot"])}
  - targets: {targets array (e.g., ["spring-boot4", "spring-boot"])}
  - rules_dir: <migration_dir>/rules
  - sections: {chunk subset — list of `{heading, start_line, end_line}` objects}
  - all_headings: {full list of ALL content section headings from step 2a — not just this chunk's sections. Enables each agent to detect multi-path migrations even when variant sections are in other chunks.}
  - output_file: <migration_dir>/patterns-{chunk_number}.json
**Parallel:** yes
**Expect:**
  - patterns_count, output_file

**2c. Merge, verify, and construct:**

After all agents complete, merge the partial patterns files, verify source artifacts, and build rules:

```bash
go run ./cmd/merge-patterns <log_flags> --output <migration_dir>/patterns.json <migration_dir>/patterns-1.json <migration_dir>/patterns-2.json ...
go run ./cmd/verify <log_flags> --patterns <migration_dir>/patterns.json --output <migration_dir>/verify-results.json --cache <migration_dir>/verify-cache
go run ./cmd/construct <log_flags> --patterns <migration_dir>/patterns.json --output <migration_dir>/rules
go run ./cmd/validate <log_flags> --rules <migration_dir>/rules
```

**Verification step:** `cmd/verify` downloads source JARs from Maven Central and checks that each pattern's `source_fqn` exists in the declared `source_artifact`. This catches hallucinated class names before rules reach testing.

The verify stdout JSON contains `verified`, `not_found`, `skipped`, and `offline` counts. Print:

```
[verify] <verified> verified, <not_found> not found, <skipped> skipped
```

If `not_found > 0`, read `<migration_dir>/verify-results.json` and note which `pattern_index` values have `status: "not_found"`. After construct runs, use the `pattern_rule_map` from construct's stdout JSON to map each `pattern_index` → `rule_id`. Carry `verified_rule_ids` and `not_found_rule_ids` forward to step 5 (report).

If validation fails, fix `<migration_dir>/patterns.json` (remove invalid entries) and re-run construct.

```
[extract] Done — <patterns_count> patterns → <rules_count> rules in <migration_dir>/rules/ (<elapsed>)
```

Save `source`, `target`, `patterns_count`, `rules_count`, and verify counts for the final summary.

### 2d. Coverage Check

Run the coverage tool to find sections with named artifacts that weren't extracted:

```bash
go run ./cmd/coverage <log_flags> --guide <migration_dir>/guide.md --patterns <migration_dir>/patterns.json --language <language>
```

If `gap_count > 0` in the JSON output, print:

```
[coverage] <gap_count> sections with uncovered artifacts: <heading_1>, <heading_2>, ...
```

Convert gaps to sections and send back to the rule-writer in chunk mode for targeted re-extraction. Each gap has `heading` and `line` — use the sections index to get the `start_line` and `end_line` for that heading. Mention the uncovered artifacts in the prompt so the sub-agent focuses on them.

**Invoke:** `rule-writer` (chunk mode)
**Purpose:** Extract patterns from specific sections that the coverage check flagged.
**Inputs:**
  - guide: <migration_dir>/guide.md
  - sources: {sources array}
  - targets: {targets array}
  - rules_dir: <migration_dir>/rules
  - sections: {gap sections converted to `[{heading, start_line, end_line}]` format}
  - output_file: <migration_dir>/patterns-gaps.json
**Parallel:** no
**Expect:**
  - patterns_count, output_file

The sub-agent writes only the new patterns to `<migration_dir>/patterns-gaps.json`. It does NOT read or inspect existing patterns — the orchestrator handles deduplication. After the sub-agent returns, merge, verify, and rebuild:

```bash
go run ./cmd/merge-patterns <log_flags> --output <migration_dir>/patterns.json <migration_dir>/patterns.json <migration_dir>/patterns-gaps.json
go run ./cmd/verify <log_flags> --patterns <migration_dir>/patterns.json --output <migration_dir>/verify-results.json --cache <migration_dir>/verify-cache
go run ./cmd/construct <log_flags> --patterns <migration_dir>/patterns.json --output <migration_dir>/rules
go run ./cmd/validate <log_flags> --rules <migration_dir>/rules
```

Re-run the coverage check. If gaps remain, accept them — one re-extraction pass is enough.

```
[coverage] Done — <final_gap_count> remaining gaps (accepted) (<elapsed>)
```

If `gap_count == 0`:

```
[coverage] No gaps found (<elapsed>)
```

### Checkpoint

After extraction/coverage:

- If `mode=non_interactive`, do **not** prompt:
  - `checkpoint_behavior=continue` or `ask`: continue automatically
  - `checkpoint_behavior=stop_after_extract`: skip testing and go to Step 5 (Report)
- If `mode=interactive`:
  - `checkpoint_behavior=continue`: continue automatically
  - `checkpoint_behavior=stop_after_extract`: skip testing and go to Step 5 (Report)
  - `checkpoint_behavior=ask`: ask:
    - `Continue with test generation and validation? (y/n)`
    - if no, skip to Step 5 (Report) with untested rules

**Rules-only report:** When skipping to Step 5, invoke `cmd/report` with `--passed 0 --failed 0 --kantra-limitation 0` and empty `--passed-rules`, `--failed-rules`, `--kantra-limitation-rules` flags. Pass `--rules-total`, `--verified-rules`, and `--not-found-rules` from extraction. All rules get `test_status: untested`.

### 3. Test Generation

**3a. Scaffold (orchestrator runs this directly, not an agent):**

```bash
go run ./cmd/scaffold <log_flags> --rules <migration_dir>/rules --output <migration_dir>/tests --patterns <migration_dir>/patterns.json
```

This creates all directories, `.test.yaml` files, and `manifest.json`. No LLM needed.

**3b. Read manifest and split into batches:**

Read `<migration_dir>/tests/manifest.json`. It contains a `groups` array — each group has `name`, `data_dir`, `files` (paths to generate), and `rule_ids`.

Split the groups into **batches of ~5 groups each** (minimum 1 batch, maximum 5 batches), balanced by rule count.

**3c. Spawn test-generator agents:**

```
[test-gen] Generating test data for <rules_count> rules (<B> parallel agents)...
```

**Invoke:** `test-generator` (one per batch, up to 5 in parallel)
**Purpose:** Generate compilable test source code that triggers the assigned rules.
**Inputs per invocation:**
  - rules_dir: <migration_dir>/rules
  - tests_dir: <migration_dir>/tests
  - groups: {batch subset from manifest.json — include name, data_dir, rule_ids, files}
**Parallel:** yes
**Expect:**
  - groups_completed, files_written

**3d. Collect results and sanitize.** After all agents complete:

Collect `suspected_kantra_limitations` from all rule-writer and test-generator agent returns (merge, deduplicate by `rule_id`). If non-empty, print:

```
[test-gen] <N> suspected kantra limitations (Maven Central: no plain-semver version): <rule_id_1>, ...
```

Carry this merged list forward as `pre_classified_kantra_limitations` — it will be passed to the rule-validator to skip the fix loop for these rules.

Sanitize XML:

```bash
go run ./cmd/sanitize <log_flags> --dir <migration_dir>/tests/data
```

```
[test-gen] Done — <total_groups> groups, <total_files> files (<elapsed>)
```

### 4. Validate (orchestrator-driven loop)

The orchestrator runs tests and uses a sub-agent for LLM-driven repairs on failures.

**4a. Run tests:**

```bash
go run ./cmd/test <log_flags> --rules <migration_dir>/rules --tests <migration_dir>/tests --timeout 5m
```

The CLI runs each test file sequentially and automatically retries timed-out files once (`--retry-timeouts`, on by default). Tests run locally via `--run-local=true`. To run a subset, use `--files` with bare filenames (e.g., `--files data-1.test.yaml,data-2.test.yaml`), resolved relative to `--tests` dir.

Print the result:

```
[validate] <total_passed>/<total_rules> passed (<elapsed>)
```

Or if failures:

```
[validate] <total_passed>/<total_rules> passed — <F> failures: <rule_id_1>, <rule_id_2>, ...
```

If all passed (elapsed shown above), skip to step 5.

**4b. Fix (only if failures):**

```
[fix] Fixing <F> failures: <rule_id_1>, <rule_id_2> ...
```

**Invoke:** `rule-validator`
**Purpose:** Fix test data for failing rules so they pass kantra validation.
**Inputs:**
  - rules_dir: <migration_dir>/rules
  - tests_dir: <migration_dir>/tests
  - failing_rules: {list with rule_id, test_file, error for each failure}
  - pre_classified_kantra_limitations: {merged list from rule-writer and test-generator suspected_kantra_limitations — may be empty}
  - max_iterations: 1
**Parallel:** no
**Expect:**
  - fixed_rules, still_failing, kantra_limitation_rules, iterations_used, fix_details
  - results_by_rule (with `result_type` + `recommended_action`)

The validator agent owns the fix-verify loop — it fixes files, re-runs tests via `cmd/test`, and iterates up to `max_iterations`. Default is 1 iteration; pass up to 3 if the user requests more attempts.

**The orchestrator must NOT run its own fix attempts.** Do not manually edit test files, create stub classes, toggle analysisParams, or retry. All fix work goes through the rule-validator sub-agent. When the validator returns `still_failing`, accept that result and move on.

Print the result:

```
[fix] <fixed_count> fixed, <still_failing_count> still failing, <kantra_limitation_count> kantra limitations (<validate elapsed>)
```

The `<validate elapsed>` is the total time since the validate stage started (step 4a), including test runs and all fix iterations.

If `kantra_limitation_rules` is non-empty, print:

```
[fix] kantra limitations (engine cannot test — rule is correct): <rule_id_1>, ...
```

Group unresolved failures by `result_type` in `results_by_rule` and include grouped counts in the final summary (e.g., `still_failing_timeout: 2`, `still_failing_kantra_limitation: 1`).

If still_failing is non-empty, move on — don't block the pipeline.

### 5. Report

Collate pass/fail results from the test run and the fix loop. Generate `report.yaml` with per-rule test and verification status. No labels are written to rule YAML files — all pipeline metadata lives in the report.

```bash
go run ./cmd/report <log_flags> --source <comma-separated sources> --target <comma-separated targets> --output <migration_dir>/report.yaml --rules-total <N> --passed <P> --failed <F> --kantra-limitation <K> --passed-rules <comma-separated passed rule IDs> --failed-rules <comma-separated failed rule IDs> --kantra-limitation-rules <comma-separated kantra limitation rule IDs> --verified-rules <comma-separated verified rule IDs> --not-found-rules <comma-separated not-found rule IDs>
```

The `--verified-rules` and `--not-found-rules` flags come from the verification mapping built in step 2c (pattern_index → rule_id via construct's pattern_rule_map).

The report includes a `rules` section with per-rule `test_status` (passed/failed/kantra-limitation/untested) and `source_verified` (true/false). The pass rate is computed as `passed / (passed + failed)` — kantra limitations are excluded since they are not failures, but also not confirmed passes.

### 6. Summary

Print a formatted summary table using GitHub-flavored markdown:

```markdown
## Summary

| | |
|---|---|
| **Input** | <guide title or URL as a markdown link> |
| **Migration** | <primary_source> → <primary_target> (<language>) |
| **Guide** | <GUIDE_LINES> lines, <N> sections → <M> produced patterns, <K> skipped |
| **Rules** | <rules_count> generated, **<P>/<N> passed (<percent>%)** |
| **Verification** | <verified>/<total> source-verified, <not_found> not found, <skipped> skipped (omit row if all skipped) |
| **Kantra limitations** | <K> rules correct but not auto-testable — engine cannot compare non-semver versions (omit row if K=0) |
| **Fix iterations** | <iterations used, 0 if none> |
| **Timing** | total: <total elapsed> — ingest: <elapsed>, extract: <elapsed>, coverage: <elapsed>, test-gen: <elapsed>, validate: <elapsed>, report: <elapsed> |
| **Output** | `<migration_dir>/patterns.json` (patterns), `<migration_dir>/rules/` (rules), `<migration_dir>/tests/` (tests), `<migration_dir>/report.yaml` (report) |

### Rule Categories

Use the `groups` array from the `construct` JSON output — it lists each file and its rule count. Do not iterate rule files manually.

| Group | Rules | Status |
|---|---|---|
| <group_name> (<brief description of what rules cover>) | <rule_count> | <passed>/<total> passed |
| ... | ... | ... |
```

If there are failures, add a row to the top-level table:

```
| **Failed** | <rule_id_1>, <rule_id_2>, ... |
```

If coverage was low (< 30% of sections produced patterns), add a row:

```
| **Warning** | Low extraction coverage (<M>/<K> sections) |
```

**Rules-only summary (when `checkpoint_behavior=stop_after_extract`):**

When testing was skipped, use this simplified table instead:

```markdown
## Summary

| | |
|---|---|
| **Input** | <guide title or URL as a markdown link> |
| **Migration** | <primary_source> → <primary_target> (<language>) |
| **Guide** | <GUIDE_LINES> lines, <N> sections → <M> produced patterns, <K> skipped |
| **Rules** | <rules_count> generated (untested) |
| **Verification** | <verified>/<total> source-verified, <not_found> not found, <skipped> skipped (omit row if all skipped) |
| **Timing** | total: <total elapsed> — ingest: <elapsed>, extract: <elapsed>, coverage: <elapsed>, report: <elapsed> |
| **Output** | `<migration_dir>/patterns.json` (patterns), `<migration_dir>/rules/` (rules), `<migration_dir>/report.yaml` (report) |
| **Next step** | Run `/generate-rules-with-test resume_from=scaffold migration_dir=<migration_dir>` to add tests |
```

Omit these rows: **Kantra limitations**, **Fix iterations**, **Failed** (no testing ran).
Omit `tests/` from the **Output** row and test-gen/validate timing from the **Timing** row.
Include the **Rule Categories** table as normal.
