# Authorization and Access Control

Authentication is "who are you"; authorization is "what are you allowed to do." Broken access control is consistently the #1 risk in the OWASP Top 10 because it's so easy to get subtly wrong — the user is logged in, so it *feels* secure, but nothing stops them from reaching into another user's data.

## Checklist

- [ ] A user cannot access resources they don't own
- [ ] No IDOR — changing an ID in a request can't expose someone else's data
- [ ] Authorization is enforced on the backend, not just hidden in the frontend
- [ ] Roles and permissions (RBAC) are checked correctly
- [ ] Admin endpoints require both authentication and authorization
- [ ] "Hard to guess" URLs are not treated as a security control

## What to look for

### Ownership checks (IDOR / BOLA)
This is the most common and most damaging finding. Look for endpoints that take an object ID — `GET /api/orders/123`, `GET /users/456/profile`, `DELETE /documents/789` — and fetch by that ID **without checking that the object belongs to the current user**.

```python
# VULNERABLE: any logged-in user can read any order
order = Order.objects.get(id=request.GET["id"])

# FIXED: scope the query to the authenticated user
order = Order.objects.get(id=request.GET["id"], owner=request.user)
```

Test it mentally: if user A changes the ID to user B's, do they get B's data? If yes, it's an IDOR (a.k.a. Broken Object Level Authorization). Severity is typically **High**, **Critical** if it exposes bulk/sensitive data or allows modification.

**Fix:** Always scope queries to the authenticated principal, or explicitly verify ownership after fetching. Don't trust any ID supplied by the client.

### Backend enforcement vs. frontend hiding
Flag authorization that exists only in the UI — hiding an "Admin" button, a route guard in React, a disabled field. The API behind it must independently enforce the rule, because an attacker calls the API directly (curl, Postman, devtools), never your UI.

**Fix:** Enforce every permission check server-side on the endpoint that performs the action. Treat the frontend as a convenience, never a gate.

### RBAC correctness
Where roles exist, check that the role is verified on the server for each protected action, that role is read from a trusted server-side source (not a client-supplied header or editable JWT claim without verification), and that there's no privilege escalation (a user setting their own `role=admin` in a profile update).

**Fix:** Centralize permission checks in middleware/decorators. Never let users write their own role/permission fields. Default-deny: unknown or missing role gets no access.

### Admin endpoints
Administrative routes (`/admin`, `/internal`, management APIs, debug endpoints) must require authentication *and* an admin authorization check. Flag any that rely on obscurity or are reachable by any logged-in user.

**Fix:** Put admin routes behind an explicit admin-role check, and ideally network restrictions (VPN/allowlist) too.

### Hidden URLs are not security
Unguessable or undocumented URLs (`/api/v2/_internal/export?token=...` with no real auth) get found — in logs, browser history, referrer headers, JS bundles, and crawlers.

**Fix:** Every sensitive endpoint needs a real authentication + authorization check regardless of how obscure its path is.
