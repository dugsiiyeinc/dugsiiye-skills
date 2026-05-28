# API Security

APIs are the front door to your data and logic. Unlike a rendered web page, they're called directly by clients you don't control, so every endpoint is an attack surface. This category aligns with the OWASP API Security Top 10.

## Checklist

- [ ] All incoming requests are validated (type, format, length, structure)
- [ ] Sensitive endpoints sit behind authentication middleware
- [ ] Public endpoints have rate limiting
- [ ] Error messages don't leak internals (SQL, stack traces, file paths, versions)
- [ ] Responses return only the minimum required data

## What to look for

### Request validation
Flag handlers that read `request.body` / query params and use them directly without schema validation. Missing validation enables injection, type-confusion bugs, oversized payloads (DoS), and mass-assignment.

**Fix:** Validate every request against an explicit schema — type, required fields, allowed values, length/range bounds, and *reject unknown fields*. Use a schema library (Zod, Joi, Pydantic, `class-validator`, JSON Schema, Django/DRF serializers). Validate at the boundary before any business logic runs. See [input-validation.md](input-validation.md) for content-level sanitization.

### Auth middleware on sensitive routes
Map which endpoints require auth and confirm middleware actually covers them. Common mistakes: a route registered outside the protected group, a new endpoint that forgot the decorator, an auth check that runs but whose result is ignored.

**Fix:** Default-deny at the router level — require auth globally and explicitly opt specific routes into "public," rather than the reverse. Audit that each sensitive route is inside the protected group.

### Rate limiting
Public and unauthenticated endpoints (login, signup, password reset, search, anything expensive) need rate limiting to resist brute force, scraping, and abuse.

**Fix:** Apply per-IP and per-account limits (gateway, framework middleware, or a library). Tighten limits on auth and costly endpoints; consider quotas for authenticated API keys.

### Error messages that leak internals
Flag responses that echo raw exceptions, SQL errors, stack traces, file paths, framework/version banners, or "user not found" vs "wrong password" distinctions. Each detail helps an attacker map your system or enumerate accounts.

**Fix:** Return generic, stable error messages and appropriate status codes to clients; log the detail server-side with a correlation ID. Make auth failures indistinguishable (don't reveal which field was wrong). Disable verbose/debug error pages in production — see [secrets-and-config.md](secrets-and-config.md).

### Excessive data exposure
Flag endpoints that serialize whole DB models and return them — including fields like `password_hash`, `is_admin`, internal flags, other users' info, or PII the client doesn't need. Returning extra data ("the frontend just ignores it") leaks it to anyone reading the response.

**Fix:** Use explicit response schemas / serializers that whitelist fields. Return the minimum the client needs. Paginate and filter list endpoints so a single call can't dump the whole table.
