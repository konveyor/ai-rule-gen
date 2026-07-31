---
name: eval
description: One-stop eval for generated Konveyor migration rules. Runs the deterministic eval (quality scores, app coverage, overlap detection) and the LLM judge (per-rule accuracy, coherence, gap analysis) in a single invocation. Optionally fixes issues it finds.
---

# Eval — Migration Rule Evaluator

You review each generated Konveyor migration rule individually against the source migration guide. For every rule you produce a verdict on whether its detection condition and migration message are correct and complete. You also identify guide patterns that no rule covers, and suggest concrete rule specifications for each gap.

## Why this exists

The deterministic eval (`cmd/eval`) answers "did the rules work?" — quality scores, app coverage, cross-reference analysis. It cannot answer "are the rules right?" — whether each rule detects the correct API and tells the developer the correct replacement. That requires reading the guide, reading the rule, and comparing them. That is your job. You run both: the deterministic eval for hard data, then the deep review for correctness.

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `guide_source` | yes | URL or file path of the migration guide |
| `rules_dir` | yes | Path to the generated rule YAML files |
| `app_dir` | no | Path to a sample application for coverage analysis (passed to `cmd/eval --app-dir`) |
| `migration` | no | Migration name for snapshot storage (e.g., `httpclient4-to-httpclient5`). Auto-inferred from `rules_dir` path if omitted. |

## Output

A human-readable markdown report. Full format is defined in Step 7 below.

## Workflow

### Step 0: Detect language and load reference

Scan the rule YAML files in `rules_dir` to determine the target language. Look at the condition provider prefix in `when` blocks:

| Provider prefix | Language |
|----------------|----------|
| `java.referenced`, `java.dependency` | java |
| `go.referenced`, `go.dependency` | go |
| `nodejs.referenced` | nodejs |
| `csharp.referenced` | csharp |
| `python.referenced` | python |
| `builtin.filecontent`, `builtin.file`, `builtin.xml`, `builtin.json` | (language-agnostic — check other rules or guide content) |

Create a run-scoped output directory for this eval session:

```bash
mkdir -p /tmp/eval-$(date +%Y%m%d-%H%M%S)/
```

Store the path — you will write `findings.json` here in Step 7.

Load the language-specific reference: `references/languages/<language>.md`

This reference contains:
- Language-specific migration map examples and action types
- Condition accuracy checks (location types, pattern qualification, breadth concerns)
- Calibration examples showing expected review depth

If no language reference exists for the detected language, proceed with the generic guidance in this file. Flag in the output summary that no language-specific reference was available.

### Step 1: Load the guide and build a migration map

If `guide_source` is a URL, fetch it as markdown:

```bash
go run ./cmd/ingest --input <url> --output /tmp/eval-guide.md
```

If the guide has multiple sub-pages, ingest each one separately into numbered files (`/tmp/eval-guide-1.md`, `/tmp/eval-guide-2.md`, etc.) so you capture the full guide.

Read the full guide content. Extract every **actionable migration pattern** into a structured migration map. Each entry should capture:

```yaml
old_api: <fully qualified name or identifier of the old API>
new_api: <replacement API or approach>
guide_section: <section title where this was found>
action_type: <category of change — see language reference for language-specific types>
severity: high | medium | low
code_before: <old usage pattern>
code_after: <new usage pattern>
```

Severity levels:
- **high** — compile/build error if not migrated (removed API, renamed symbol, changed signature)
- **medium** — behavioral change or deprecated API that still compiles but breaks at runtime
- **low** — optional improvement, convenience API, or edge case

Skip informational content that doesn't require code changes (background context, version history, performance notes).

Refer to the loaded language reference for language-specific migration map examples and action types.

#### Auto-generate ground truth

After building the migration map, check whether `evals/<migration>/ground_truth.yaml` exists. If it does not, generate it from the migration map you just extracted:

1. First, try the CLI regex extractor against the ingested guide files:

```bash
go run ./cmd/ground-truth \
  --from-guide /tmp/eval-guide.md \
  --guide-url <guide_url> \
  --output evals/<migration>/ground_truth.yaml
```

2. The regex extractor only catches full FQNs that appear literally in the guide text. Most guides use short class names, so the yield is low. Supplement by appending your LLM-extracted migration map entries: for each entry in your migration map that has a fully qualified `old_api`, add it to the ground truth file if not already present:

```yaml
entries:
  - old_api: <fully qualified old API from migration map>
    new_api: <replacement API>
    action_type: <from migration map>
    severity: <from migration map>
    guide_section: <section where found>
    reviewed_by: eval-skill-extract
    reviewed_date: "<today's date>"
```

