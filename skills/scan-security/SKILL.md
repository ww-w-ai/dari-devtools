---
name: scan-security
description: Detects security vulnerabilities via static pattern matching based on OWASP Top 10
triggers:
  - scan security
  - security scan
  - check security
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Glob
  - Grep
  - Bash
---

# scan-security -- OWASP-Based Security Vulnerability Detection

Corresponding rule: `security.md`

## Purpose

Detect security vulnerabilities in code via static pattern matching based on OWASP Top 10. Does not auto-fix; reports only suspected items.

## Input -- Dynamic Target Resolution

Resolve target directories in this order:
1. CLAUDE.md "Project Structure" section -- extract source code directories
2. If CLAUDE.md is missing or has no structure info: Glob source files (`**/*.{ts,tsx,js,jsx,py,java,go,rs}`) -- use the top-level directory containing matched files
3. If no source files are found: report status as SHALLOW and state the reason

Default exclusions: node_modules/, .git/, dist/, build/, coverage/
Skill-specific exclusions: `*.test.*`, `*.spec.*`, `test_*`, `*_test.*` (security test code excluded)

### Mandatory Output: Target Resolution Result

Output the following before proceeding with validation. Do not advance until resolution is complete.

| Step | Result |
|------|--------|
| CLAUDE.md check | Found/Not found |
| Resolved directories | {list or "none"} |
| Source file count | {count} |
| Resolution method | CLAUDE.md / Glob fallback / SHALLOW |

## Validation Categories

### 1. Hardcoded Secrets

Detect passwords, API keys, and tokens embedded directly in code.

Search patterns:
- `API_KEY\s*=\s*["']`
- `password\s*=\s*["']`
- `secret\s*=\s*["']`
- `token\s*=\s*["']`
- `PRIVATE_KEY\s*=\s*["']`
- `aws_access_key_id\s*=\s*["']`
- `aws_secret_access_key\s*=\s*["']`

Severity: **CRITICAL**

### 2. SQL Injection

Detect patterns that build SQL queries via string interpolation.

Search patterns:
- `f"SELECT`
- `f"INSERT`
- `f"UPDATE`
- `f"DELETE`
- `f"DROP`
- `` `SELECT.*\$\{` `` (SQL in template literals)
- `"SELECT.*" \+` (SQL via string concatenation)

Severity: **CRITICAL**

### 3. XSS (Cross-Site Scripting)

Detect patterns that inject user input into HTML without escaping.

Search patterns:
- `innerHTML\s*=`
- `dangerouslySetInnerHTML`
- `v-html=`
- `\{!!.*!!\}` (Blade unescaped output)
- `\|safe` (Django/Jinja safe filter)

Severity: **HIGH**

### 4. Command Injection

Detect patterns that pass external input directly to shell commands.

