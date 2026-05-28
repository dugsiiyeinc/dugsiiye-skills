---
name: security-review
description: Review application code and configuration for security vulnerabilities before deployment. Use this skill whenever the user asks to "review my code for security", "is this secure", "check this before I deploy", "do a security audit", "find vulnerabilities", "harden this", or raises a specific concern like exposed API keys or secrets, hardcoded credentials, weak or broken authentication, session handling, IDOR or broken access control, overly permissive CORS, SQL injection, XSS, unsafe file uploads, missing input validation, client-side-only business logic (pricing/discounts/payments), or risky third-party dependencies. Trigger it even when the user doesn't say the word "security" but is clearly about to ship code, share a repo, or asks "what could go wrong here?" about a backend, API, auth flow, or web app.
---

# Security Review

Review application code and configuration for security issues *before* it ships, when fixes are cheap and a breach hasn't happened yet. This skill gives you a structured checklist across the areas where real-world applications most often get compromised, so the review is thorough instead of ad hoc.

The checklist maps to widely used industry standards — the **OWASP Top 10**, the **Secure Software Development Lifecycle (SSDLC)**, **API security best practices**, and **DevSecOps** principles. You don't need to recite these to the user, but they're why the categories below are the right ones to check.

## Mindset

Treat every input as hostile, every client as untrusted, and every secret as something that will eventually leak if it can. The attacker controls the request, the browser, the network, and anything the frontend touches — so any security control that lives only on the client is decorative. Your job is to find the gap between what the developer *assumed* and what an attacker can *actually do*, and to point it out with a fix concrete enough to act on.

## Workflow

Follow these four steps. Don't skip step 1 — reviewing the wrong thing wastes everyone's time.

### 1. Establish scope

If it's not already obvious from the conversation, ask briefly what you're reviewing before diving in:

- **What is it?** A backend API, a frontend app, an auth flow, config/infra files, a full repo?
- **What's the stack?** Language, framework, database, hosting. This changes what the common pitfalls are.
- **What's sensitive?** Does it handle payments, personal data, authentication, file uploads, admin actions?
- **How far along?** A pre-deploy gate, a periodic audit, or a specific worry the user already has.

If the user already pointed you at specific files or a specific concern, scope to that and skip the interrogation.

### 2. Walk the relevant categories

Go through the nine categories below. Skip the ones that don't apply (e.g., no file uploads, no need to review upload handling), but say so rather than silently dropping them — a skipped category the user didn't realize applied is itself a finding. For anything you're genuinely unsure about, open the matching `references/` file for deeper checks, patterns, and code-level examples before concluding.

Read the actual code where you can. Grep for the obvious smells — hardcoded keys, `eval`, raw string-concatenated SQL, `cors({ origin: '*' })`, `debug = True`, `password ==`, missing auth decorators. Don't just pattern-match, though; trace how data and trust flow through the request.

### 3. Report findings with severity and a concrete fix

For each issue, give the user:

- **Severity** — Critical / High / Medium / Low (see rubric below)
- **Location** — file and line where possible (`auth/login.py:42`)
- **What's wrong** — the vulnerability and, briefly, how it could be exploited
- **The fix** — a specific, actionable suggestion (ideally a code snippet or config change), not "validate your inputs"

Use this severity rubric so ratings are consistent:

| Severity | Meaning |
|----------|---------|
| **Critical** | Directly exploitable now for full compromise or mass data exposure — e.g., hardcoded production secret in a public repo, auth that can be trivially bypassed, SQL injection on an unauthenticated endpoint. Fix before any deploy. |
| **High** | Exploitable with modest effort or under common conditions — e.g., IDOR exposing other users' data, missing authz on an admin endpoint, plaintext password storage. |
| **Medium** | Real weakness that needs a precondition or has limited blast radius — e.g., missing rate limiting, verbose error messages leaking internals, weak session expiry. |
| **Low** | Hardening and defense-in-depth — e.g., missing security headers, slightly over-broad CORS on a low-value endpoint, dead code. |

Lead with the worst issues. If you found nothing in a category, say so — "no issues found in X" is useful signal and tells the user the area was actually checked. Don't invent findings to look thorough; a clean review honestly reported builds more trust than padding.

### 4. Point to the deeper reference

When a finding (or the user's follow-up) needs more than a one-line fix, point them to the matching file in `references/`. Each one has detailed checks, exploitation context, and remediation patterns for that category.

## The nine categories

| # | Category | What it covers | Reference |
|---|----------|----------------|-----------|
| 1 | **Secrets & configuration** | Hardcoded keys/tokens/passwords, secrets in logs/errors, `.env` in git, server-side keys in frontend code, CORS, debug mode in prod, default credentials. | [secrets-and-config.md](references/secrets-and-config.md) |
| 2 | **Authentication & sessions** | Password hashing (bcrypt/Argon2), rate limiting & lockout, reset-link expiry, logout invalidation, unpredictable session IDs, idle expiration, MFA. | [authentication.md](references/authentication.md) |
| 3 | **Authorization & access control** | Ownership checks, IDOR, backend-enforced authz (not just UI), RBAC, protected admin endpoints, "hidden URL ≠ security". | [authorization.md](references/authorization.md) |
| 4 | **API security** | Request validation (type/format/length/structure), auth middleware on sensitive routes, rate limiting, errors that don't leak internals, minimal data in responses. | [api-security.md](references/api-security.md) |
| 5 | **Input validation** | All input untrusted, sanitize before DB queries, SQL injection, XSS on user-generated content, validating params/forms/bodies against expected formats. | [input-validation.md](references/input-validation.md) |
| 6 | **File uploads** | File type & MIME validation, size limits, filename sanitization / path traversal, not serving raw uploads from public dirs, malware scanning. | [file-uploads.md](references/file-uploads.md) |
| 7 | **Business logic** | Pricing/discount/payment validated server-side, client logic is manipulable, critical workflows enforced on the backend, extra verification for sensitive actions. | [business-logic.md](references/business-logic.md) |
| 8 | **Dependencies & code** | Updated dependencies, audited SDKs, removing dead code/unused endpoints/deprecated features, secrets scrubbed from git history. | [dependencies.md](references/dependencies.md) |
| 9 | **Security by design** | Building security in from the start rather than bolting it on; why AI-accelerated code still needs a security mindset. | [security-by-design.md](references/security-by-design.md) |

## Report template

Use a structure like this so findings are scannable:

```markdown
# Security Review — <what was reviewed>

**Scope:** <files / components / stack reviewed>
**Date:** <date>

## Summary
<one-paragraph overview: overall posture and the headline risks>

## Findings

### [CRITICAL] Hardcoded Stripe secret key in frontend bundle
- **Location:** `src/config.js:12`
- **Issue:** The live `sk_live_…` key is embedded in client-side JS shipped to every visitor. Anyone can extract it from the bundle and issue charges/refunds as you.
- **Fix:** Move the key server-side, read it from an environment variable, and rotate the exposed key immediately. The browser should only ever see the publishable (`pk_…`) key.

### [HIGH] IDOR on GET /api/orders/:id
- ...

## Checked, no issues found
- Input validation on the signup form
- File upload size limits

## Skipped (not applicable)
- File upload security — no upload functionality in this codebase
```

Always end with the most important next actions, ordered by severity, so the user knows what to fix first.
