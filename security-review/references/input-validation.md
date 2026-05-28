# User Input Validation

The foundational rule of application security: **all input is untrusted until proven otherwise.** Query params, form fields, JSON bodies, headers, cookies, file names, webhook payloads — anything that crosses the trust boundary can be hostile. Most injection attacks (SQLi, XSS, command injection) are failures to validate or properly handle input. This maps to OWASP "Injection."

## Checklist

- [ ] All input is treated as untrusted
- [ ] Input is sanitized/parameterized before reaching database queries
- [ ] SQL injection is prevented (parameterized queries / ORM, never string concatenation)
- [ ] User-generated content is sanitized/escaped to prevent XSS
- [ ] Query params, form inputs, and request bodies are validated against expected formats

## What to look for

### SQL injection
The classic. Flag any query built by concatenating or interpolating user input into SQL:

```python
# VULNERABLE
cursor.execute("SELECT * FROM users WHERE email = '" + email + "'")
db.query(f"SELECT * FROM orders WHERE id = {order_id}")

# FIXED: parameterized query — the driver separates code from data
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
```

The same applies to NoSQL (object-injection in Mongo queries), ORMs used with raw fragments, and `ORDER BY`/column names that can't be parameterized.

**Fix:** Use parameterized/prepared statements or an ORM's safe query builder for *all* queries. For identifiers that can't be bound (table/column/sort), validate against a strict allowlist. Never concatenate user input into a query string.

### Other injection
Watch for user input flowing into shell commands (`os.system`, `exec`, `subprocess` with `shell=True`), `eval`/`Function()`, template engines (SSTI), LDAP, XPath, file paths (see [file-uploads.md](file-uploads.md)), and OS-level calls.

**Fix:** Avoid shelling out with user input; use library APIs and pass arguments as arrays, not a shell string. Never `eval` user input.

### Cross-site scripting (XSS)
Flag user-controlled content rendered into HTML without escaping: `innerHTML = userInput`, `dangerouslySetInnerHTML`, `v-html`, `|safe` / `mark_safe`, template autoescaping turned off, building DOM from untrusted strings. Stored XSS (saved then shown to others) is especially damaging.

**Fix:** Rely on the framework's automatic output escaping; render untrusted content as text, not HTML. When HTML *must* be allowed (rich text), sanitize with a vetted allowlist library (DOMPurify, `bleach`, sanitize-html). Add a Content-Security-Policy as defense-in-depth. Escape based on context (HTML body vs attribute vs JS vs URL).

### Format and structure validation
Beyond injection, validate that input *matches what you expect* — email looks like an email, age is a positive integer in range, enums are within allowed values, strings are length-bounded, arrays aren't unboundedly large. This blocks malformed-data bugs and many DoS vectors.

**Fix:** Validate with a schema library at the boundary (Zod, Joi, Pydantic, DRF serializers). Prefer allowlists ("must be one of these") over blocklists ("must not contain these") — blocklists are routinely bypassed. Validate on the server even if the client already does; client validation is for UX, not security.
