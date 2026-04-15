# Security Principles -- Detailed Spec

> WHY: Without security constraints, Claude generates code containing OWASP Top 10 vulnerabilities by default.
> It injects user input into SQL without validation, hardcodes API keys in source code,
> and exposes stack traces directly in error responses.
> Without this rule, production deployment leads to data leaks, service outages, and legal liability.

## SOURCE

| Evidence ID | Basis | Type | Source |
|-------------|-------|------|--------|
| E-IS-007 | Input validation, output escaping, secret management, dependency security | INDUSTRY_STD | OWASP Top 10 (2021) |
| E-CC-006 | Blocking `.env`/secret file access via the CC settings deny list | CC_OFFICIAL | Claude Code official docs -- Permissions |
| E-AD-003 | Security boundaries and least-privilege principles for AI agent tool use | ANTHROPIC_DOCS | Anthropic Security Guidelines |

## Scope

- paths: all source code (especially code that handles external input or sensitive data)
- target: user input handling, data output, authentication/authorization, error responses, secret management, package dependencies

## 1. All External Input Must Be Validated Before Use

All data coming from users, APIs, and files is validated at the system entry point.

A whitelist (allow-list) approach takes priority. "Allow only what is permitted" is safer than "block only what is forbidden."
Validate type, range, and format, and pass only validated data to business logic.

### CONSTRAINT

- Without input validation: immediate exposure to injection attacks such as SQL injection, path traversal, and command injection
- Blacklist-only: easily bypassed via case mixing, Unicode encoding, or double encoding -- defenseless against new attack patterns

### Good

```python
# Whitelist validation -- only permitted values pass
ALLOWED_SORT_FIELDS = {"name", "created_at", "price"}

def get_products(sort_by):
    if sort_by not in ALLOWED_SORT_FIELDS:
        raise ValidationError(f"Disallowed sort field: {sort_by}")
    return repository.find_all(order_by=sort_by)

# Type + range validation combined
def set_page_size(size):
    if not isinstance(size, int) or not (1 <= size <= 100):
        raise ValidationError("page_size must be an integer between 1 and 100")
    return size
```

### Bad

```python
# Inject input directly into the query with no validation
def get_products(sort_by):
    return db.query(f"SELECT * FROM products ORDER BY {sort_by}")

# Blacklist only -- easily bypassed
def sanitize(user_input):
    blocked = ["DROP", "DELETE", "UPDATE"]
    for word in blocked:
        user_input = user_input.replace(word, "")
    return user_input
```

-- Putting `name; DROP TABLE products--` into `sort_by` drops the table. The blacklist is bypassed with `DrOp`.

## 2. Apply Context-Appropriate Escaping on Output

When injecting data into SQL, HTML, command lines, or URLs, use the safe method for that context instead of string concatenation.

Never mix data and code (queries, markup, shell commands) via string concatenation.

| Output Context | Safe Method |
|----------------|-------------|
| SQL | Parameter binding (`?`, `$1`, `:name`) |
| HTML | Template engine auto-escaping |
| Shell command | Argument array (`shell=False`) |
| URL | URL encoding (`encodeURIComponent`, etc.) |

### CONSTRAINT

- Building SQL via string concatenation: SQL injection -- full data leak, deletion, or admin privilege escalation
- Omitting HTML escaping: XSS -- session cookie theft, malicious script execution, phishing page injection

### Good

```python
# SQL -- parameter binding
result = db.query("SELECT * FROM users WHERE email = ?", [user_email])

# HTML -- template engine performs auto-escaping
template.render(username=user_input)

# Shell command -- pass as argument array, bypasses shell interpretation
subprocess.run(["convert", input_path, output_path])
```

### Bad

```python
# SQL -- built via string concatenation
result = db.query(f"SELECT * FROM users WHERE email = '{user_email}'")

# HTML -- user input injected directly
page = f"<h1>Welcome, {username}!</h1>"

# Shell -- command built as a string
os.system(f"convert {input_path} {output_path}")
```

-- Putting `' OR 1=1 --` into `user_email` leaks the entire users table. Putting `<script>document.cookie</script>` into `username` hijacks the session.

## 3. Do Not Put Secrets in Code

API keys, passwords, tokens, and certificates must not be hardcoded in source code.

Inject them at runtime through environment variables or a secret manager.
Register files containing secrets (`.env`, `credentials.*`, `*.pem`, `*.key`) in `.gitignore`.