Search patterns:
- `os.system(f"`
- `subprocess.*shell=True`
- `exec(` (dynamic code execution)
- `eval(` (dynamic code execution)
- `` child_process.exec(`.* `` (Node.js)

Severity: **CRITICAL**

### 5. Error Information Exposure

Detect patterns that include internal system information in user responses.

Search patterns:
- `traceback.format_exc()` -- near return/response
- `str(e)` -- included directly in the response object
- `stack.*trace` -- included in the response
- `exc_info=True` -- inside a user-response function

Severity: **MEDIUM**

### 6. .gitignore Check

Verify that secret files are included in .gitignore.

Required patterns (must exist in .gitignore):
- `.env` related: `.env`, `.env.*`, `.env.local`
- Credential files: `credentials`, `credentials.*`
- Key files: `*.pem`, `*.key`, `*.p12`, `*.pfx`
- Secret directories: `secrets/`, `.secrets/`

Severity: **HIGH** when missing

## Execution Logic

1. `Glob` to collect target source files (excluding test files)
2. Categories 1-5: `Grep` each category's patterns across all source files
3. Category 6: `Read` the `.gitignore` file and check for required patterns
4. Sort detected items by severity and output as a table

### Mandatory Output: Security Scan Verification Matrix

Output the matrix below before generating the final report. Do not proceed until every category has been scanned.

| Category | Status | Items Checked | Findings | Severity | Evidence |
|----------|:------:|:-------------:|:--------:|:--------:|----------|
| Hardcoded Secrets | ? | ? | ? | CRITICAL | {tool, files, pattern} |
| SQL Injection | ? | ? | ? | CRITICAL | {tool, files, pattern} |
| XSS | ? | ? | ? | HIGH | {tool, files, pattern} |
| Command Injection | ? | ? | ? | CRITICAL | {tool, files, pattern} |
| Error Information Exposure | ? | ? | ? | MEDIUM | {tool, files, pattern} |
| .gitignore | ? | ? | ? | HIGH | {tool, files, pattern} |

Status values: PASS (verification complete, no issues), NOT_APPLICABLE (no source files), SKIPPED (plugin issue), SHALLOW (target resolution failed)

### Pre-Output Checklist (Mandatory Before Final Output)

Check every item before writing the report. If any item is unchecked, go back and complete it.

- [ ] Every validation category has a Status value (no empty Status cells)
- [ ] Every category with Status != SKIPPED has an Evidence value
- [ ] Every finding includes file path, line number (where applicable), and matched pattern
- [ ] The .gitignore check has verified all required patterns
- [ ] Report language matches the user's conversation language

### Schema Compliance Check (Mandatory Before Saving to .ww-w-ai/)

Before writing to .ww-w-ai/devtools/scan-security/, verify the JSON output:
- [ ] Every "required" field in schema.json is present and non-empty
- [ ] The findings[] array contains every detected item
- [ ] severitySummary counts match the actual finding counts per severity

## Output

Generate the scan report in the user's conversation language.

Output detection results as a Markdown table in this format:

```
===== Security Scan Report =====

| File:Line | Type | Severity | Pattern |
|-----------|------|----------|---------|
| `src/config/db.ts:15` | Hardcoded Secret | CRITICAL | `password = "admin123"` |
| `src/api/users.ts:42` | SQL Injection | CRITICAL | `f"SELECT * FROM users WHERE id = {id}"` |
| `src/components/Post.tsx:18` | XSS | HIGH | `dangerouslySetInnerHTML` |
| `.gitignore` | Unprotected Secret | HIGH | `.env` pattern missing |
| `src/handlers/error.ts:30` | Error Information Exposure | MEDIUM | `str(e)` returned directly |

Total suspected items: {N}
  CRITICAL: {C}
  HIGH: {H}
  MEDIUM: {M}

==============================
```

When there are no suspected items:

```
===== Security Scan Report =====

No suspected items.

==============================
```

## Output Persistence

After generating the security scan report, save results to `.ww-w-ai/devtools/scan-security/`:

1. Create `.ww-w-ai/devtools/scan-security/` if missing
2. Write `latest.json` -- structured result following `templates/schema.json`
3. Write `latest.md` -- human-readable report following `templates/report.template.md`
4. Archive to `history/` -- copy latest.json to `.ww-w-ai/devtools/scan-security/history/{timestamp}.json`

`latest.md` is generated in the user's conversation language. JSON field names stay in English regardless of language.
The JSON output enables machine-parseable history tracking and cross-run comparison.
The history/ directory preserves previous executions for trend analysis.

## Permission Rationale

- **Write**: Exclusively for .ww-w-ai/ output persistence. No project source modification.
- **Bash**: Exclusively for read-only git/system queries. No file modifications.

## Notes

- Bash is permitted for directory creation (`mkdir -p .ww-w-ai/devtools/scan-security/history`). The scan itself is read-only and does not modify project files.
- Based on static pattern matching, so **false positives are possible**. Results are presented as "suspected items"; developers must confirm whether real vulnerabilities exist.
- Environment variable assignments (`os.environ["API_KEY"]`) are not hardcoding and are excluded from detection.
- Security test code (code bearing a `SECURITY` prefix comment) is treated as intentionally vulnerable per security.md exceptions.

## Spec Reference

For detailed validation criteria, evidence tables, and examples:
- Corresponding rule spec: `../../docs/specs/security.md`