Write the merged result to `evals/<migration>/ground_truth.yaml`. The deterministic eval auto-discovers this file for guide specificity gap analysis.

This generated ground truth is less comprehensive than japicmp-derived ground truth (which enumerates every changed API between two JAR versions), but it covers what the guide documents — which is exactly what matters for evaluating whether the rules match the guide.

### Step 2: Load the rules

Read every YAML file in `rules_dir`. For each rule, extract: `ruleID`, `description`, `message`, `when` condition (type, pattern, location), `category`, `effort`, `links`.

### Step 3: Run deterministic eval

Run the deterministic eval to get quality metrics, (optionally) app coverage, and save the snapshot:

```bash
# Without app coverage
go run ./cmd/eval --rules-dir <rules_dir> --save --migration <migration> 2>/dev/null > /tmp/eval-det.json

# With app coverage (if app_dir is provided)
go run ./cmd/eval --rules-dir <rules_dir> --app-dir <app_dir> --save --migration <migration> 2>/dev/null > /tmp/eval-det.json

# With explicit ground truth (auto-inferred from evals/<migration>/ground_truth.yaml if it exists)
go run ./cmd/eval --rules-dir <rules_dir> --ground-truth <path_to_ground_truth.yaml> --save --migration <migration> 2>/dev/null > /tmp/eval-det.json
```

The `--save` flag persists the deterministic snapshot to `evals/<migration>/runs/<timestamp>.json` for regression tracking. If `migration` was not provided as input, omit `--migration` — the CLI infers it from the `rules_dir` path.

Read the JSON output. Note:
- **Quality metrics** — which rules are missing messages, links, effort ratings, or before/after guidance
- **App coverage** (if `app_dir` was provided) — which rules fired (confirmed working) and which didn't, how many incidents each rule generated

This adds context but doesn't change verdicts — a rule can fire and produce incidents but still have an inaccurate message, or not fire because the app doesn't use that API.

Include the deterministic eval summary in the report output (see Step 7).

### Step 4: Per-rule review

For each rule, find the corresponding entry in your migration map. Then evaluate two dimensions:

#### Condition accuracy

Check the `when` condition against the guide:

- Does the pattern match the correct old API? Compare the identifier/name against what the guide says should be detected.
- Is the condition type appropriate? The language reference defines which condition types and location values are valid for this language — verify the rule uses the right ones.
- Could the pattern match unrelated code? An unqualified name without scoping could match unrelated APIs. Flag this as `warn`.
- Is the pattern too narrow or too broad? A broad package/module-level pattern is appropriate for package-level migration; a generic method name without scoping is too broad.

Refer to the loaded language reference for language-specific condition checks (location type appropriateness, pattern qualification, breadth concerns).

Verdicts:
- **pass** — pattern targets the API described in the guide and the condition type is appropriate
- **warn** — pattern is mostly right but could be more precise (missing qualification, slightly wrong condition type, could match unrelated code in edge cases)
- **fail** — pattern targets the wrong API, uses wrong condition type, or will clearly match incorrect code

#### Message accuracy

Check the `message` against the guide:

- Does the message describe the correct replacement? Compare what the rule tells the developer to do against what the guide says.
- Does the message include the new API name/class/method/function? A message saying "this is deprecated" without naming the replacement is a `warn`.
- Is the replacement code correct? If the rule says "replace X with Y" but the guide says "replace X with Z", that's a `fail`.
- Does the message include concrete code when the guide provides it? If the guide shows a before/after code snippet and the rule message is vague prose, that's a `warn`.

Verdicts:
- **pass** — message accurately describes the migration action per the guide, includes concrete replacement
- **warn** — message is correct but vague or incomplete (missing replacement name, no code example when guide provides one, missing important caveats)
- **fail** — message recommends wrong replacement, contradicts the guide, or is misleading

#### Suggestions

For every `warn` or `fail`, provide concrete actionable suggestions. Not "improve the message" — specific text:

- "Qualify pattern with full identifier: `<old_api_full_name>`"
- "Add replacement code to message: `<new_api_usage>`"
- "Change effort from 5 to 1 — this is a simple rename"
- "Change condition type from X to Y — the pattern is a method name, not a class"

#### Issue category

Classify each finding as one of two categories:

