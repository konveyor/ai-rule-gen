# ai-rule-gen

Skill-first architecture. All LLM orchestration lives in agent skills.
Go CLI is purely deterministic — no LLM calls, no API keys, no prompt templates.

## Global Tool Restrictions

Agents MUST only use tools from their skill's permissions table. FORBIDDEN:
`python`, `python3`, `pip`, `pip3`, `node`, `npm`, `npx`, `ruby`, `perl`,
`bash -c`, `sh -c`, and any scripting runtime not explicitly listed.
Do not write or execute ad-hoc scripts in any language.
If a task cannot be done with allowed tools, report the limitation.

## Active Technologies

- Go 1.25+ with stdlib `flag`, `gopkg.in/yaml.v3`, `github.com/JohannesKaufmann/html-to-markdown`, `golang.org/x/net`

## Project Structure

```text
cmd/                      # CLI commands (go run ./cmd/<name>)
  ingest/                 # Fetch migration guide → clean markdown
  sections/               # Index guide sections with content classification
  construct/              # patterns.json → rule YAML + ruleset.yaml
  validate/               # Validate rule YAML files
  merge-patterns/         # Merge partial patterns files with deduplication
  verify/                 # Verify extracted FQNs against published source artifacts
  contract-validate/      # Validate skill input/return payloads against contract.json
  scaffold/               # Create test dirs, .test.yaml, manifest.json
  sanitize/               # Fix illegal XML comments in a directory
  test/                   # Run kantra tests (sequential, auto-retry timeouts)
  report/                 # Generate YAML summary report
  coverage/               # Post-extraction coverage check
  log/                    # Append structured events to pipeline log
  eval/                   # Deterministic quality scoring + app coverage analysis
  ground-truth/           # Extract API changes via japicmp or guide parsing
  feedback/               # Aggregate pipeline run trends and failure analysis
  compare/                # Compare rulesets side-by-side with optional app coverage
internal/                 # Go library code (no LLM dependencies)
agents/                   # Agent skills (agentskills.io format)
  <skill>/SKILL.md        # Skill definition (inputs, returns, workflow)
  <skill>/contract.json   # Machine-readable input/return contract
  <skill>/references/     # Skill-specific reference docs
languages/<lang>/         # Per-language scaffold config (java, go, nodejs, csharp, python)
```

## Skill Composition

```text
generate-rules (orchestrator skill)
  ├── rule-writer        — extract patterns, produce rule YAML
  ├── test-generator     — generate test source code (parallel)
  └── rule-validator     — fix failing tests, verify loop
```

All four are skills under `agents/`, each with `SKILL.md`, `contract.json`,
and `## Inputs` / `## Returns` sections defining their contract.

## Sub-agent Dispatch

When a skill says **"Invoke: `<skill-name>`"**, dispatch a new agent:

1. Spawn an isolated agent using your runtime's agent dispatch tool.
2. Prompt: "Read and follow `agents/<skill-name>/SKILL.md`." + the invoke inputs.
3. Wait for results.

**The orchestrator must NOT execute the sub-skill's workflow itself.** Dispatch and wait.
The orchestrator must NOT read files under `agents/<skill>/references/` — sub-agents
read their own references.
Do not pre-digest data or compose step-by-step instructions for the sub-agent.
If `Parallel: yes`, dispatch concurrently when your runtime supports it.

**Fallback:** If your runtime cannot dispatch sub-agents, read the SKILL.md and
follow its workflow exactly as written — do not reinterpret or reimagine the steps.

## CLI Commands

```bash
go run ./cmd/ingest    --input <url-or-file> --output output/<src>-to-<tgt>/guide.md
go run ./cmd/sections  --guide output/<src>-to-<tgt>/guide.md
go run ./cmd/construct --patterns output/<src>-to-<tgt>/patterns.json --output output/<src>-to-<tgt>/rules/
go run ./cmd/validate  --rules output/<src>-to-<tgt>/rules/
go run ./cmd/merge-patterns --output output/<src>-to-<tgt>/patterns.json output/<src>-to-<tgt>/patterns-*.json
go run ./cmd/contract-validate --contract agents/<skill>/contract.json --mode inputs|returns --payload-file payload.json
go run ./cmd/scaffold  --rules output/<src>-to-<tgt>/rules/ --output output/<src>-to-<tgt>/tests/
go run ./cmd/sanitize  --dir output/<src>-to-<tgt>/tests/data/
go run ./cmd/test      --rules output/<src>-to-<tgt>/rules/ --tests output/<src>-to-<tgt>/tests/ [--files a.test.yaml,b.test.yaml] [--timeout 5m]
go run ./cmd/report    --source <src> --target <tgt> --output output/<src>-to-<tgt>/report.yaml [--kantra-limitation N] [--kantra-limitation-rules id4,id5]
go run ./cmd/coverage  --guide output/<src>-to-<tgt>/guide.md --patterns output/<src>-to-<tgt>/patterns.json [--language java]
go run ./cmd/verify    --patterns output/<src>-to-<tgt>/patterns.json --output output/<src>-to-<tgt>/verify-results.json --cache output/<src>-to-<tgt>/verify-cache
go run ./cmd/log       --log output/<src>-to-<tgt>/pipeline.log --agent <agent-name> --model <model-id> --message "<message>"
go run ./cmd/eval      --rules-dir output/<src>-to-<tgt>/rules/ [--app-dir <app>] [--ground-truth <ground_truth.yaml>] [--save] [--compare <baseline.json>]
go run ./cmd/ground-truth --old-artifact <groupId:artifactId:version> --new-artifact <groupId:artifactId:version> [--output ground_truth.yaml]
go run ./cmd/feedback  [--output-dir output] [--migration-path <filter>] [--format text|json] [--min-runs 2]
go run ./cmd/compare   --a output/<src>-to-<tgt>-run1/rules/ --b output/<src>-to-<tgt>-run2/rules/ [--app-dir <app>] [--out report.md]

go test ./internal/...   # Unit tests
```

