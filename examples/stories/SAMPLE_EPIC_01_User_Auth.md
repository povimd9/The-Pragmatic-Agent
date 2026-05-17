# [EPIC] LLD #01.1: Authentication — JWT Middleware & RBAC

> **This is a SAMPLE.** TaskFlow is fictional. All LLD references, line numbers, acceptance-criteria commands, dates, version strings, and team names are illustrative. Replace with your project's actuals before adopting.

**Epic ID:** EPIC_01.1
**File Name:** `EPIC_01.1_JWT_Middleware_RBAC.md`
**Implementation Domain:** Backend
**Assigned Team:** Backend Team
**LLD Reference:** [LLD #01 — Authentication & Authorization](../design/llds/SAMPLE_LLD_01_Authentication.md) (lines 1-280)
**Spec Reference:** [specs/api/REST_ENDPOINTS.md](../../specs/api/REST_ENDPOINTS.md) (section 2.1, lines 15-42)
**PR Scope:** Implement JWT auth middleware, role guard, and tenant context extraction
**Estimated Complexity:** Medium
**Story Count:** 3
**Dependencies:** Keycloak configured and running (EPIC_08.1)

---

## Critical Implementation Principles

_(See [EPIC_TEMPLATE.md](./EPIC_TEMPLATE.md) for the full list. All principles apply.)_

**Key principles for this epic:**
- Auth failures = hard 401/403. NEVER fall back to anonymous or default roles.
- Missing `KEYCLOAK_URL` or `JWT_AUDIENCE` env vars = crash on startup.
- Empty catch blocks in auth code = immediate rejection in review.

---

## Epic Summary

Implement the JWT authentication middleware, role-based access control (RBAC) guard, and tenant context extraction for the TaskFlow API. After this epic, all API endpoints (except `/health`) require a valid JWT, and role-protected routes enforce the RBAC matrix defined in LLD #01 §5.2.

### Scope

**In Scope:**
- JWT verification middleware (RS256, JWKS from Keycloak)
- Tenant ID extraction from JWT claims
- RBAC role guard middleware
- Token blocklist check (Redis)
- Auth-related error responses (401, 403)

**Out of Scope:**
- Keycloak server setup (EPIC_08.1)
- Login/registration UI (EPIC_02.1, Frontend)
- Refresh token rotation (EPIC_01.2)
- Mobile biometric auth (EPIC_06.1, Mobile)

### Success Criteria

- [ ] All API routes except `/health` reject requests without valid JWT
- [ ] JWT verification uses RS256 with JWKS from Keycloak
- [ ] `req.user` contains `user_id`, `tenant_id`, `email`, `role` after auth
- [ ] Role guard rejects unauthorized role access with 403
- [ ] Token blocklist check works via Redis
- [ ] Unit tests >80% coverage on auth middleware
- [ ] Auth middleware adds <5ms latency (p95)

---

## User Stories

### Story 1: JWT Verification Middleware

**As a** backend API
**I want** to verify JWT tokens on every incoming request
**So that** only authenticated users can access protected endpoints

**Acceptance Criteria (with Verification Commands):**

| # | Criterion | Verification Command | Expected Output |
|---|-----------|---------------------|-----------------|
| 1 | Request without Authorization header returns 401 | `curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/v1/tasks` | `401` |
| 2 | Request with invalid JWT returns 401 | `curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer invalid" http://localhost:3000/api/v1/tasks` | `401` |
| 3 | Request with valid JWT returns 200 | `curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $(./scripts/get-test-token.sh)" http://localhost:3000/api/v1/tasks` | `200` |
| 4 | `/health` endpoint accessible without auth | `curl -s http://localhost:3000/health` | `{"status":"ok","version":"0.5.0"}` |
| 5 | Missing KEYCLOAK_URL crashes on startup | `unset KEYCLOAK_URL && npm start 2>&1 \| head -5` | Error: "KEYCLOAK_URL environment variable is required" |

**Implementation Tasks:**
- [ ] Task 1.1: Create `src/middleware/auth.ts` with `extractToken()`, `verifyJWT()`, `attachUser()` functions
- [ ] Task 1.2: Configure `jwks-rsa` client to fetch keys from `KEYCLOAK_URL/realms/{realm}/protocol/openid-connect/certs`
- [ ] Task 1.3: Implement JWKS cache with 1-hour TTL (LLD #01 §4.1)
- [ ] Task 1.4: Add fail-fast config validation on startup for `KEYCLOAK_URL`, `KEYCLOAK_REALM`, `JWT_AUDIENCE`
- [ ] Task 1.5: Register middleware on all routes except `/health`
- [ ] Task 1.6: Write unit tests for all 5 acceptance criteria

**LLD Traceability:**
- Section: §5.1 Middleware Chain
- File: `SAMPLE_LLD_01_Authentication.md` (lines 130-165)
- Specific: extractToken → checkBlocklist → verifyJWT → extractClaims → attachUser chain

**Estimated Effort:** 4 hours

---

### Story 2: Tenant Context Extraction

**As a** backend API
**I want** to extract `tenant_id` from the JWT and set the PostgreSQL RLS context
**So that** every database query is automatically scoped to the current tenant

**Acceptance Criteria (with Verification Commands):**

| # | Criterion | Verification Command | Expected Output |
|---|-----------|---------------------|-----------------|
| 1 | `req.user.tenant_id` is populated after auth | `curl -s -H "Authorization: Bearer $(./scripts/get-test-token.sh)" http://localhost:3000/api/v1/debug/whoami` | JSON with `tenant_id` field |
| 2 | JWT missing `tenant_id` claim returns 401 | `curl -s -w "%{http_code}" -H "Authorization: Bearer $(./scripts/get-test-token.sh --no-tenant)" http://localhost:3000/api/v1/tasks` | `401` |
| 3 | PostgreSQL `app.current_tenant` is set per request | `npm run test:integration -- --grep "tenant context"` | All tenant context tests pass |
| 4 | Tenant A cannot see Tenant B's tasks | `npm run test:integration -- --grep "cross-tenant"` | Cross-tenant access tests pass (data isolated) |

**Implementation Tasks:**
- [ ] Task 2.1: Create `src/middleware/tenant.ts` that extracts `tenant_id` from `req.user` and calls `SET app.current_tenant`
- [ ] Task 2.2: Add tenant middleware after auth middleware in the chain
- [ ] Task 2.3: Write integration test: create task as tenant A, verify invisible to tenant B
- [ ] Task 2.4: Write integration test: JWT without `tenant_id` claim → 401

**LLD Traceability:**
- Section: §5.1 Middleware Chain (extractClaims step)
- File: `SAMPLE_LLD_01_Authentication.md` (lines 155-165)
- Also: §9 Invariant Compliance — #1 Tenant Isolation

**Estimated Effort:** 3 hours

---

### Story 3: RBAC Role Guard

**As a** backend API
**I want** to enforce role-based access control on protected routes
**So that** users can only perform actions their role permits

**Acceptance Criteria (with Verification Commands):**

| # | Criterion | Verification Command | Expected Output |
|---|-----------|---------------------|-----------------|
| 1 | Member can create tasks | `curl -s -o /dev/null -w "%{http_code}" -X POST -H "Authorization: Bearer $(./scripts/get-test-token.sh --role member)" -H "Content-Type: application/json" -d '{"title":"Test"}' http://localhost:3000/api/v1/tasks` | `201` |
| 2 | Guest cannot create tasks | `curl -s -o /dev/null -w "%{http_code}" -X POST -H "Authorization: Bearer $(./scripts/get-test-token.sh --role guest)" -H "Content-Type: application/json" -d '{"title":"Test"}' http://localhost:3000/api/v1/tasks` | `403` |
| 3 | Member cannot delete projects | `curl -s -o /dev/null -w "%{http_code}" -X DELETE -H "Authorization: Bearer $(./scripts/get-test-token.sh --role member)" http://localhost:3000/api/v1/projects/test-id` | `403` |
| 4 | Owner can delete projects | `curl -s -o /dev/null -w "%{http_code}" -X DELETE -H "Authorization: Bearer $(./scripts/get-test-token.sh --role owner)" http://localhost:3000/api/v1/projects/test-id` | `200` or `204` |

**Implementation Tasks:**
- [ ] Task 3.1: Create `src/middleware/rbac.ts` with `requireRole(...roles)` middleware factory
- [ ] Task 3.2: Apply role guards to all routes per RBAC matrix (LLD #01 §5.2)
- [ ] Task 3.3: Write unit tests for each role × resource × action combination
- [ ] Task 3.4: Create `scripts/get-test-token.sh` helper for generating test JWTs with specific roles

**LLD Traceability:**
- Section: §5.2 RBAC Permission Matrix
- File: `SAMPLE_LLD_01_Authentication.md` (lines 170-192)
- Specific: Guest/Member/Admin/Owner permissions per resource

**Estimated Effort:** 3 hours

---

## Technical Notes

### Architecture Decisions

**Decision:** Use middleware-based auth (not per-query).
**Rationale:** Centralizes auth logic, prevents accidental unprotected routes. Per LLD #01 §5.1.

**Decision:** Fail-closed on Redis failure (reject request, don't skip blocklist).
**Rationale:** Accepting potentially revoked tokens is a security risk. Per LLD #01 §7.2.

### Interface Contracts

Per LLD #01 §3.1, auth middleware attaches to `req.user`:
```typescript
{
  user_id: string;    // JWT sub claim
  tenant_id: string;  // JWT tenant_id claim
  email: string;      // JWT email claim
  role: TenantRole;   // JWT tenant_role claim
}
```

### Performance Requirements

Per LLD #01 §6.1:
- Full auth middleware: <5ms (p95)
- JWT verification with cached JWKS: <2ms (p95)

---

## Testing Strategy

### Unit Tests
- [ ] Auth middleware: all 5 Story 1 acceptance criteria
- [ ] Role guard: all role × resource combinations from LLD #01 §5.2
- [ ] Config validation: crash on missing env vars

### Integration Tests
- [ ] Cross-tenant isolation: tenant A data invisible to tenant B
- [ ] Token blocklist: revoked token rejected
- [ ] Full request flow: auth → tenant context → route handler → response

### Validation Criteria

| Metric | Pass Criteria |
|--------|--------------|
| Auth middleware latency (p95) | < 5ms |
| Unit test coverage | > 80% |
| Cross-tenant isolation | 100% (zero leaks) |

---

## Observability

### Metrics
- `auth.requests.total{status}`: Count of auth outcomes (success, invalid, expired, forbidden)
- `auth.latency.ms`: Histogram of auth middleware duration

### Logs
- `auth.rejected` (WARN): Invalid or expired token
- `auth.forbidden` (WARN): Valid token, insufficient role
- `auth.jwks.error` (ERROR): JWKS fetch failure

---

## Dependencies & Blockers

**Upstream Dependencies:**
- [ ] EPIC_08.1: Keycloak configured and running with test realm

**External Dependencies:**
- [ ] jsonwebtoken: 9.0.2 (per VERSIONS.yaml)
- [ ] jwks-rsa: 3.1.0 (per VERSIONS.yaml)
- [ ] ioredis: 5.3.2 (for blocklist check)

**Known Blockers:**
- None. Keycloak setup (EPIC_08.1) is complete.

---

## Definition of Done

### PRE-COMPLETION CHECKLIST (MANDATORY)

- [ ] **Runtime Verification:**
  - [ ] `npm run dev` starts without errors
  - [ ] All 5 Story 1 curl commands return expected status codes
  - [ ] `npm run test:integration` passes cross-tenant tests

- [ ] **Placeholder Scan Passed:**
  ```bash
  grep -rn "placeholder\|TODO\|FIXME" src/middleware/ --include="*.ts"
  ```
  **Result:** _(empty)_

- [ ] **Fail-Fast Violation Scan Passed:**
  ```bash
  grep -rn "catch.*{}" src/middleware/ --include="*.ts"
  grep -rn '|| ""' src/middleware/ --include="*.ts"
  ```
  **Result:** _(empty, or each match reviewed with user)_

- [ ] **All Verification Commands Executed:**
  | Criterion | Command | Result |
  |-----------|---------|--------|
  | Story 1, AC 1 | `curl -s -o /dev/null -w "%{http_code}" ...` | 401 |
  | Story 1, AC 2 | `curl -s -o /dev/null -w "%{http_code}" ...` | 401 |
  | ... | ... | ... |

### Standard Definition of Done

- [ ] All 3 stories completed with acceptance criteria met
- [ ] All verification commands executed with proof recorded
- [ ] Spec compliance verified (REST endpoint paths match spec)
- [ ] Unit tests pass with >80% coverage on `src/middleware/`
- [ ] Integration tests pass (cross-tenant, blocklist)
- [ ] Auth middleware latency <5ms (p95)
- [ ] Merged to main branch

### Session Scope Limits

**Stories 1-2 in first session. Story 3 in second session.**

---

## References

- **LLD:** [LLD #01 — Authentication & Authorization](../design/llds/SAMPLE_LLD_01_Authentication.md)
- **HLD:** [HLD v1.0.0](../design/SAMPLE_HLD.md) §5.1, §5.2, §9
- **Related Epics:** EPIC_08.1 (Keycloak setup), EPIC_01.2 (Refresh token rotation), EPIC_02.1 (Login UI)