- **precision** — the rule detects the right thing but its pattern is broader than necessary. Typical cause: unqualified method name that could match unrelated classes. The fix is mechanical (add a qualifier, scope with `as`/`from`), or the broad pattern is an accepted tradeoff due to analyzer limitations.
- **coherence** — the condition scope and message scope disagree. The rule fires on a broad set of code (e.g., any import of a class) but the message only describes one specific use case, or vice versa. The fix requires rethinking the rule's design — either narrow the condition to match what the message says, or broaden the message to cover everything the condition matches.

Coherence issues are harder to fix and more likely to confuse developers who see the rule fire. Precision issues are lower risk — the rule fires too often but the message is still correct when it does fire.

Do not produce entries for rules that pass both dimensions — only report what needs attention.

### Step 4.5: Cross-rule coherence check

After reviewing individual rules, read all rule messages together. Check for cross-rule issues:

1. **Contradictory advice** — Do any two rules give conflicting replacement guidance? For example, rule A says "replace X with Y" while rule B says "replace X with Z."
2. **Per-file coherence** — If a developer sees all incidents in one file, does the combined guidance make a coherent migration path? Or do rules for the same class give contradictory or overlapping instructions?
3. **Implicit ordering dependencies** — Are there rules where the order matters? For example, "rename package imports before changing method calls" — if so, note the dependency.

Report cross-rule issues in the `cross_rule_issues` array of findings.json (see Step 7).

For each finding:
- **rule_ids**: the conflicting rule IDs
- **issue**: one-line description
- **severity**: `warn` for minor inconsistencies, `fail` for contradictory guidance
- **fix_type**: `deduplicate` (redundant rules), `reorder` (ordering dependency), `reconcile_messages` (conflicting advice)

If no cross-rule issues are found, report nothing — passing is silent.

### Step 5: Gap analysis

#### Ground truth (if available)

Check for `evals/<migration>/ground_truth.yaml`. If it exists, use it as the authoritative source for gap analysis instead of your LLM-extracted migration map. The ground truth is a human-curated list of every actionable migration pattern.

If `guide_version` in the ground truth is older than 90 days, include a warning in the report:
> "Ground truth last reviewed <date> — consider re-validating against current guide."

When using ground truth, also report extraction metrics at the end of the gap analysis section:
- **Extraction recall**: X/Y ground truth entries found by your migration map extraction
- **Extraction precision**: X/Y of your extracted entries verified in ground truth

These metrics measure the judge's own reliability and help calibrate future runs.

If no ground truth exists, use your LLM-extracted migration map as before.

#### Finding gaps

For each entry in your migration map (or ground truth), check whether any rule's `when` condition would **detect** that specific pattern. A gap exists when:

- No rule's condition pattern matches the old API identifier (even partially)
- A rule's condition is too broad to count as coverage — e.g., a PACKAGE-level rule covers class renames within that package, but an IMPORT-level rule for class X does NOT cover method-level changes on class X unless the message explicitly addresses them

A rule "covers" a guide pattern only if:
1. The condition would fire on code using that old API, AND
2. The message tells the developer what to do about that specific pattern

If a rule's condition fires but the message doesn't mention the specific migration action, that's a coherence finding (Step 4), not gap coverage. Report it in Findings, and also report the gap.

Be specific in gap entries — name the exact API and what the rule should detect/say.

#### Specificity gaps (from deterministic eval)

The deterministic eval reports `specificity_gaps` in the JSON output. These are imports in the sample app that are only covered by a broad PACKAGE-level rule (e.g., `org.apache.http` at PACKAGE) but have no dedicated IMPORT/TYPE/METHOD_CALL-level rule with class-specific guidance.

When `--ground-truth` is provided (or auto-inferred from `evals/<migration>/ground_truth.yaml`), the eval also reports `guide_specificity_gaps` — old APIs from the ground truth that lack dedicated rules. This works without a sample app, making it useful for migrations where no example application is available.

Include both types of specificity gaps in the gaps section of findings.json. Each gap should recommend a specific rule with an IMPORT-level condition for the missing class. These are high-value gaps — the developer gets only generic guidance today.

### Step 6: Structured output (findings.json)

Write a machine-readable `findings.json` to the run-scoped directory created in Step 0.

**Schema:**

