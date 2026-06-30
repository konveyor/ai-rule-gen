# agent-eval-harness: Evaluation Report for generate-rules-with-test

**Plugin:** [opendatahub-io/agent-eval-harness](https://github.com/opendatahub-io/agent-eval-harness) v1.9.1 (commit `7bc8c66`)
**Skill under test:** `generate-rules-with-test` (skill hash `798c67061a73`)
**Runner:** Claude Code 2.1.187, model `claude-opus-4-6`
**Dates:** 2026-06-24 (two runs: v1 and v2)

---

### Short Summary

The [agent-eval-harness](https://github.com/opendatahub-io/agent-eval-harness) is a good starting point for any skill that has no eval today. It gets real signal fast — point it at a skill definition and it designs judges, sources test cases, and runs an end-to-end eval without writing any eval code. For this project, it caught infrastructure issues (sandbox permissions, missing JAVA_HOME, turn budget exhaustion) that were not visible during manual testing.

However, the harness has a low ceiling for ongoing quality work. Its judges score at the case level ("4/5 for this migration guide"), not per output artifact — so it cannot identify which specific rule is wrong or why. It has no auto-fix capability and no way to validate output against real-world usage. The v1.9.1 (at the time of running this eval) also needed 4 local patches to work correctly for this project.

For anyone evaluating whether to adopt agent-eval-harness or build a custom eval:

**What the harness does well (any skill type):**

- Reads the skill definition and auto-designs judges — no eval expertise required
- Sources test cases autonomously (from existing output + web search)
- Catches pipeline/infrastructure failures (permissions, timeouts, env vars, turn exhaustion)
- Structural output checks — did the skill produce the expected files with the expected shape?
- LLM judges for subjective quality assessment with configurable prompts
- Repeatable — same config, re-run anytime
- The harness has `/eval-optimize` which can iteratively edit the skill's SKILL.md to improve judge scores, but it fixes the **skill definition**, not individual output artifacts 

**Where the harness falls short (any skill type):**

- Case-level scoring only — "4/5 on this run" does not identify which output artifact is wrong
- No domain-specific validation (e.g., running generated rules against a real application)
- Required 4 local patches for this project — other complex skills may hit similar gaps
- $50-70 per 5-case run makes rapid iteration expensive
- Judge quality depends on auto-generated prompt quality, which may vary by skill complexity


**When to use which:**


| Situation                                         | Recommendation                                                                 |
| ------------------------------------------------- | ------------------------------------------------------------------------------ |
| No eval exists, need signal fast                  | Agent-eval-harness — operational in under an hour                              |
| Need pipeline reliability testing                 | Agent-eval-harness — this is its primary strength                              |
| Need to iterate on output quality                 | Custom eval — per-artifact granularity and auto-fix pay for themselves quickly |
| Need to validate output against real-world inputs | Custom eval — the harness has no equivalent of app coverage testing            |
| Mature skill, long-term quality tracking          | Custom eval — the harness ceiling is too low for deep quality work             |


For this project, the eval skill (`agents/eval/SKILL.md`) provides per-rule review, auto-fix, ground truth generation, and app coverage testing — capabilities the harness does not offer. The harness remains useful as a periodic pipeline health check. Section 7 provides a detailed comparison.

### What this eval covered

The harness was used to evaluate the `generate-rules-with-test` skill — a pipeline that takes a migration guide and produces Konveyor analyzer rules with validated tests.

**How the eval was set up:** The harness read the skill definition and autonomously designed 7 judges across three tiers: a cost budget check, 4 structural output checks (do the right files exist with the right shape), and 2 LLM judges that assess extraction completeness and rule correctness on a 1-5 scale. For test data, the harness identified 2 cases from existing skill output (HttpClient, Spring Boot) and web-searched for 3 more (Log4j, Tomcat, JUnit) to diversify guide complexity and format.

**What the eval found:** Across 2 runs and 5 migration guides, the skill consistently produces high-quality rules — LLM judges scored extraction at 4.2/5 and correctness at 4.0-4.6/5. Every failure in both runs was a pipeline completion issue (turn budget exhaustion), never a quality issue.

---

## Details

## 1. How the Harness Analyzed the Skill and Designed Judges

The evaluation started with `/eval-analyze`, which read the skill definition (`agents/generate-rules-with-test/SKILL.md`) and produced `eval.md` — a structured analysis of the skill's inputs, outputs, pipeline flow, and quality criteria.

### What the analyzer discovered

The analyzer identified that the skill is a **multi-stage pipeline** (ingest → extract → construct → scaffold → test → report) and that its outputs are highly structured — `patterns.json`, rule YAML files, test manifests, test data directories, and a `report.yaml`. This structure made it possible to design judges at two levels:

1. **Structural checks** — does the output exist and have the right shape?
2. **Semantic quality** — is the content correct and complete?



### The 7 judges it proposed

The analyzer proposed all 7 judges, split into three tiers:

**Tier 1: Automated/builtin (1 judge)**


| Judge          | How it works                                                                                                |
| -------------- | ----------------------------------------------------------------------------------------------------------- |
| `budget_check` | Builtin cost comparator — checks `cost_usd` from `run_result.json` against a $35/case cap. No LLM involved. |


**Tier 2: Structured output checks (4 judges)**

These are Python `check:` blocks embedded in `eval.yaml` that parse the skill's output files directly. The analyzer designed them by reading the output schema in the skill definition:


| Judge              | What it parses                                      | Pass condition                                                                                                                          |
| ------------------ | --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `report_exists`    | `report.yaml`                                       | File exists, valid YAML, has `generated_at`, `sources`, `targets`, `rules_total`, `pass_rate`, `rules` array, and `rules_total > 0`     |
| `rules_generated`  | `rules/*.yaml`                                      | At least one rule file exists, sampled rules have valid `ruleID` and `when` fields                                                      |
| `patterns_quality` | `patterns.json`                                     | Valid JSON, patterns have `source_pattern`, `category`, `concern`, `message`; pattern count meets `expected_min_rules` from annotations |
| `test_coverage`    | `tests/manifest.json`, `*.test.yaml`, `tests/data/` | Manifest exists, test YAML files exist, test data directory is populated                                                                |


These checks are deterministic — they either pass or fail based on file existence and structure. The `patterns_quality` judge is the only one that references the test case annotations (`expected_min_rules`), tying the check to a per-case quality bar.

**Tier 3: LLM judges (2 judges)**

These are the most interesting. The analyzer identified two quality dimensions that require judgment rather than structural checks:


| Judge                     | What it evaluates                                                                                                                                                       | Scoring rubric                                                                                |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| `extraction_completeness` | Compares `guide.md` against `patterns.json` — were all migration artifacts in the guide captured as patterns?                                                           | 1-5 scale: 1 = <40% extracted, 5 = 95%+ extracted                                             |
| `rule_correctness`        | Reviews rule YAML files for accurate `when` conditions, before/after code examples, specific documentation URLs, correct category/effort, and no hallucinated artifacts | 1-5 scale: 1 = wrong conditions or hallucinations, 5 = all rules accurate with clear examples |


Both LLM judges use Claude Opus 4.6 (same model as the skill) and receive the full skill output via `{{ outputs }}` template injection. The threshold for both is `min_mean: 3.5` across all cases.

The judge prompts are specific about what to check. For example, `extraction_completeness` explicitly asks: "Are there classes/types/methods named in the guide that don't appear as source_pattern? Are there dependencies listed that lack corresponding patterns? Are there configuration property changes not captured?" This level of specificity came directly from the analyzer understanding the skill's output schema.

### Why this judge set worked well

The three-tier design proved effective because each tier catches different failure modes:

- **Builtin** catches runaway cost before anything else is examined
- **Structural checks** catch pipeline completion failures — if test data or the report is missing, the pipeline didn't finish
- **LLM judges** catch quality issues that structural checks can't see — a rule file can exist and have valid YAML structure but still have wrong `when` conditions or miss key migration artifacts

In both runs, the structural checks (`report_exists`, `test_coverage`) caught pipeline completion failures (case-004 in v1, case-003 in v2), while the LLM judges confirmed that quality was high even on incomplete cases.

---



## 2. How the Harness Sourced Test Data

The harness created 5 test cases from two sources:

### From existing skill output (2 cases)

The `harness`examined prior skill runs in the project's `output/` directory and identified two migration guides that had already been successfully processed:


| Case     | Guide URL                                                                             | Why selected                                                             |
| -------- | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| case-001 | `https://hc.apache.org/httpcomponents-client-5.6.x/migration-guide/preparation.html`  | Found in existing output — a small, focused best-practices migration doc |
| case-002 | `https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide` | Found in existing output — a large, comprehensive migration wiki page    |




### From web search (3 cases)

The agent then searched the web for additional Java migration guides to test the skill against unseen inputs. It selected guides that diversified the test suite along two axes: **guide complexity** (number of migration patterns) and **guide format** (tables vs prose vs tutorials):


| Case     | Guide URL                                                       | Why selected                                                                                                              |
| -------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| case-003 | `https://logging.apache.org/log4j/2.x/migrate-from-log4j1.html` | Reference-heavy with numbered tables (7 tables mapping old → new APIs) — tests extraction from structured tabular content |
| case-004 | `https://tomcat.apache.org/migration-10.html`                   | Short official migration notes — tests handling of compact guides with javax→jakarta namespace changes                    |
| case-005 | `https://www.baeldung.com/junit-5-migration`                    | Tutorial-style with inline code examples — tests extraction from narrative, example-driven content                        |




### How each case is structured

Every case directory contains two files:

- `input.yaml` — the `guide_source` URL passed to the skill, plus optional `sources`/`targets` labels
- `annotations.yaml` — expected outcomes for annotation-aware judging: `expected_min_rules`, `expected_language`, `expected_categories`, and `key_patterns` (specific FQN values that must appear)

The annotations were written by the setup agent based on reading each guide and estimating what the skill should extract. No `guide.md` files are stored in the case directories — all 5 cases fetch the guide at runtime via URL, meaning the eval tests the full ingest pipeline.

### Test data design strengths

The mix of sourcing approaches created a well-rounded test suite:

- **Existing output cases** validate that the eval agrees with prior human review
- **Web-searched cases** test generalization to unseen guides
- **Complexity spread** ranges from 4 rules (HttpClient) to 72 rules (Spring Boot)
- **Format diversity** covers wiki pages, official docs, reference tables, and tutorials
- **All URL-based** means every run tests the full pipeline from guide fetch to final report

---



## 3. Results Overview



### Run v1 (baseline)


| Judge                   | Result     | Status              |
| ----------------------- | ---------- | ------------------- |
| budget_check            | 100% pass  | **PASS**            |
| rules_generated         | 100% pass  | **PASS**            |
| patterns_quality        | 100% pass  | **PASS**            |
| extraction_completeness | 4.2/5 mean | **PASS**            |
| rule_correctness        | 4.0/5 mean | **PASS**            |
| report_exists           | 80% (4/5)  | **FAIL** — case-004 |
| test_coverage           | 80% (4/5)  | **FAIL** — case-004 |


**$52.16 total | 880 turns | 26 min wall clock**

### Run v2 (with fixes)


| Judge                   | Result     | Status                 |
| ----------------------- | ---------- | ---------------------- |
| budget_check            | 100% pass  | **PASS**               |
| rules_generated         | 100% pass  | **PASS**               |
| extraction_completeness | 4.2/5 mean | **PASS**               |
| rule_correctness        | 4.6/5 mean | **PASS**               |
| patterns_quality        | 90% mean   | **PARTIAL** — case-001 |
| report_exists           | 80% (4/5)  | **FAIL** — case-003    |
| test_coverage           | 80% (4/5)  | **FAIL** — case-003    |


**$66.63 total | 780 turns | 32 min wall clock**

### Per-case results (v2)


| Case                   | Source          | Rules | Cost   | Extraction | Correctness | Tests      |
| ---------------------- | --------------- | ----- | ------ | ---------- | ----------- | ---------- |
| case-001 (HttpClient)  | Existing output | 4     | $5.95  | 4/5        | 4/5         | 100%       |
| case-002 (Spring Boot) | Existing output | 72    | $18.86 | 4/5        | 4/5         | 100%       |
| case-003 (Log4j)       | Web search      | 42    | $27.10 | **5/5**    | **5/5**     | Incomplete |
| case-004 (Tomcat)      | Web search      | 14    | $7.18  | 4/5        | **5/5**     | 100%       |
| case-005 (JUnit)       | Web search      | 16    | $7.54  | 4/5        | **5/5**     | 93.75%     |




### What improved between runs


| Metric                | v1                                 | v2                                       |
| --------------------- | ---------------------------------- | ---------------------------------------- |
| rule_correctness mean | 4.0/5                              | **4.6/5**                                |
| case-004 (Tomcat)     | Stuck at 275 turns, never finished | Completed in 115 turns, 14/14 tests pass |
| Test execution        | 0% all cases (no JAVA_HOME)        | 93-100% where pipeline completes         |
| Permission denials    | ~20-227 per case                   | Near zero                                |


---



## 4. What the Eval Found



### Structural judges caught pipeline completion issues

The `report_exists` and `test_coverage` judges were the primary failure detectors. In both runs, they identified the exact case where the pipeline didn't finish:

- v1: case-004 (Tomcat) — stalled during test generation due to sandbox permission denials
- v2: case-003 (Log4j) — exhausted 205 turns on a complex 42-rule guide before test data could be written

These are pipeline completion issues, not quality issues. The LLM judges confirmed this — both incomplete cases scored 4-5/5 on extraction and correctness.

### LLM judges provided actionable quality feedback

The rationales from `extraction_completeness` and `rule_correctness` were detailed enough to drive specific improvements:

**Extraction gaps identified:**

- case-001: 10 patterns with non-FQN source names that couldn't become rules
- case-002: Jackson 2 compatibility properties and classic starters not captured
- case-004: HTTP/2 connector config and logging behavioral change missed

**Correctness issues identified:**

- case-001 (v1): 3/5 — 10 dropped patterns left key migration recipes uncovered
- case-002: 2 rules failed source verification (JsonValueDeserializer, StreamBuilderFactoryBeanCustomizer)
- case-004: Minor overlap between javax.servlet* and javax.servlet.jsp* patterns

**Top quality performers:**

- case-003 (Log4j): 5/5 on both judges — most comprehensive extraction of any case
- case-005 (JUnit): 5/5 correctness — all FQNs verified against junit-4.13.2.jar, zero hallucinations



### The `patterns_quality` check needs annotation calibration

The structured `patterns_quality` judge compares pattern count against `expected_min_rules` from annotations. In v2, case-001 scored 0.5 because only 9 patterns were extracted against an annotation of 10. The LLM judge gave 4/5, confirming 90% of available artifacts were captured. The annotation was too high for a best-practices document. This is a dataset quality issue, not a skill or judge issue.

---



## 5. Local Patches Required

The harness (v1.9.1) needed four patches to work correctly for this eval:


| Patch                       | Why needed                                                                                                                    | Files changed                                          |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| `max_turns` support         | Without turn limits, stalled pipelines ran indefinitely (case-004 hit 275 turns in v1)                                        | `config.py`, `base.py`, `claude_code.py`, `execute.py` |
| Per-case metrics in scoring | Scorer read aggregate metrics from `run_result.json`, so budget_check compared total cost ($52) against per-case budget ($35) | `score.py`                                             |
| Rationale parsing fixes     | Judge rationales truncated to 200 chars; regex broke on escaped quotes in JSON                                                | `score.py`                                             |
| Judge `max_tokens=4096`     | Default 1024 tokens was too short for judges evaluating 72+ rules, causing truncated responses and parse failures             | `score.py`                                             |


All four are local patches applied directly to the installed plugin. They have not been upstreamed.

---



## 6. Cost Analysis


| Metric         | v1     | v2     |
| -------------- | ------ | ------ |
| Total cost     | $52.16 | $66.63 |
| Cost per turn  | $0.059 | $0.085 |
| Cost per rule  | $0.31  | $0.45  |
| Cache hit rate | 94.6%  | 93.0%  |


Cost scales with guide complexity:


| Complexity | Example       | Rules | Cost | Turns   |
| ---------- | ------------- | ----- | ---- | ------- |
| Small      | Tomcat, JUnit | 14-16 | $7-8 | 115-125 |
| Medium     | HttpClient    | 4     | $6   | 96      |
| Large      | Spring Boot   | 72    | $19  | 240     |
| Very large | Log4j         | 42    | $27  | 205     |


---



## 7. Comparison with the Project's Eval Skill

The ai-rule-gen project has its own eval system ([evals/](https://github.com/konveyor/ai-rule-gen/tree/main/evals)) built around the `eval` skill (`agents/eval/SKILL.md`). It combines a deterministic CLI (`cmd/eval`) with an LLM judge that does per-rule review — a different approach from agent-eval-harness.

### What the project eval skill does

The eval skill is a two-layer system:

**Layer 1: Deterministic CLI (**`cmd/eval`**)**

Scores pre-existing rule files on structural and runtime metrics:


| Metric                   | What it measures                                                                                            |
| ------------------------ | ----------------------------------------------------------------------------------------------------------- |
| `quality_avg`            | Per-rule quality score (max ~6) — structural checks on `when` conditions, messages, links                   |
| `guidance_depth_avg`     | How detailed the migration guidance is in rule messages                                                     |
| `effective_coverage_pct` | What percentage of a real sample app's migration issues the rules detect (optional — requires a sample app) |
| `overlap_conflict_count` | Rules that fire on the same code (redundancy/conflict)                                                      |
| `specificity_gap_count`  | Ground truth API changes that lack a dedicated rule                                                         |


**Layer 2: LLM judge (per-rule)**

The eval skill reads the migration guide, builds a migration map of every actionable pattern, then reviews **each rule individually** against the guide on two dimensions:

- **Condition accuracy** — does the `when` pattern match the correct old API? Is the condition type appropriate? Could it match unrelated code?
- **Message accuracy** — does the message describe the correct replacement? Does it include concrete code examples?

Each dimension gets a verdict: `pass`, `warn`, or `fail` with a concrete, copy-pasteable fix suggestion. The skill also does cross-rule coherence checks (contradictory advice, implicit ordering dependencies) and gap analysis (guide patterns with no corresponding rule).

**Ground truth flexibility**

The eval skill supports three ground truth modes:

1. **japicmp** — compares old and new JARs to enumerate every API-level breaking change. Most comprehensive, but requires knowing Maven coordinates (Java only).
2. **Guide extraction** — auto-generates ground truth from the migration guide itself using regex + LLM extraction. Lower yield than japicmp but works for any language with no SME input.
3. **None** — skips gap analysis entirely; still runs quality scoring, app coverage, and overlap detection.

When japicmp ground truth is not available, the eval skill auto-generates it from the guide. This means the eval can run against any migration with just a guide URL and a rules directory — no sample app or Maven coordinates required.

**Auto-fix loop**

The eval skill can also fix the issues it finds: it mirrors the ruleset, applies precision fixes, generates rules for gaps, validates with `cmd/validate`, re-runs the deterministic eval, and re-judges — up to 2 iterations, with stop conditions that prevent regressions.

### Coverage and maturity

The project eval covers 8 migrations with up to 18 iterative runs per migration (e.g., HttpClient). It includes 3 generated-vs-handcrafted comparisons. App coverage testing (running rules against a real sample app with kantra) is available for 2-3 migrations where sample apps have been written; the remaining migrations use guide-extracted ground truth and structural scoring.

### How agent-eval-harness differs


| Aspect                        | Project eval skill                                                                       | Agent-eval-harness                                                                |
| ----------------------------- | ---------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **What it evaluates**         | Pre-existing rule files (rules must already be generated)                                | The full skill invocation — from guide URL to final report                        |
| **Pipeline visibility**       | None — only sees the output rules                                                        | Catches pipeline failures (sandbox errors, turn exhaustion, missing output files) |
| **Test generation**           | Not evaluated                                                                            | Judges whether test manifest, test YAMLs, and test data exist and are correct     |
| **Rule quality assessment**   | Per-rule LLM review with pass/warn/fail verdicts and concrete fix suggestions            | Per-case LLM judge with 1-5 scores and rationale                                  |
| **Granularity**               | Individual rule level — each rule gets its own condition and message verdict             | Case level — one score per migration guide covering all rules together            |
| **Ground truth**              | Three modes: japicmp JAR diffing, guide auto-extraction, or none                         | Annotations with expected pattern counts and key FQNs                             |
| **App coverage**              | Runs rules against a real sample app with kantra (where available)                       | Not tested                                                                        |
| **Auto-fix**                  | Built in — fixes precision issues and generates gap rules with regression guards         | Not available                                                                     |
| **Setup for a new migration** | Guide URL + rules directory (minimum). Sample app and japicmp are optional enhancements. | Guide URL only — harness designs judges and sources test cases autonomously       |
| **Per-run cost**              | LLM cost for per-rule judging (scales with rule count) + optional skill invocation cost  | $50-70 per 5-case run (skill execution + LLM judges)                              |




### Results side by side

Where both systems evaluated the same migrations, the findings are consistent:

**HttpClient 4 → 5:**


| Metric              | Project eval                                 | Agent-eval-harness (v2)              |
| ------------------- | -------------------------------------------- | ------------------------------------ |
| Rules evaluated     | 70 (iterated ruleset)                        | 4 (single fresh invocation)          |
| Quality             | 5.94 / 6 (deterministic)                     | 4/5 correctness (LLM)                |
| Coverage            | 92% (against sample app)                     | 4/5 extraction (LLM)                 |
| Ground truth        | 124 entries (japicmp)                        | 10 expected patterns (annotation)    |
| Rule-level findings | Per-rule pass/warn/fail with fix suggestions | Aggregate rationale across all rules |


The project eval ran against a mature, iterated 70-rule set and provides per-rule verdicts. Agent-eval-harness tested a single fresh skill invocation that produced only 4 rules. Different points in the rule lifecycle and different granularity.

**Spring Boot 3 → 4:**


| Metric          | Project eval                     | Agent-eval-harness (v2)           |
| --------------- | -------------------------------- | --------------------------------- |
| Rules evaluated | 94 (iterated ruleset)            | 72 (single invocation)            |
| Quality         | 5.55 / 6                         | 4/5 correctness                   |
| Coverage        | 84% (against sample app)         | 4/5 extraction                    |
| Ground truth    | 999 entries (japicmp, 3 modules) | 40 expected patterns (annotation) |


Both systems found high quality with minor gaps.


### Where each system fits

**The project eval skill is suited for:**

- Per-rule quality assessment with actionable fix suggestions (pass/warn/fail per rule)
- Auto-fixing precision issues and generating rules for gaps
- Ground truth gap analysis — via japicmp when available, or auto-extracted from the guide
- App coverage testing where sample apps exist (2-3 migrations currently)
- Iterating on rule quality after generation 
- Scales to new migrations with just a guide URL and rules directory — sample apps and japicmp are optional enhancements

**Agent-eval-harness is suited for:**

- Testing the skill pipeline end-to-end (ingest → extract → construct → test → report)
- Catching infrastructure issues (permissions, JAVA_HOME, turn budgets)
- Evaluating across multiple unseen guides to test generalization
- Automated test data sourcing and judge design from skill analysis

**Key tradeoff:** The project eval skill provides deeper, per-rule quality assessment with auto-fix capability and can scale to new migrations with minimal setup (guide URL + rules directory). Agent-eval-harness provides pipeline-level validation that catches failures before rules even exist. The project eval skill cannot detect pipeline completion failures; agent-eval-harness cannot do per-rule review or auto-fix. They are complementary.

---



## 8. Conclusions



### What the harness delivered

1. **Judge design from skill analysis worked.** The `/eval-analyze` agent read the skill definition and produced 7 judges across 3 tiers — all were meaningful, none were redundant, and together they covered both structural completeness and semantic quality.
2. **Mixed test data sourcing created a strong suite.** Using 2 cases from existing output (known-good baseline) plus 3 web-searched cases (unseen inputs) balanced validation against exploration. The web-searched cases produced the highest-quality output (Log4j 5/5) and revealed a new failure mode (turn exhaustion on complex guides).
3. **LLM judges provided actionable rationales.** The detailed scoring rationales — especially after the truncation fix — gave specific, per-case feedback on missed patterns, FQN accuracy, and source verification results. These directly informed skill improvements.
4. **Structural judges caught what LLM judges cannot see.** Pipeline completion failures are invisible to LLM judges (which only assess quality of what exists), but the `report_exists` and `test_coverage` checks caught them immediately.


