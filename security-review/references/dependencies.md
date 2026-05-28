# Dependency and Code Security

Most of a modern application is code you didn't write — third-party libraries, SDKs, and their transitive dependencies. Each one runs with your app's privileges, so a vulnerable or malicious package is your problem. On top of that, code that lingers (dead endpoints, deprecated features, secrets in git history) keeps an attack surface alive long after anyone's looking at it. This maps to OWASP "Vulnerable and Outdated Components."

## Checklist

- [ ] Dependencies are kept reasonably up to date
- [ ] Dependencies are audited for known vulnerabilities
- [ ] Third-party SDKs are vetted before integrating
- [ ] Dead code, unused endpoints, and deprecated features are removed
- [ ] Secrets are not exposed anywhere in git history

## What to look for

### Known-vulnerable dependencies
Check for an audit step and how stale the lockfile is. Run the ecosystem's auditor:

- Node: `npm audit` / `pnpm audit` / `yarn audit`
- Python: `pip-audit`, `safety`
- Multi-language / CVE + license: `osv-scanner`, Trivy, Snyk, GitHub Dependabot

Flag critical/high advisories, and dependencies pinned to old major versions that no longer get security fixes.

**Fix:** Update vulnerable packages to patched versions. Enable automated dependency updates (Dependabot/Renovate) and a CI audit gate. Commit lockfiles so builds are reproducible.

### Vetting new SDKs/packages
Before a dependency is added, it's worth a sanity check — especially for anything touching auth, crypto, or payments. Watch for typosquatting (a package name one character off a popular one), abandoned/unmaintained packages, very low download counts on something claiming to be popular, and excessive/unexpected install scripts.

**Fix:** Prefer well-maintained, widely-used libraries. Verify the exact package name and publisher. Pin versions and review what postinstall scripts do. Minimize dependency count — each one is trust extended.

### Dead code and unused surface
Flag commented-out blocks, disabled-but-reachable endpoints, old API versions, test/debug routes left in prod, feature flags for removed features, and deprecated auth paths. Unused code is unmonitored code — it doesn't get patched but can still be exploited.

**Fix:** Delete it. Remove unused endpoints and routes, drop deprecated features, and prune unused dependencies (`depcheck`, `pip-autoremove`). If you can't delete yet, ensure it's unreachable and access-controlled.

### Secrets in git history
A secret removed in the latest commit is still sitting in history, where it can be recovered by anyone with the repo. Scan history, not just the working tree.

```bash
# scan committed history for secrets
gitleaks detect --source . --redact
trufflehog git file://. --only-verified
```

**Fix:** **Rotate any exposed secret first** — scrubbing history doesn't un-leak a key that's already been cloned. Then purge it from history with `git filter-repo` or BFG Repo-Cleaner and force-push (coordinate with collaborators, since this rewrites history). Add the file to `.gitignore` and a pre-commit secret scanner going forward. See [secrets-and-config.md](secrets-and-config.md).