```json
{
  "schema_version": 1,
  "language": "<detected language>",
  "timestamp": "<ISO 8601>",
  "precision_issues": [
    {
      "rule_id": "string, required",
      "pattern": "string, required — the condition pattern",
      "issue": "string, required — one-line description",
      "severity": "warn | fail",
      "fix_type": "add_scoping | qualify_fqn | narrow_pattern",
      "fix": {
        "scope_package": "string, optional",
        "qualified_pattern": "string, optional"
      }
    }
  ],
  "coherence_issues": [
    {
      "rule_id": "string, required",
      "issue": "string, required",
      "severity": "warn | fail",
      "fix_type": "narrow_condition | broaden_message | split_rule",
      "condition_scope": "broad | narrow | wrong",
      "message_scope": "broad | narrow | wrong"
    }
  ],
  "cross_rule_issues": [
    {
      "rule_ids": ["string array, required"],
      "issue": "string, required",
      "severity": "warn | fail",
      "fix_type": "deduplicate | reorder | reconcile_messages"
    }
  ],
  "gaps": [
    {
      "source_fqn": "string, required — old API identifier",
      "location": "string, optional",
      "action_type": "string, required",
      "target_api": "string, required — replacement API",
      "message_sketch": "string, required — draft message",
      "severity": "high | medium | low",
      "guide_section": "string, required"
    }
  ]
}
```

**`action_type` enum:** `class_rename`, `method_rename`, `method_removal`, `package_change`, `config_change`, `signature_change`, `behavioral_change`. Unknown values are treated as `behavioral_change` — log a warning in the markdown report under a "Schema warnings" section (only if warnings exist): `"unknown action_type '<value>' in gap for <source_fqn>, treating as behavioral_change"`.

Every field marked "required" must be present. Empty arrays are valid — omit the array entirely if there are no findings for that category.

### Step 7: Report

Produce a human-readable markdown report. The report has four sections: deterministic eval summary, judge summary, findings, and gaps. Only warn/fail rules and gaps appear — passing rules are silent.

#### Format

```markdown
## Eval Judge Report

**Guide:** <guide title or URL>
**Language:** <detected language>

### How the rules performed

| Metric | Value |
|--------|-------|
| Total rules | <total> |
| Quality score | avg <score>/<max> |
| App coverage | <fired>/<total> fired (<percent>%) — <incidents> incidents (omit if no app_dir) |

<if there are unmatched rules>
**Unmatched rules** — these rules should have fired against the sample app but didn't (likely kantra limitations):
- `<rule-id>` (<brief description>)
</if>

<if there are missing quality items>
**Quality gaps:** <N> rules missing before/after guidance
</if>

### Rules that need attention

**<pass_count> of <total> rules passed.** The <warn_count + fail_count> below need fixes:

> **What "detection" and "guidance" mean:**
> - **Detection** = the rule's `when` condition — does it find the right code?
> - **Guidance** = the rule's `message` — does it tell the developer the correct fix?
>
> **Issue types:**
> - **Precision** — detection is too broad (e.g., matches unrelated code). The guidance is still correct when the rule fires. Fix is usually mechanical.
> - **Coherence** — detection and guidance don't match. The rule fires on one thing but advises about something else. Needs a design rethink.

#### Precision issues

These rules detect the right thing but cast too wide a net. Guidance is correct when they fire.

| Rule | Detection | Guidance | What's wrong | How to fix |
|------|-----------|----------|--------------|------------|
| `<rule-id>` | too broad | ok | <one-line issue> | <concrete fix> |

#### Coherence issues

These rules have a mismatch between what they detect and what they advise. Developers may see confusing or irrelevant guidance.

| Rule | Detection | Guidance | What's wrong | How to fix |
|------|-----------|----------|--------------|------------|
| `<rule-id>` | ok | wrong scope | <one-line issue> | <concrete fix> |

### Missing rules

These migration patterns from the guide have no corresponding rule. A missing rule means affected code gets no warning at all.

| What the guide says to migrate | Guide section | Impact | Suggested detection |
|-------------------------------|---------------|--------|---------------------|
| <old API or behavior change> | <section> | high/medium/low | <pattern + location + message sketch> |
```

If there are no findings, write "All rules passed — no issues found." If there are no missing rules, write "No missing rules — all guide patterns are covered."

Use "ok" in the Detection/Guidance column when that dimension passes. Use a short phrase describing the problem when it doesn't (e.g., "too broad", "wrong scope", "wrong API", "vague").

Keep each row concise. If a rule has multiple suggestions, list them semicolon-separated in the "How to fix" cell.

### Step 8: Auto-fix (optional)

After producing the Step 7 report, ask the user: **"Should I fix these issues?"** If they confirm, proceed with the auto-fix loop below. If they decline, stop here.

#### Pre-fix snapshot

Before making any changes, capture the current state:

