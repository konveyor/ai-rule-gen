# Test Data Generation Guide

This guide explains how to generate test source code that triggers Konveyor analyzer rules. Derived from the test data generation template.

## Contents

- Goal
- Requirements
- Source Code Must Use the OLD (Source) API
- How the Analyzer Matches Each Condition Type
- java.dependency version bounds
- Output Format
- Project Structure Per Language
- Reading the manifest.json
- Dependency Resolution
- XML Sanitization
- Merging Small Test Groups
- Fix Iterations

## Goal

For each rule, generate a COMPLETE, COMPILABLE project where the source code contains code that EXACTLY matches the rule's `when` condition pattern. The analyzer will run against this code and must find at least 1 incident per rule.

## Requirements

1. Create a COMPLETE, COMPILABLE project for the detected language
2. For EACH rule, include code that EXACTLY matches the pattern in the `when` condition
3. Add a comment before each pattern: `// Rule: <ruleID>`
4. Keep code minimal — one example per rule, just enough to trigger the pattern
5. All imports/dependencies must be valid and resolve

**Rules for choosing test versions (java.dependency):**
1. The version MUST be strictly less than the `upperbound`
2. Use a realistic version that actually exists for the artifact (check Maven Central)
3. **Use plain numeric versions** (e.g., `2.3.0`) — kantra cannot compare qualified versions like `2.3-groovy-4.0` or `2.4-M4-groovy-4.0` against numeric bounds
4. When in doubt, use a version in the `3.x` range — it's almost always below the upperbound and is a realistic Spring Boot 3.x era version
5. Use the `name` field as `groupId.artifactId` (dot-separated). E.g., name `org.springframework.boot.spring-boot-starter-undertow` → `<groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-undertow</artifactId>`

## Source Code Must Use the OLD (Source) API

This is a common mistake. The test data simulates code that has NOT been migrated yet. Every import, annotation, type reference, and dependency version must use the **source** (pre-migration) API — never the target.

Example: if a rule detects `org.springframework.boot.autoconfigure.http.HttpMessageConverters` (the 3.x package), the test file must import from that exact path. Do NOT use the 4.x relocated path `org.springframework.boot.http.converter.autoconfigure.HttpMessageConverters` — the rule won't match and the test fails.

The `pattern` field in the rule YAML tells you the exact FQN to use in the test code. Copy it verbatim.

## How the Analyzer Matches Each Condition Type

This is CRITICAL — the analyzer matches patterns against fully qualified names and source references. If you don't follow these rules, the test code won't trigger the rule.

| Condition Type | Location | What the Test Code Must Do |
|---|---|---|
| `java.referenced` | `ANNOTATION` | USE the annotation on a class, method, or field (e.g., `@MockBean private Object svc;`). **An `import` statement alone is NOT enough** — the annotation must appear as `@AnnotationName` on an actual element. |
| `java.referenced` | `IMPORT` | Include the import statement (e.g., `import javax.servlet.http.HttpServlet;`) |
| `java.referenced` | `TYPE` | Declare or use the type (e.g., `HttpServlet servlet;` or a cast) |
| `java.referenced` | `METHOD_CALL` | Call the method on an explicitly typed variable — do NOT chain calls (e.g., use `Foo f = Foo.get(); f.bar();` not `Foo.get().bar()`). JDTLS in source-only mode cannot resolve return types of chained calls. |
| `java.referenced` | `CONSTRUCTOR_CALL` | Use `new ClassName()` |
| `java.referenced` | `INHERITANCE` | `class Foo extends TargetClass` |
| `java.referenced` | `IMPLEMENTS_TYPE` | `class Foo implements TargetInterface` |
| `java.referenced` | `FIELD` | Declare a field of that type |
| `java.referenced` | `VARIABLE_DECLARATION` | Declare a local variable of that type |
| `java.referenced` | `RETURN_TYPE` | Declare a method with that return type |
| `go.referenced` | — | Import and use the package/symbol (e.g., `import "golang.org/x/crypto/md4"` then use `md4.New()`) |
| `nodejs.referenced` | — | Import and use the symbol (e.g., `import { Button } from '@patternfly/react-core';`) |
| `csharp.referenced` | — | Use the fully qualified type/symbol |
| `builtin.filecontent` | — | Include text in the appropriate file that matches the regex pattern. Check the `filePattern` field to know which file type. Note: `filePattern` is a Go regex, not a glob. |
| `java.dependency` | — | The `pom.xml` must declare the dependency with a version that satisfies the rule's bounds. See the **java.dependency version bounds** section below. **No source code needed** — only the pom.xml matters. |
| `go.dependency` | — | The `go.mod` must declare the module dependency with a version within the rule's bounds |
| `builtin.xml` | — | The XML file (usually `pom.xml`) must contain elements matching the XPath expression. If `filepaths` is set, the file must be at that path. If `namespaces` is set, ensure the XML uses those namespace URIs |

