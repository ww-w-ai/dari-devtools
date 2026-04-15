---
name: validate-architecture
description: Detects inter-layer import rule violations (Clean Architecture)
triggers:
  - validate architecture
  - check architecture
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Glob
  - Grep
  - Bash
---

# validate-architecture -- Architecture Layer Violation Detection

Corresponding rule: `architecture.md`

## Purpose

Statically detect inter-layer import rule violations and circular references in a project. If CLAUDE.md defines an architecture, use that layer definition; otherwise default to the general 3-layer model (presentation/business/data access).

## Input -- Dynamic Target Resolution

Resolve target directories in this order:
1. CLAUDE.md "Project Structure" section -- extract source code directories
2. If CLAUDE.md is missing or has no structure info: Glob source files (`**/*.{ts,tsx,js,jsx,py,java,go,rs}`) -- use the top-level directory containing matched files
3. If no source files are found: report status as SHALLOW and state the reason

Default exclusions: node_modules/, .git/, dist/, build/, coverage/
Skill-specific exclusions: `*.test.*`, `*.spec.*`, `test_*`, `*_test.*`

### Mandatory Output: Target Resolution Result

Output the following before proceeding with validation. Do not advance until resolution is complete.

| Step | Result |
|------|--------|
| CLAUDE.md check | Found/Not found |
| Resolved directories | {list or "none"} |
| Source file count | {count} |
| Resolution method | CLAUDE.md / Glob fallback / SHALLOW |

## Execution Logic

1. **Read the architecture definition from CLAUDE.md**
   - Use `Read` to read the project root CLAUDE.md
   - If an Architecture section exists, use its layer structure and dependency direction as the baseline
   - If no Architecture section exists, default to the general 3-layer model:
     - Presentation (presentation, ui, pages, components, views, routes, handlers)
     - Business (domain, core, application, services, use-cases, usecases)
     - Data access (infrastructure, infra, repositories, external, adapters, gateways)
   - Dependency direction: presentation -> business <- data access. The business layer must not depend on other layers.

2. **Collect source files**
   - Use `Glob` to collect files matching `src/**/*.{ts,tsx,js,jsx,py,java}`
   - Exclude test files (`*.test.*`, `*.spec.*`, `test_*`, `*_test.*`)

3. **Collect import/from statements**
   - Use `Grep` to extract import statements from each file
   - Patterns: `import .* from`, `from .* import`, `require(`, `@import`

4. **Detect layer violation patterns**
   - domain/core -> infrastructure/infra imports (business depends on data access)
   - domain/core -> presentation/ui/pages/components imports (business depends on presentation)
   - presentation -> direct DB imports: the presentation layer directly imports DB-related packages (`sqlalchemy`, `prisma`, `typeorm`, `drizzle`, `mongoose`, `sequelize`, `knex`, `pg`, `mysql`, `sqlite`, etc.)

5. **Detect circular references**
   - Detect patterns where module A imports module B while module B also imports module A
   - Check bidirectional imports at the file level

### Mandatory Output: Architecture Validation Matrix

Output the matrix below before generating the final report. Do not proceed until every check has been performed.

| Check Item | Status | Items Checked | Violations | Severity | Evidence |
|------------|:------:|:-------------:|:----------:|:--------:|----------|
| Read CLAUDE.md architecture | ? | ? | -- | -- | {Read result} |
| Source file collection | ? | ? | -- | -- | {Glob pattern, file count} |
| domain -> infra import | ? | ? | ? | ERROR | {Grep pattern, searched files} |
| domain -> presentation import | ? | ? | ? | ERROR | {Grep pattern, searched files} |
| presentation -> DB import | ? | ? | ? | ERROR | {Grep pattern, searched files} |
| Circular references | ? | ? | ? | WARN | {Grep pattern, searched files} |

Status values: PASS (verification complete, no issues), NOT_APPLICABLE (no source files), SKIPPED (plugin issue), SHALLOW (target resolution failed)

### Pre-Output Checklist (Mandatory Before Final Output)

Check every item before writing the report. If any item is unchecked, go back and complete it.

- [ ] Every check item has a Status value (no empty Status cells)
- [ ] Every check with Status != SKIPPED has an Evidence value
- [ ] Architecture basis (CLAUDE.md or general 3-layer) is identified and stated
- [ ] All 4 violation-pattern checks performed
- [ ] Circular-reference check performed
- [ ] Report language matches the user's conversation language

### Schema Compliance Check (Mandatory Before Saving to .ww-w-ai/)

Before writing to .ww-w-ai/devtools/validate-architecture/, verify the JSON output:
- [ ] Every "required" field in schema.json is present and non-empty
- [ ] The findings[] array contains every detected violation
- [ ] summary counts match the actual finding count

## Output

Generate the validation report in the user's conversation language.

Output violations as a Markdown table in this format:

```
===== Architecture Validation Report =====

[Architecture basis]: CLAUDE.md definition / General 3-layer

| File | Violation | Severity |
|------|-----------|----------|
| `src/domain/user.ts` | Infrastructure layer import (`../../infra/db`) | ERROR |
| `src/core/order.ts` | Presentation layer import (`../../pages/OrderView`) | ERROR |
| `src/pages/Dashboard.tsx` | Direct DB import (`prisma`) | ERROR |
| `src/services/auth.ts` <-> `src/services/user.ts` | Circular reference | WARN |

Total violations: {N} (ERROR: {E}, WARN: {W})
====================================
```

When there are no violations:

```
===== Architecture Validation Report =====

[Architecture basis]: CLAUDE.md definition / General 3-layer

No violations.
====================================
```

### Severity Criteria

| Severity | Condition |
|:--------:|-----------|
| ERROR | Layer dependency-direction violation (domain -> infra, domain -> presentation, presentation -> DB) |
| WARN | Circular reference, ambiguous layer placement (file outside any layer directory) |

## Output Persistence

After generating the architecture validation report, save results to `.ww-w-ai/devtools/validate-architecture/`:

1. Create `.ww-w-ai/devtools/validate-architecture/` if missing
2. Write `latest.json` -- structured result following `templates/schema.json`
3. Write `latest.md` -- human-readable report following `templates/report.template.md`
4. Archive to `history/` -- copy latest.json to `.ww-w-ai/devtools/validate-architecture/history/{timestamp}.json`

`latest.md` is generated in the user's conversation language. JSON field names stay in English regardless of language.
The JSON output enables machine-parseable history tracking and cross-run comparison.
The history/ directory preserves previous executions for trend analysis.

## Permission Rationale

- **Write**: Exclusively for .ww-w-ai/ output persistence. No project source modification.
- **Bash**: Exclusively for read-only git/system queries. No file modifications.

## Notes

- Bash is permitted for directory creation (`mkdir -p .ww-w-ai/devtools/validate-architecture/history`). The validation itself is read-only and does not modify project files.
- Test files (`*.test.*`, `*.spec.*`) are excluded from inspection (per testing.md exceptions, internal access inside tests is permitted).
- If CLAUDE.md defines an architecture, that definition takes precedence over the general principles.
- Layer directory naming may differ per project, so CLAUDE.md definitions take highest precedence.

## Spec Reference

For detailed validation criteria, evidence tables, and examples:
- Corresponding rule spec: `../../docs/specs/architecture.md`
