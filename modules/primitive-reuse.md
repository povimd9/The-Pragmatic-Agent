# Module: Never Re-Implement Off-The-Shelf Primitives

**Load when:** you're integrating with an off-the-shelf service (identity provider, OAuth2 server, message broker, search engine, observability stack, etc.) and considering wrapping or mirroring its primitives.

**Core rule lives in:** [`AGENT-INSTRUCTIONS.md` §9.1](../AGENT-INSTRUCTIONS.md#9-code-conventions).

**Silent-plausibility variant fought:** application-layer wrappers around primitives the off-the-shelf service already provides — adds maintenance surface, drifts from upstream behavior, often re-introduces the very security bugs the off-the-shelf service was chosen to avoid.

---

## The Rule

If your project depends on an off-the-shelf service (identity provider, OAuth2 server, message broker, search engine, observability platform, etc.) that already provides a primitive (password hashing, OAuth2 token endpoint, OIDC discovery, JWKS, full-text search, pub/sub topic ACLs, query DSL, etc.), the application **MUST** use that primitive directly. NEVER wrap, mirror, or re-implement it.

## Acceptable Application-Layer Endpoints

Around an off-the-shelf primitive, the only legitimate application-layer surface is:

1. **Flow-orchestration glue** the off-the-shelf service does not natively own (e.g., "kick off the SPA's OAuth dance" entry redirect — the off-the-shelf provider doesn't know your SPA's flow shape).
2. **Application-state mirrors** of off-the-shelf events via the primitive's webhook surface — ONLY when the side-effect mutates application rows the off-the-shelf service cannot mutate itself.
3. **Validation of credentials** issued by the primitive (e.g., JWT signature + claim-shape check; webhook signature verification).

Anything else — re-hashing passwords, re-implementing OIDC discovery, mirroring identity tables, wrapping the primitive's self-service endpoints — is forbidden.

## Forbidden Patterns

- Wrapping the off-the-shelf provider's self-service endpoints behind your own routes "for consistency." (Just call the provider directly. The SPA loads the provider's UI nodes; your app server isn't in the path.)
- Re-implementing the primitive's hashing / TOTP / OAuth2 / OIDC discovery / JWKS surface in your own code.
- Mirroring the provider's identity columns into your application DB as a substitute for reading the provider's admin API at request time, when no documented incompatibility compels the mirror.
- Adding a "thin wrapper" that adds zero behavior beyond the provider's native endpoint but creates a deployment dependency between your service and the provider's URL shape.

## When Mirroring Is Actually Justified

The narrow exception is when the off-the-shelf provider's contract genuinely can't reach your application's needs:

- Webhook auth model is incompatible with your transport security requirements (the provider supports only API-key auth on webhooks but your perimeter requires mTLS).
- The provider's read API is rate-limited too aggressively for the call frequency your app needs.
- A claim needs to live in the JWT but the provider's claim source is a function your code owns (e.g., role derivation that depends on application-side state).

In each case the justification is documented inline + at the spec layer. "It would be more convenient" is never sufficient.

## Plan-Step Check

Every new endpoint that touches the off-the-shelf primitive's domain (auth, identity, search, messaging, etc.) MUST include an explicit **"Native primitive check"** section in the plan listing which off-the-shelf endpoint replaces the proposed wrapper. If a native equivalent exists and is reachable, use it directly; the application-side endpoint is rejected.

## Worked Example

```
Plan section: "Add POST /v1/users/me/password-change"

Native primitive check:
- Off-the-shelf provider exposes /self-service/settings/browser which
  handles password change end-to-end including validation, hashing,
  and audit. SPA can call it directly via the provider's documented
  flow.
- Conclusion: DROP the /v1/users/me/password-change wrapper. SPA
  calls /self-service/settings/browser directly. The only
  application-side glue needed is a webhook handler at
  /internal/provider-hooks/post-password-change that clears the
  application-side must_change_password flag — that's pattern (2)
  above, justified.
```

The wrapper that almost got built was 80 lines of validation + bcrypt + audit code that the provider already implements (correctly, audited, maintained). The 5-line webhook handler is the right shape.

## Why This Rule Compounds

Wrappers around off-the-shelf primitives are the worst kind of technical debt because:

- They drift from the upstream's behavior as the upstream improves (your wrapper's password validation rules age while the provider's improve).
- They reintroduce security bugs the provider has already fixed (your wrapper's bcrypt cost factor was right in 2020 but is too low in 2026).
- They create a fake API surface that consumers (your SPA, your other services) become dependent on, making it harder to swap providers later.
- They mask the actual provider integration, so debugging requires reading your wrapper code AND the provider's docs.

Direct integration costs more upfront (read the provider's docs, understand its flow) but pays off every quarter thereafter.