## java.dependency version bounds

This is the most common cause of test failures. Read carefully.

A `java.dependency` rule matches when:
- The pom.xml declares the artifact AND
- The declared version is **strictly less than** the `upperbound` (if set) AND
- The declared version is **greater than or equal to** the `lowerbound` (if set)

**If the declared version is >= upperbound, the rule does NOT match and the test FAILS.**

Example: a rule with `name: org.flywaydb.flyway-core` and `upperbound: 4.0.0`:
- `<version>3.2.1</version>` → MATCHES (3.2.1 < 4.0.0)
- `<version>9.22.0</version>` → DOES NOT MATCH (9.22.0 >= 4.0.0) — **test fails**
- `<version>8.13.0</version>` → DOES NOT MATCH (8.13.0 >= 4.0.0) — **test fails**

**Handling artifacts that only publish qualified versions:**

Some artifacts never publish plain numeric versions:
- **Spock** (`org.spockframework:spock-spring`): versions are `2.3-groovy-3.0`, `2.3-groovy-4.0`, etc.
- **Hibernate** (`org.hibernate.orm:hibernate-*`): versions are `6.4.0.Final`, `6.5.2.Final`, etc.

For these artifacts, use the Spring Boot parent BOM to manage the version. Declare the dependency **without** a `<version>` tag:

```xml
<dependency>
    <groupId>org.spockframework</groupId>
    <artifactId>spock-spring</artifactId>
    <!-- version managed by spring-boot-starter-parent BOM -->
</dependency>
```

The BOM resolves a version that the kantra parser can extract and compare. If the BOM doesn't manage the artifact, you must use the qualified version — kantra handles `.Final` suffixes but NOT `-groovy-4.0` style qualifiers. Test and verify.

**Handling discontinued artifacts:**

Some artifacts were discontinued before the migration target version. For example, `hibernate-proxool` and `hibernate-vibur` were dropped in Hibernate 6 — they were never published under `org.hibernate.orm`.

For discontinued artifacts:
1. Use the **last published groupId and version**. Example: `org.hibernate:hibernate-proxool:5.6.15.Final` (not `org.hibernate.orm:hibernate-proxool:6.4.0.Final` which never existed)
2. Check the rule's `dependency_name` field — it specifies the groupId.artifactId to use. If the rule says `org.hibernate.orm.hibernate-proxool`, use that exact groupId/artifactId, but with a version that actually existed for that coordinate
3. If no version of the artifact was ever published under the rule's groupId, flag it — the rule's `dependency_name` may need correction

## Output Format

Generate EXACTLY TWO fenced code blocks per test group:

**FIRST block:** Build file contents (pom.xml, go.mod, package.json, or Project.csproj)

**SECOND block:** Main source file contents (Application.java, main.go, App.tsx, or Program.cs)

Do NOT include any other text or code blocks.

## Project Structure Per Language

### Java
```
<data-dir>/
├── pom.xml                                        # Maven build file
└── src/main/java/com/example/Application.java     # Source code
```
- Build file: `pom.xml` (type: xml)
- Source dir: `src/main/java/com/example`
- Main file: `Application.java` (type: java)
- Dependencies must be valid Maven coordinates that resolve

### Go
```
<data-dir>/
├── go.mod     # Module definition
└── main.go    # Source code
```
- Build file: `go.mod` (type: go)
- Source dir: `.` (root)
- Main file: `main.go` (type: go)
- After writing, run `go mod tidy` (do NOT vendor — vendored deps cause false positives in analysis)

### Node.js / TypeScript
```
<data-dir>/
├── package.json    # NPM package definition
└── src/App.tsx     # Source code
```
- Build file: `package.json` (type: json)
- Source dir: `src`
- Main file: `App.tsx` (type: tsx)

### C# / .NET
```
<data-dir>/
├── Project.csproj    # .NET project file
└── Program.cs        # Source code
```
- Build file: `Project.csproj` (type: xml)
- Source dir: `.` (root)
- Main file: `Program.cs` (type: csharp)

## Reading the manifest.json

`go run ./cmd/scaffold` outputs a `manifest.json` that tells you exactly what files to generate:

