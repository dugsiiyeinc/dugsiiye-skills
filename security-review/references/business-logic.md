# Business Logic Security

Business-logic flaws aren't classic injection bugs — the code does exactly what it was written to do, but the *rules* can be abused. They're dangerous precisely because automated scanners miss them and they often map directly to money: paying less, granting yourself a refund, skipping a verification step. The root cause is almost always **trusting the client** to enforce a rule that only the server can safely enforce.

## Checklist

- [ ] Pricing, discounts, and payment amounts are validated server-side
- [ ] Critical workflows are enforced on the backend, not assumed from client state
- [ ] Client-side logic is treated as manipulable (it runs in the user's browser)
- [ ] Sensitive actions (email change, account deletion, payment confirmation) require extra verification

## What to look for

### Client-supplied prices and amounts
The cardinal sin of e-commerce code: trusting a price, total, discount, or quantity sent from the client.

```javascript
// VULNERABLE: client sends the price, attacker sets it to 0.01
const total = req.body.price * req.body.quantity;
charge(req.body.total);

// FIXED: server looks up the authoritative price and computes the total
const item = await db.items.findById(req.body.itemId);
const total = item.price * clampQuantity(req.body.quantity);
```

An attacker opens devtools or replays the request and changes `price`, `total`, `currency`, or `discount`. Always **High/Critical**.

**Fix:** The server is the source of truth for prices, taxes, fees, and totals. Look up prices from the database by product ID. Validate coupon codes server-side (existence, validity window, per-user/global usage limits, stacking rules). Recompute the total server-side and verify it against the payment processor's confirmed amount.

### Workflow / state enforcement
Flag multi-step flows (checkout, KYC, onboarding) where the server trusts the client to say "I already completed step 2." An attacker can skip steps or jump straight to the final action.

**Fix:** Track workflow state server-side and verify each step's preconditions on the server before allowing the next. Don't rely on hidden form fields, client flags, or step order in the UI.

### Manipulable client logic
Any check that lives only in JavaScript — "you must be 18," "max 5 per customer," "members only" — is advisory. The user controls the browser and can bypass it entirely.

**Fix:** Mirror every security- or money-relevant rule on the server. Client-side checks are for UX (instant feedback), never enforcement. See also [authorization.md](authorization.md).

### Sensitive actions need step-up verification
Account-takeover-enabling or destructive actions — changing email/password, disabling MFA, deleting the account, confirming a payment, changing payout details — should require re-authentication or a confirmation step, so a hijacked session or CSRF can't trigger them silently.

**Fix:** Require the current password (or a fresh MFA challenge) for these actions. Send a confirmation/notification to the *old* email when email changes. Add CSRF protection to state-changing requests. Apply rate limits to high-value operations.

### Race conditions on value operations
Flag value-bearing actions that can be fired concurrently — redeeming a one-time coupon, withdrawing a balance, claiming limited stock — without atomicity. Parallel requests can double-spend.

**Fix:** Use atomic DB operations, row locking, or idempotency keys so the same value can't be claimed twice.
