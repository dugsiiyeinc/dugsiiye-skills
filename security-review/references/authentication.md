# Authentication and Session Security

Authentication proves *who* a user is; sessions keep them logged in afterward. Weaknesses here lead to account takeover, credential stuffing, and session hijacking — and they affect every user at once. This maps to OWASP's "Identification and Authentication Failures."

## Checklist

- [ ] Passwords are never stored in plaintext
- [ ] Passwords are hashed with bcrypt or Argon2 (not MD5/SHA-1/SHA-256-alone)
- [ ] Login has rate limiting and account lockout against brute force
- [ ] Password reset links expire (and are single-use)
- [ ] Sessions are invalidated on logout
- [ ] Session IDs / tokens are long and unpredictable
- [ ] Inactive sessions expire
- [ ] MFA is available for sensitive or privileged accounts

## What to look for

### Password storage
Flag any plaintext storage, reversible encryption, or fast/general-purpose hashes (`md5`, `sha1`, `sha256`, `hashlib.sha256`) used directly for passwords. Fast hashes are brute-forceable at billions/sec on a GPU.

**Fix:** Use a slow, salted password hash designed for the job — **bcrypt** or **Argon2id** (also acceptable: scrypt, PBKDF2 with a high iteration count). Most frameworks ship this (`bcrypt`, `argon2-cffi`, Django's `make_password`, `passlib`). Never roll your own.

### Brute force and credential stuffing
Check the login endpoint for rate limiting (per IP and per account) and lockout/backoff after repeated failures. Absence means attackers can try millions of passwords.

**Fix:** Add rate limiting (e.g., 5–10 attempts/min/account), exponential backoff or temporary lockout, and a CAPTCHA or step-up challenge after repeated failures. Consider checking passwords against known-breached lists.

### Password reset flow
Reset tokens should be cryptographically random, expire quickly (e.g., 15–60 min), be single-use, and be invalidated once the password changes. Flag tokens that are guessable (sequential IDs, user email, timestamps), never expire, or remain valid after use.

**Fix:** Generate a high-entropy random token, store its hash, set a short expiry, invalidate on use and on password change. Don't reveal whether an email exists ("if that address exists, we sent a link").

### Logout and session invalidation
After logout, the session/token must be unusable. Flag flows that only delete a client-side cookie but leave a server session valid, or stateless JWTs with no revocation path.

**Fix:** Destroy the server-side session on logout. For JWTs, keep tokens short-lived and maintain a revocation/denylist or rotate a server-side secret for true invalidation.

### Session ID predictability
Session identifiers and tokens must be long and generated from a CSPRNG. Flag sequential IDs, timestamps, or anything derived from user data.

**Fix:** Use the framework's session machinery (it uses secure randomness). Set cookies `HttpOnly`, `Secure`, and `SameSite`. Regenerate the session ID on login to prevent session fixation.

### Idle and absolute expiration
Sessions that never expire let a stolen token work forever.

**Fix:** Enforce an idle timeout (e.g., 30 min for sensitive apps) and an absolute maximum lifetime, then require re-auth.

### MFA
For admin, financial, or otherwise privileged accounts, single-factor auth is a high risk.

**Fix:** Offer TOTP/WebAuthn MFA; require it for privileged roles and sensitive actions. Prefer app-based or hardware factors over SMS where feasible.
