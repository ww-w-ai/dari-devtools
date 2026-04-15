---
name: validate-tests
description: Detects test anti-patterns based on behavior testing principles
triggers:
  - validate tests
  - check tests
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Glob
  - Grep
  - Bash
---

# validate-tests -- Test Anti-Pattern Detection

Corresponding rule: `testing.md`

## Purpose

Statically detect anti-patterns in test code based on behavior-driven testing principles (testing.md). Identify patterns that hurt maintainability, such as implementation-detail testing, excessive mocking, and shared global state.

## Input

- Target: test files (`*.test.*`, `*.spec.*`, `test_*`, `*_test.*`)
- Default search path: entire project. If CLAUDE.md specifies a test directory, that path takes precedence.

## Validation Categories

### 1. Vague Test Names

Detect patterns where test names provide no clue to the cause of failure.

Search patterns:
- `test_\d+$` (names ending only in a number: `test_1`, `test_2`)
- `test_it_works`
- `test_function`
- `test_ok`
- `test_success`
- `it\(["']works["']` (JS/TS: `it('works')`)
- `it\(["']should work["']`
- `test\(["']test["']` (JS/TS: `test('test')`)

Severity: **WARN**

### 2. Implementation-Detail Tests

Detect patterns that verify call count/existence of internal methods.

Search patterns:
- `.call_count`
- `.called`
- `.assert_called_with`
- `.assert_called_once`
- `.assert_called_once_with`
- `.assert_any_call`
- `.toHaveBeenCalledTimes` (Jest)
- `.toHaveBeenCalledWith` (Jest)
- `verify(.*).times(` (Java Mockito)

Severity: **WARN**

### 3. Excessive Mocks

Detect patterns where a single test function uses 4 or more Mock objects.

Search patterns:
- `Mock()` or `mock.patch` appears 4+ times in a single test function
- `jest.fn()` or `jest.mock` appears 4+ times in a single test
- `@mock.patch` decorators: 4+ on a single test function

Severity: **WARN** (when 4 or more)

### 4. Shared Global State

Detect patterns where test files create instances at module level, sharing state across tests.

Search patterns:
- `= new ` (instance creation) -- at the top level of a test file (outside functions/classes)
- Variables with `shared_` prefix
- Module-level `let ` declarations that are mutated by tests (JS/TS)
- Use of the `global` keyword at the top level of a test file

Severity: **WARN**

### 5. DB Dependency in Unit Tests

Detect patterns where unit test files directly call DB connections/queries. Exclude integration test files whose names contain `integration`, `e2e`, or `integ`.

Search patterns:
- `db.query`
- `db.execute`
- `db.session`
- `db.connect`
- `prisma.` (query methods)
- `mongoose.connect`
- `connection.execute`
- `pool.query`

Exclusion condition: file name contains `integration`, `e2e`, or `integ`

Severity: **WARN**

## Execution Logic

1. `Glob` to collect test files (`*.test.*`, `*.spec.*`, `test_*`, `*_test.*`)
2. Categories 1, 2, 5: `Grep` each pattern across all test files
3. Category 3: `Read` test files and tally Mock count per function
4. Category 4: `Read` test files and detect instance creation at module level (outside functions/classes)
5. Group detected anti-patterns by category and output as a table

### Mandatory Output: Test Validation Matrix

Output the matrix below before generating the final report. Do not proceed until every category has been scanned.

| Category | Status | Items Checked | Findings | Severity | Evidence |
|----------|:------:|:-------------:|:--------:|:--------:|----------|
| Vague Test Names | ? | ? | ? | WARN | {Glob/Grep, files, pattern} |
| Implementation-Detail Tests | ? | ? | ? | WARN | {Glob/Grep, files, pattern} |
| Excessive Mocks | ? | ? | ? | WARN | {Read, scanned files} |
| Shared Global State | ? | ? | ? | WARN | {Read, scanned files} |
| DB Dependency | ? | ? | ? | WARN | {Grep, files, pattern} |

Status values: PASS (verification complete, no issues), NOT_APPLICABLE (no test files), SKIPPED (plugin issue), SHALLOW (partial scan)

### Pre-Output Checklist (Mandatory Before Final Output)

Check every item before writing the report. If any item is unchecked, go back and complete it.

- [ ] Every category has a Status value (no empty Status cells)
- [ ] Every category with Status != SKIPPED has an Evidence value
- [ ] All 5 validation categories scanned (or marked NOT_APPLICABLE)
- [ ] Every finding includes file path and line number
- [ ] Legacy-allowance annotations applied where applicable
- [ ] Report language matches the user's conversation language

### Schema Compliance Check (Mandatory Before Saving to .ww-w-ai/)

Before writing to .ww-w-ai/devtools/validate-tests/, verify the JSON output:
- [ ] Every "required" field in schema.json is present and non-empty
- [ ] The findings[] array contains every detected anti-pattern
- [ ] categorySummary counts match the actual finding counts per category

## Output

Generate the validation report in the user's conversation language.

Output detection results as a Markdown table in this format:

```
===== Test Validation Report =====

| File:Line | Type | Description |
|-----------|------|-------------|
| `src/services/auth.test.ts:15` | Vague Name | `test_1` -- cannot identify failure cause |
| `src/services/order.test.ts:42` | Implementation Detail | `.assert_called_once_with` -- internal-call verification |
| `src/services/payment.test.ts:20` | Excessive Mocks | 6 Mocks used -- design review needed |
| `test_user.py:5` | Global State | `shared_cart = Cart()` -- state shared across tests |
| `src/utils/calc.test.ts:30` | DB Dependency | Direct `db.query` call -- unit test depends on DB |

Total anti-patterns: {N}
  Vague Names: {A}
  Implementation Detail: {B}
  Excessive Mocks: {C}
  Global State: {D}
  DB Dependency: {E}

===============================
```

When no anti-patterns are found:

```
===== Test Validation Report =====

No anti-patterns.

===============================
```

## Output Persistence

After generating the test validation report, save results to `.ww-w-ai/devtools/validate-tests/`:

1. Create `.ww-w-ai/devtools/validate-tests/` if missing
2. Write `latest.json` -- structured result following `templates/schema.json`
3. Write `latest.md` -- human-readable report following `templates/report.template.md`
4. Archive to `history/` -- copy latest.json to `.ww-w-ai/devtools/validate-tests/history/{timestamp}.json`

`latest.md` is generated in the user's conversation language. JSON field names stay in English regardless of language.
The JSON output enables machine-parseable history tracking and cross-run comparison.
The history/ directory preserves previous executions for trend analysis.

## Permission Rationale

- **Write**: Exclusively for .ww-w-ai/ output persistence. No project source modification.
- **Bash**: Exclusively for read-only git/system queries. No file modifications.

## Notes

- Bash is permitted for directory creation (`mkdir -p .ww-w-ai/devtools/validate-tests/history`). The validation itself is read-only and does not modify project files.
- "Change-detection tests" in legacy code are temporarily allowed per testing.md exceptions. If the file or directory contains `legacy`, `migration`, or `compat`, annotate the result with "(legacy allowance)."
- Implementation-detail test detection (Category 2) may yield false positives. `.toHaveBeenCalledWith` can also be used for external-dependency verification (e.g., API calls), so results are presented as advisory.
- Excessive-Mocks judgment (Category 3) is performed per function. Mocks set up in `beforeEach`/`setUp` are distributed and tallied against each test.

## Spec Reference

For detailed validation criteria, evidence tables, and examples:
- Corresponding rule spec: `../../docs/specs/testing.md`