```bash
go run ./cmd/eval --rules-dir <rules_dir> [--app-dir <app_dir>] 2>/dev/null > /tmp/eval-<timestamp>/pre-fix.json
```

This is the pre-fix snapshot that all comparisons measure against.

#### Mirror and fix

1. **Mirror the full ruleset** into the run-scoped directory:
   ```bash
   cp -r <rules_dir> /tmp/eval-<timestamp>/proposed-rules/
   ```
   This is a full copy — all rules, not just modified files. This ensures `cmd/eval` can run on the complete set.

2. **Fix precision issues**: For each precision finding in findings.json, invoke the rule-writer to apply the fix (add scoping, qualify FQN, narrow pattern). Write changes to the mirrored copy.

3. **Generate gap rules**: For each gap entry with severity `high` or `medium`, feed the gap as a pattern to the rule-writer. Write new rule files to the mirrored copy.

4. **Validate**:
   ```bash
   go run ./cmd/validate --rules-dir /tmp/eval-<timestamp>/proposed-rules/
   ```

5. **Re-run deterministic eval**:
   ```bash
   go run ./cmd/eval --rules-dir /tmp/eval-<timestamp>/proposed-rules/ [--app-dir <app_dir>] 2>/dev/null > /tmp/eval-<timestamp>/post-fix.json
   ```

6. **Re-run judge** (Steps 4 and 4.5 only) on the modified rules to produce an updated findings.json. Compare against prior iteration findings.

#### Stop conditions — do NOT continue if:

- Any deterministic metric (coverage, quality_avg) drops below the pre-fix snapshot
- `cmd/validate` fails
- Judge coherence issues regress: compare by fingerprint (`rule_id + fix_type`). Regression = any new `fail`-severity issue not in prior iteration, OR total `fail` count increases
- **Warn budget exceeded**: total `warn`-severity issue count grows by more than 50% vs pre-fix state (minimum threshold: 3 new warns). Example: pre-fix has 4 warns → budget allows up to 6; pre-fix has 0 warns → budget allows up to 3

If none of the stop conditions are hit AND issues remain, run **one more fix iteration** (max 2 total).

#### Apply

Present the diff (proposed-rules vs original) to the user. If approved:

1. `mv <rules_dir> <rules_dir>.bak`
2. `mv /tmp/eval-<timestamp>/proposed-rules/ <rules_dir>`
3. Confirm with user, then `rm -rf <rules_dir>.bak`

If step 2 fails (cross-filesystem), fall back to: `cp -r proposed-rules <rules_dir>.new && mv <rules_dir> <rules_dir>.bak && mv <rules_dir>.new <rules_dir>`.

**Rollback**: `rm -rf <rules_dir> && mv <rules_dir>.bak <rules_dir>`

#### Scope restriction

The auto-fix loop only modifies files under the rules output directory. It never touches `SKILL.md`, test files, or Go source code.

## Calibration

The language reference file (`references/languages/<language>.md`) contains calibration examples showing what constitutes a pass, warn, fail, and gap for that language. Read and internalize those examples before starting reviews.

The key qualities of good findings:
- **Specificity** — cite the exact API, pattern, and guide section, not vague summaries
- **Actionability** — every suggestion is concrete text the rule author can copy-paste or directly apply
- **Proportionality** — a minor qualification issue is `warn`, not `fail`; a wrong replacement is `fail`, not `warn`
- **Signal over noise** — only surface what needs attention; passing rules stay silent

## Permissions

| Operation | Pattern | Purpose |
|-----------|---------|---------|
| shell | `go run ./cmd/eval *` | Run deterministic eval for quality/coverage metrics |
| shell | `go run ./cmd/ingest *` | Fetch migration guide as markdown |
| shell | `go run ./cmd/ground-truth *` | Generate ground truth from guide or japicmp |
| shell | `mkdir -p /tmp/eval-*` | Create run-scoped output directory |
| read | `output/**` | Read rules, eval output |
| read | `/tmp/**` | Read fetched guide content, eval results |
| read | `references/**` | Read language-specific reference |
| read | `evals/**` | Read ground truth, eval configs |
| write | `evals/**` | Write auto-generated ground truth |
| write | `/tmp/eval-*/**` | Write findings.json and intermediate files |
| shell | `go run ./cmd/validate *` | Validate proposed rules (auto-fix) |
| shell | `cp -r *` | Mirror ruleset for auto-fix |
| shell | `mv *` | Apply proposed rules (auto-fix, user-approved) |
| shell | `rm -rf *.bak` | Remove backup after confirmed apply |
