# Security by Design

Security works best when it's part of how software is built, not a patch applied after a scare. "Bolted-on" security — added under deadline pressure after the architecture is fixed — tends to be incomplete and brittle, because the design already assumed trust in the wrong places. Building it in from the start (the core idea of the Secure Software Development Lifecycle, SSDLC, and DevSecOps) is cheaper and far more effective. This file is less a checklist and more the mindset behind the other eight categories.

## Principles to evaluate against

- **Threat-model early.** Before building, ask: who would attack this, what do they want, where's the trust boundary? A login form, a payment flow, and a public API have very different threat profiles. Design controls around the answers.

- **Default deny / least privilege.** Access, permissions, network exposure, and database grants should start closed and open only where needed. A service that talks only to one table shouldn't have a god-mode DB user. See [authorization.md](authorization.md).

- **Defense in depth.** Don't rely on a single control. Validate input *and* parameterize queries *and* escape output. If one layer fails, the next still holds.

- **Fail securely.** When something errors, the safe state is denied access and a generic message — not an open door or a stack trace. See [api-security.md](api-security.md).

- **Trust boundaries are explicit.** Be clear about where untrusted data enters (the client, third parties, uploads) and validate at that boundary. The browser is always outside the boundary. See [input-validation.md](input-validation.md) and [business-logic.md](business-logic.md).

- **Secure defaults.** The out-of-the-box configuration should be the safe one — HTTPS on, debug off, secrets externalized, CORS restricted — so that doing nothing special still yields a secure deployment. See [secrets-and-config.md](secrets-and-config.md).

- **Minimize attack surface.** Every endpoint, dependency, feature flag, and stored field is something to defend. Less code and fewer dependencies mean less to attack. See [dependencies.md](dependencies.md).

## The AI-accelerated coding caveat

AI coding assistants (including this one) let people ship working software faster than ever — often faster than their security knowledge grows. Generated code optimizes for "it works," and a feature can look complete and pass its happy-path tests while quietly missing an ownership check, validating input only on the client, or hardcoding a key. The speed is real and valuable; the gap is that nobody paused to think like an attacker.

So when reviewing AI-generated or rapidly-built code, apply *more* scrutiny to exactly the areas a generator tends to skip: authorization on every endpoint, server-side enforcement of business rules, secret handling, and input validation at the boundary. Velocity is not a substitute for a security mindset — it raises the stakes of not having one. The goal isn't to slow people down; it's to make the secure path the easy, default one so that fast and safe stop being in tension.

## How to use this in a review

When you finish walking the eight concrete categories, step back and ask the design-level questions: Is security assumed somewhere it shouldn't be? Is there a single point of failure? Does the architecture make the insecure thing the easy thing? Surface those as findings too — they're often the root cause behind several of the specific issues you found.