### CONSTRAINT

- Hardcoded secrets in code: permanently recorded in Git history -- `git rm` cannot remove them, key rotation is the only remedy
- Not registered in `.gitignore`: secret files get pushed to the remote repository and publicly exposed -- automated scanners exploit them within minutes

### Good

```python
# Injected via environment variables
import os

api_key = os.environ["PAYMENT_API_KEY"]
db_url = os.environ["DATABASE_URL"]

# Register secret file patterns in .gitignore
# .env
# .env.*
# credentials/
# *.pem
# *.key
```

### Bad

```python
# Hardcoded directly in source code
API_KEY = "sk-1234567890abcdef"
DB_URL = "postgresql://admin:P@ssw0rd@db.prod.example.com/main"

# Real key used in test code
def test_payment():
    client = PaymentClient(api_key="sk-live-actual-production-key")
```

-- The moment it is pushed to Git, the key is permanently recorded in history. The `sk-live-actual-production-key` in test code also lives in history and is leaked.

## 4. Do Not Expose Internal Information in Error Responses

Stack traces, DB queries, internal file paths, and server versions are recorded in the logging system only.

Return only generic messages that do not reveal specific causes to users.

### CONSTRAINT

- Responses containing stack traces: hand attackers the internal file structure, framework version, and code flow for free
- Responses containing DB queries: table and column names are exposed, directly aiding precise targeting for SQL injection attacks

### Good

```python
# Internal logging -- record detailed information
try:
    receipt = process_payment(order)
except PaymentError as e:
    logger.error(f"Payment failed: order_id={order.id}, error={e}", exc_info=True)
    # User response -- generic message only
    return error_response(
        status=500,
        message="An error occurred while processing payment. Please try again later."
    )
```

### Bad

```python
# Include all internal system information in the error response
try:
    result = db.query("SELECT * FROM payments WHERE order_id = ?", [oid])
except Exception as e:
    return {
        "error": str(e),
        "query": f"SELECT * FROM payments WHERE order_id = {oid}",
        "traceback": traceback.format_exc(),
        "server": "Python 3.12 / PostgreSQL 16.2"
    }
```

-- DB query, table structure, server version, and file paths are all included in the response. Directly usable in an attacker's reconnaissance phase.

## 5. Manage Dependency Security Vulnerabilities

Do not leave packages with known vulnerabilities unpatched.

Run a security audit when adding new packages, and periodically scan existing dependencies in the CI pipeline.
Commit the lock file to guarantee identical versions across environments.

| Timing | Action |
|--------|--------|
| Right after package install | Run audit: `npm audit` / `pip audit` / `cargo audit`, etc. |
| CI/CD pipeline | Automated audit -- fail the build when vulnerabilities are detected |
| On vulnerability disclosure | Update to a patched version or switch to an alternative package |

### CONSTRAINT

- Leaving vulnerable packages unpatched: automated scanners detect published CVEs and exploit them within hours -- attacks are already known, leaving no buffer for defense
- Not committing the lock file: different versions get installed per environment, causing "works on my machine" problems and non-reproducible builds

### Good

```bash
# Run an audit after adding a package
pnpm install some-package
pnpm audit

# Include an audit step in the CI pipeline (fail the build on vulnerabilities)
# Commit the lock file to pin versions
# package-lock.json / pnpm-lock.yaml / Pipfile.lock, etc.
```

### Bad

```bash
# Ignore vulnerability warnings and deploy
# "Will fix it next sprint" -> still unfixed six months later

# Register the lock file in .gitignore
# -> Different versions get installed per environment; vulnerable versions ship to production

# Specify versions with wildcards
# "some-package": "*" -> major updates apply without notice, breaking compatibility
```

-- Leaving a package with a known CVE unpatched for six months lets automated attack tools exploit it immediately. Without the lock file, you cannot even track which version was deployed.

## Exceptions

- Development-only tools (`devDependencies`): not included in the production bundle, so security audits have lower priority
- Internal-network-only services: input validation levels may be relaxed when there is no external input (but consider insider threats)
- Prototype/PoC: secret management rules are not exempt -- hardcoded keys in a PoC still live permanently in Git history
- Security test code: intentional vulnerability creation is permitted. Only inside test directories, with a mandatory `SECURITY` prefix comment

## DEPENDS

- None
