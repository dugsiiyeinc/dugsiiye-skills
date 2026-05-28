# Secrets and Configuration Security

Secrets — API keys, tokens, passwords, signing keys, connection strings — are the keys to the kingdom. A single leaked production key can mean attacker-issued charges, full database access, or account takeover. They leak through code, git history, logs, error pages, and frontend bundles. Configuration mistakes (debug mode, permissive CORS, default credentials) widen the blast radius.

## Checklist

- [ ] No hardcoded API keys, tokens, or passwords in source code
- [ ] No secrets written to logs or returned in error messages
- [ ] `.env` (and other secret files) are gitignored and never committed
- [ ] No server-side / secret keys embedded in frontend code
- [ ] CORS is restricted to known origins, not wide open
- [ ] Dependencies have been audited for known vulnerabilities
- [ ] Default credentials are removed or changed
- [ ] Debug mode is off in production

## What to look for

### Hardcoded secrets
Grep the codebase for tell-tale prefixes and assignments: `sk_live_`, `sk_test_`, `AKIA` (AWS), `AIza` (Google), `ghp_` (GitHub), `xoxb-` (Slack), `Bearer `, `password =`, `api_key =`, `secret =`, `PRIVATE KEY`. Any literal credential in code is a finding even if the repo is currently private — repos get forked, leaked, or made public.

**Fix:** Load secrets from environment variables or a secrets manager (AWS Secrets Manager, Vault, Doppler, platform env vars). Commit a `.env.example` with empty placeholder keys, never real values. **If a real secret was ever committed, rotating it is mandatory** — removing it from the latest commit does nothing, because it's still in history and may already be scraped.

### Secrets in logs and errors
Look for `console.log`, `print`, `logger.info` that dump request bodies, headers, tokens, or full user objects. Look for error handlers that return stack traces or config dumps to the client.

**Fix:** Redact sensitive fields before logging. Return generic error messages to clients; log details server-side only.

### `.env` in git
Check `.gitignore` includes `.env`, `.env.*`, `*.pem`, `*.key`, credentials files. Run `git log --all --full-history -- .env` to see if it was ever committed.

**Fix:** Add to `.gitignore`. If already committed, rotate every secret in it and scrub history (`git filter-repo` / BFG) — see [dependencies.md](dependencies.md) for git-history scrubbing.

### Server-side keys in frontend code
Anything in a frontend bundle (`src/`, React/Vue/Angular, anything served to the browser) is fully public. A common mistake: putting a secret key behind a `REACT_APP_`/`VITE_`/`NEXT_PUBLIC_` env var — these get inlined into the client bundle.

**Fix:** Only publishable/public keys belong in frontend code. Secret keys must stay on a backend the browser calls. Proxy privileged operations through your server.

### CORS
Flag `Access-Control-Allow-Origin: *`, `origin: '*'`, or reflecting the request's `Origin` header back unconditionally — especially when combined with `credentials: true` (browsers forbid `*` with credentials, but reflected-origin + credentials is a common dangerous workaround that effectively allows any site to make authenticated requests).

**Fix:** Maintain an allowlist of known origins. Only enable credentials for origins you control.

### Default credentials
Look for `admin/admin`, `root` with no password, seeded default users, default DB or dashboard passwords, sample JWT secrets like `"secret"` or `"changeme"`.

**Fix:** Remove default accounts; require strong unique credentials at setup; generate signing secrets randomly.

### Debug mode in production
Flag `DEBUG = True` (Django), `app.debug = True`/`FLASK_ENV=development` (Flask), `NODE_ENV` not set to `production`, framework debug toolbars, source maps shipped to prod. Debug mode leaks stack traces, env vars, and sometimes an interactive console.

**Fix:** Gate debug strictly to local/dev via environment; ensure production config sets it off.