## Session Logging

All CLI commands support `--log`, `--agent`, and `--model` flags for pipeline logging with agent/model attribution. Pass `--log <path>` to append timestamped JSON output to a log file. Pass `--model <id>` when an LLM agent invokes the command (logs `[model=<id>]`). Omit `--model` only for manual human CLI usage (logs `[cli]`).

```bash
# Orchestrator invocations (orchestrator is an LLM agent):
go run ./cmd/ingest --log pipeline.log --agent orchestrator --model claude-opus-4-6 --input ... --output ...
go run ./cmd/test   --log pipeline.log --agent orchestrator --model claude-opus-4-6 --rules ... --tests ...

# Sub-agent invocations (e.g., rule-validator running tests):
go run ./cmd/test   --log pipeline.log --agent rule-validator --model claude-sonnet-4-20250514 --rules ... --tests ...

# Human manual invocation (no --model):
go run ./cmd/validate --log pipeline.log --agent manual --rules ...
```

Log format:
- Agent call: `[HH:MM:SS] [cmd-name] [agent=X] [model=Y] output: {json}`
- Manual CLI call: `[HH:MM:SS] [cmd-name] [agent=X] [cli] output: {json}`

The `RULE_GEN_LOG` environment variable also works as a fallback — `--log` takes precedence if both are set.

## Key Concepts

**patterns.json** — Intermediate JSON between agent extraction and `cmd/construct`.
Contains source, target, language, and MigrationPattern objects (source_fqn,
dependency_name, provider_type, location_type, complexity, category, concern).

**manifest.json** — Output of `cmd/scaffold`. Tells the agent what source files
to generate per test group.

## Key Packages

### `internal/rules`

Core types for migration patterns and rules.

- **`MigrationPattern`** — extracted pattern from a migration guide
- **`ArtifactCoordinates`** — identifies a published library in a package registry (groupId, artifactId, version)
- **`Rule`** — a Konveyor analyzer rule with conditions, labels, messages
- **`ExtractOutput`** — the patterns.json schema (source, target, language, patterns)

### `internal/verify`

Deterministic verification of extracted patterns against real published library artifacts.

- **`Verifier` interface** — per-language verification (`Verify(pattern) → Result`)
- **`JavaVerifier`** — downloads JARs from Maven Central, runs `jar tf`, checks FQN existence
- **`Run(extract, cacheDir)`** — verify all patterns, returns `[]Result`
- **`Summarize(results)`** — aggregate results into verified/not_found/skipped/offline counts
- **`CleanCache(cacheDir)`** — remove cached artifacts

Cache layout: `<cacheDir>/<groupId>/<artifactId>/<version>/classes.txt`

### `internal/construct`

Converts patterns into rule YAML files.

- **`Run(extract, rulesDir)`** — returns `Result` with `PatternRuleMap` (pattern index → rule ID)
- `PatternRuleMap` enables post-construction operations (like verification labeling) to map pattern-level results to rule IDs

### `internal/workspace`

Pipeline workspace management.

- **`Workspace`** — manages output directory structure
- **`Report`** — pipeline summary with per-rule `RuleStatus` (test_status, source_verified) and optional `Verification *VerificationStats`
- **`VerifyCacheDir()`** — returns path for cached verification artifacts

## Verification

The verification layer checks whether extracted FQNs actually exist in published source library artifacts. This catches hallucinated patterns that the LLM creates.

1. Rule-writer agents emit `source_artifact` metadata on each pattern
2. `cmd/verify` downloads the JAR from Maven Central and lists its classes via `jar tf`
3. Checks if the FQN exists in the class listing
4. Results are recorded in `report.yaml` as per-rule `source_verified` status:
   - `source_verified: "true"` — FQN confirmed in published artifact
   - `source_verified: "false"` — FQN not found in published artifact
   - Omitted if verification didn't run (offline, no source_artifact, unsupported language)

Currently Java only. The `Verifier` interface supports adding Go, Node.js, and C# verifiers.
When Maven Central is unreachable, verification skips without error — all patterns get `skipped` status and the pipeline continues normally.

## Labels

Rules carry only stable metadata labels. Pipeline status (test results, verification) lives in `report.yaml`, not on rules.

### Rule labels (set by `cmd/construct`)

| Label | Purpose |
|-------|---------|
| `konveyor.io/source=<source>` | Source technology |
| `konveyor.io/target=<target>` | Target technology |
| `konveyor.io/generated-by=ai-rule-gen` | Generation provenance |

### Report per-rule status (set by `cmd/report`)

| Field | Values | Meaning |
|-------|--------|---------|
| `test_status` | `passed`, `failed`, `kantra-limitation`, `untested` | Kantra test result |
| `source_verified` | `true`, `false`, or omitted | FQN verified against published artifact |

## Testing

```bash
go test ./...                                    # Unit tests
go test ./... -tags=integration -timeout 120s    # Integration tests (requires network, jar)
```

Files tagged with `//go:build integration` require external resources (Maven Central, jar command) and are skipped by default.

## Code Style

Go 1.25+. Standard conventions. No LLM dependencies in Go code.
All new code needs tests. CLI commands output JSON to stdout.
Structured logging via `cmd/internal/cli`. Verification is additive and non-blocking.
