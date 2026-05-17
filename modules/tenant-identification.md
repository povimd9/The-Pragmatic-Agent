# Module: Never Trust Client-Supplied Tenant / Account Identifiers

**Load when:** you're writing or reviewing a multi-tenant request handler (any code that decides which tenant's data to read or write).

**Core rule lives in:** [`AGENT-INSTRUCTIONS.md` §7.1](../AGENT-INSTRUCTIONS.md#7-security).

**Silent-plausibility variant fought:** handlers that look like they enforce tenant isolation but trust the client to identify itself, opening an OWASP A01 (broken access control) vector.

---

## The Rule

The tenant or account identifier **MUST** come from the validated JWT / session claim, NEVER from query strings, request bodies, headers, or path parameters (except where a path parameter is then explicitly validated against the JWT before use).

Persistence-layer code (DB connection key, S3 prefix, Redis prefix, message-bus topic) **MUST** derive from the JWT-validated identifier. The path/body param is allowed only as a cross-check input ("does the JWT's `account_ids` claim contain this path's `:account_id`?") — never as the authoritative source.

## Examples

```go
// FORBIDDEN — connection string built from path param
pathID := c.Param("account_id")
conn := persist.ConnFor(pathID)       // OWASP A01 — caller chooses the tenant

// CORRECT — JWT is the truth; path is cross-checked
jwtClaims := auth.ClaimsFrom(c)
pathID := c.Param("account_id")
if !jwtClaims.HasAccount(pathID) {
    audit.LogCrossAccountAttempt(c, jwtClaims.Sub, pathID)
    c.JSON(403, ErrCrossAccount)
    return
}
conn := persist.ConnFor(jwtClaims.PrimaryAccount(pathID))
```

## Cross-Account Access Logging

Every cross-account access ATTEMPT — even rejected — is logged for audit. Silent rejections hide enumeration attacks: an attacker scanning a sequence of account IDs gets the same 403 for each, but without audit you can't tell whether one user tried 5 IDs or 5,000.

The audit row records: timestamp, requesting user (from JWT sub), attempted account ID (from path/body), source IP, requested resource. Audit tables are retention-protected — never truncate, never roll over without explicit operator approval.

## Defense-in-Depth Layers

The tenant-identifier rule sits at the **handler** layer. Two more layers reinforce it:

1. **Database RLS (Row-Level Security).** Even if a handler bug let a wrong tenant ID through, RLS enforces the constraint at the query level — `SET app.current_tenant = ?` before every query, RLS policies on every tenant-scoped table.
2. **Schema-level tenant scoping.** S3 keys include tenant prefix (`tenants/{tenant_id}/...`); Redis keys include tenant prefix; message-bus topics are namespaced by tenant. A handler bug + a missing RLS policy still can't cross tenants if the storage layer rejects the operation.

The handler-layer rule is the cheapest enforcement; RLS + storage-scoping are the catch-net.

## Common Pattern: Admin Endpoints Touching Multiple Tenants

Admin endpoints that legitimately touch multiple tenants (e.g., a billing dashboard for the platform operator) still derive the SET of accessible tenants from the JWT — never from a request param. The JWT's `roles` or `classes` claim names what the user can do; the JWT's `account_ids` or equivalent names which tenants. Crossing the two yields the effective access list. Path params on admin endpoints are SCOPE NARROWING within that list, not authorization in their own right.

## Test the Failure Path

The cross-tenant access test is mandatory: spin up two tenants, authenticate as tenant A, attempt to access tenant B's resource by manipulating the path param, verify the response is 403 AND the audit row exists. If the test passes (i.e., the cross-tenant attempt succeeds), the handler has a tenant-bypass bug — fix before merging.