```json
{
  "language": "java",
  "groups": [
    {
      "name": "web",
      "data_dir": "tests/data/web",
      "test_file": "tests/web.test.yaml",
      "rule_count": 3,
      "providers": ["java"],
      "files": [
        {"path": "tests/data/web/pom.xml", "file_type": "xml", "purpose": "build"},
        {"path": "tests/data/web/src/main/java/com/example/Application.java", "file_type": "java", "purpose": "source"}
      ],
      "rule_ids": ["rule-00010", "rule-00020", "rule-00030"]
    }
  ]
}
```

For each group:
1. Read the rules referenced by `rule_ids` to see what patterns must be matched
2. Generate the build file at the `path` with `purpose: "build"`
3. Generate the source file at the `path` with `purpose: "source"`
4. The source code must trigger ALL rules in that group

## Dependency Resolution

- **Go:** Always run `go mod tidy` (do NOT run `go mod vendor` — vendored deps cause false positives in `go.referenced` analysis)
- **Java:** Do NOT run `mvn compile`, `mvn dependency:resolve`, or any Maven command. Do NOT import the project into an IDE. Kantra resolves `java.dependency` rules by parsing the pom.xml directly, and source-only analysis resolves IMPORT/ANNOTATION/TYPE patterns without downloaded JARs. Running Maven or IDE imports creates `.classpath`, `.project`, `.settings/`, `target/`, and `.factorypath` artifacts that pollute the test data.
- **Node.js:** `npm install` only if needed for type resolution
- **C#:** `dotnet restore` only if needed for type resolution

### Java pom.xml: prefer minimal dependencies

Prefer the lightest dependency that provides the class you need — e.g., `spring-boot-autoconfigure` instead of a full starter like `spring-boot-starter-web`. Lighter dependencies reduce analysis time and avoid resolution issues.

### Use realistic dependency versions

Test pom.xml files must use versions that actually exist on Maven Central. Do NOT fabricate version numbers. Common mistakes:
- `elasticsearch-rest-client:3.9.0` — this artifact never had a 3.x release (real versions: 7.x, 8.x)
- `hibernate-jpamodelgen:3.6.0.Final` — very old and likely not what was intended
- `hibernate-proxool:6.4.0.Final` — this module was dropped before Hibernate 6, never published under `org.hibernate.orm`
- `spock-spring:2.3.0` — plain `2.3.0` does not exist; real versions are `2.3-groovy-3.0`, `2.3-groovy-4.0`

When you need a version below an upperbound and don't know a real version, use the Spring Boot parent BOM version. For Spring Boot 3.x migrations, `spring-boot-starter-parent` version `3.2.0` or `3.3.0` with managed dependencies is the safest approach — the BOM resolves correct transitive versions automatically.

**Version verification checklist** (apply to every `java.dependency` rule in the group):
1. Is this version plain numeric? If not (e.g., `.Final`, `-groovy-4.0`), prefer BOM-managed
2. Does this groupId + artifactId + version actually exist? If unsure, omit `<version>` and let the BOM manage it
3. Is the version strictly below the upperbound?
4. Was this artifact ever published under this groupId? (e.g., `org.hibernate.orm` vs `org.hibernate`)

### java.dependency and builtin.xml tests

These condition types do not use JDTLS. Test YAML files for `java.dependency` and `builtin.xml` rules should omit `analysisParams: mode: source-only`.

## XML Sanitization

After generating all test files, run `go run ./cmd/sanitize --dir <tests-dir>` to clean XML files. LLMs frequently generate comments like `<!-- --add-opens flag -->` which contains `--` inside a comment — this is illegal XML and breaks Maven's POM parser. The sanitizer replaces `--` sequences inside XML comments with spaces.

## Merging Small Test Groups

When many small test groups use the same provider (e.g., multiple groups with 1-3 `java.referenced` rules each), consider merging them into fewer groups. Each group runs a separate JDTLS session, and too many sessions increase analysis time. For example, merge `jackson-1` (8 rules) and `jackson-2` (3 rules) into a single `jackson-1` group with 11 rules — one JDTLS session instead of two.

The scaffold command groups by concern automatically, but manual merging may be needed when:
- Multiple concerns share the same provider and have few rules each

To merge: edit the `.test.yaml` file to add the extra rule entries, and add the corresponding test code to the shared source file. Update the `rulesPath` in the `.test.yaml` if rules are in different YAML files (you can list multiple rules paths or restructure).

## Fix Iterations

When a rule fails kantra tests, you'll receive:
- The failing rule IDs
- Their patterns (from the rule YAML)
- Failure context from the validator

On fix iterations:
- Regenerate ONLY the failing test groups (preserve passing groups)
- Use the fix guidance to understand what the test code is missing
- The most common failure is: the test code doesn't actually use the API that the rule pattern matches

**Rule integrity:** The rule is authoritative — never change a rule to make a test pass. Fix the test data instead.
