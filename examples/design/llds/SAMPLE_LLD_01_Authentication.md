# LLD #01: Authentication & Authorization

> **This is a SAMPLE.** TaskFlow is fictional. All dates, version strings, owner names, library versions, and field-level details are illustrative. Replace with your project's actuals before adopting.

**Version:** 1.0.0
**Status:** Approved
**Last Updated:** 2025-03-15
**Owner:** Backend Team
**HLD Reference:** §5.1 Authentication & Authorization

---

## 1. Purpose and Scope

### What This Component Implements
The authentication and authorization layer for the TaskFlow API. Handles JWT validation, tenant context extraction, role-based access control (RBAC), and session management via Keycloak OIDC.

### What It Does NOT Implement
- User registration UI (owned by Web Frontend)
- Keycloak server configuration (owned by Platform Team, see LLD #08)
- Mobile biometric auth (owned by Mobile Team, see LLD #06)

### HLD References
- §5.1 Authentication & Authorization — OIDC flow, token format, RBAC model
- §5.2 Multi-Tenancy — tenant_id extraction from JWT claims
- §9 Critical Invariants — #2 "Auth Required" invariant

---

## 2. Context and Dependencies

### Upstream Dependencies

| Component | What It Provides | Interface |
|-----------|-----------------|-----------|
| Keycloak | OIDC tokens, JWKS endpoint, user management | OIDC / REST API |
| API Gateway (ALB) | TLS termination, request routing | HTTPS |

### Downstream Consumers

| Component | What It Consumes | Interface |
|-----------|-----------------|-----------|
| All API Routes | Authenticated user context (`req.user`) | Express middleware |
| Tenant Middleware | `tenant_id` from validated JWT | Express middleware chain |
| WebSocket Server | Auth token for connection handshake | Socket.io middleware |
| Audit Logger | `user_id`, `tenant_id` for audit entries | Function call |

### External Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| jsonwebtoken | 9.0.2 | JWT verification (RS256) |
| jwks-rsa | 3.1.0 | JWKS key fetching from Keycloak |
| express | 4.18.3 | HTTP middleware framework |

---

## 3. Interfaces and Contracts

### 3.1 Auth Middleware Interface

```typescript
// Attaches to Express request after successful auth
interface AuthenticatedRequest extends Request {
  user: {
    user_id: string;       // Keycloak subject (sub claim)
    tenant_id: string;     // From custom JWT claim "tenant_id"
    email: string;         // User email
    role: TenantRole;      // "owner" | "admin" | "member" | "guest"
  };
}
```

### 3.2 OIDC Token Claims (Expected)

```json
{
  "sub": "user-uuid",
  "email": "user@example.com",
  "tenant_id": "tenant-uuid",
  "tenant_role": "admin",
  "iss": "https://auth.taskflow.example.com/realms/taskflow",
  "aud": "taskflow-api",
  "exp": 1710500000,
  "iat": 1710499100
}
```

### 3.3 Error Responses

| Condition | HTTP Status | Response Body |
|-----------|-------------|---------------|
| No Authorization header | 401 | `{"error": "missing_token", "message": "Authorization header required"}` |
| Invalid/expired JWT | 401 | `{"error": "invalid_token", "message": "Token validation failed"}` |
| Missing tenant_id claim | 401 | `{"error": "invalid_token", "message": "Token missing tenant_id claim"}` |
| Insufficient role | 403 | `{"error": "forbidden", "message": "Requires role: admin"}` |

### 3.4 Error Model

All auth errors return immediately — no fallbacks, no default roles, no anonymous access. A missing or invalid token is a hard 401. A missing role is a hard 403.

---

## 4. Data Models and Persistence

### 4.1 JWKS Cache

JWKS keys are cached in-memory with a 1-hour TTL to avoid hitting Keycloak on every request.

```typescript
interface JWKSCache {
  keys: JsonWebKey[];
  fetched_at: Date;
  ttl_ms: 3600000; // 1 hour
}
```

### 4.2 Session Blocklist (Redis)

Revoked tokens are tracked in Redis until their natural expiry:

```
Key:    blocklist:jwt:{jti}
Value:  "revoked"
TTL:    remaining token lifetime (max 15 minutes)
```

### 4.3 No Persistent Database Tables

Auth state lives in Keycloak and JWT tokens. This component does not own any PostgreSQL tables.

---

## 5. Detailed Design

### 5.1 Middleware Chain

```
Incoming Request
    │
    ▼
extractToken()
    │ Extract Bearer token from Authorization header
    │ FAIL → 401 "missing_token"
    ▼
checkBlocklist()
    │ Check Redis blocklist for revoked JTI
    │ FOUND → 401 "token_revoked"
    ▼
verifyJWT()
    │ Verify signature using JWKS (RS256)
    │ Check exp, iss, aud claims
    │ FAIL → 401 "invalid_token"
    ▼
extractClaims()
    │ Extract user_id, tenant_id, email, role
    │ MISSING tenant_id → 401 "invalid_token"
    ▼
attachUser()
    │ Set req.user with validated claims
    ▼
Next Middleware (Tenant Context → Route Handler)
```

### 5.2 RBAC Permission Matrix

| Resource | Action | Guest | Member | Admin | Owner |
|----------|--------|-------|--------|-------|-------|
| Tasks | Read (own project) | ✅ | ✅ | ✅ | ✅ |
| Tasks | Create | ❌ | ✅ | ✅ | ✅ |
| Tasks | Update (any) | ❌ | ❌ | ✅ | ✅ |
| Tasks | Delete | ❌ | ❌ | ✅ | ✅ |
| Projects | Create | ❌ | ❌ | ✅ | ✅ |
| Projects | Delete | ❌ | ❌ | ❌ | ✅ |
| Members | Invite | ❌ | ❌ | ✅ | ✅ |
| Members | Remove | ❌ | ❌ | ✅ | ✅ |
| Tenant | Settings | ❌ | ❌ | ❌ | ✅ |

### 5.3 Role Guard Middleware

```typescript
// Per taskflow-specs/api/REST_ENDPOINTS.md:
// Role enforcement is middleware-based, not per-query
function requireRole(...roles: TenantRole[]) {
  return (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    if (!roles.includes(req.user.role)) {
      res.status(403).json({
        error: "forbidden",
        message: `Requires role: ${roles.join(" or ")}`,
      });
      return;
    }
    next();
  };
}

// Usage:
router.delete("/tasks/:id", requireRole("admin", "owner"), deleteTask);
```

### 5.4 Configuration

| Config Key | Type | Default | Description |
|------------|------|---------|-------------|
| `KEYCLOAK_URL` | string | REQUIRED | Keycloak base URL |
| `KEYCLOAK_REALM` | string | REQUIRED | OIDC realm name |
| `JWT_AUDIENCE` | string | REQUIRED | Expected `aud` claim |
| `JWKS_CACHE_TTL_MS` | number | 3600000 | JWKS cache duration |
| `TOKEN_BLOCKLIST_ENABLED` | boolean | true | Enable Redis blocklist |

All `REQUIRED` configs crash on startup if missing. No defaults.

---

## 6. Timing and Performance

### 6.1 Latency Budgets

| Operation | Target (p50) | Target (p95) | Target (p99) |
|-----------|-------------|-------------|-------------|
| JWT verification (cached JWKS) | < 1ms | < 2ms | < 5ms |
| JWKS fetch (cache miss) | < 50ms | < 100ms | < 200ms |
| Redis blocklist check | < 1ms | < 2ms | < 5ms |
| Full auth middleware | < 3ms | < 5ms | < 10ms |

### 6.2 Throughput

| Metric | Steady State | Peak |
|--------|-------------|------|
| Auth validations/sec | 200 | 1,000 |
| JWKS fetches/hour | 1 (cache hit) | 5 (key rotation) |
| Blocklist checks/sec | 200 | 1,000 |

---

## 7. Reliability and Failure Handling

### 7.1 Failure Modes

| Failure | Detection | Impact | Mitigation |
|---------|-----------|--------|------------|
| Keycloak down | JWKS fetch fails | New keys can't be fetched | Cached JWKS used until TTL; alarm after 2 failures |
| Redis down | Blocklist check fails | Revoked tokens not caught | Fail-open disabled; fail-closed: reject request |
| JWKS key rotation | Old key ID not found | JWT verification fails | Fetch fresh JWKS on key-not-found (max 1 retry) |
| Clock skew | JWT exp/iat validation fails | Valid tokens rejected | NTP sync required; 30s clock tolerance |

### 7.2 Retry and Backoff

- **JWKS fetch:** 1 retry with 500ms delay. If both fail, use cached keys. If no cache, reject request (503).
- **Redis blocklist:** No retry. On Redis failure, reject the request (503) rather than skipping the check.
- **JWT verification:** No retry. Invalid = 401.

---

## 8. Security and Privacy

### 8.1 Token Security

- RS256 signatures (asymmetric — API never holds the private key)
- 15-minute access token expiry
- Refresh tokens: 7-day expiry, single-use, rotated on each refresh
- JTI (JWT ID) tracked for revocation

### 8.2 Data Classification

| Data Element | Classification | Handling |
|-------------|---------------|----------|
| JWT token | Confidential | Never logged; transmitted only over TLS |
| user_id | Internal | Logged for audit; not exposed to other tenants |
| email | PII | Logged redacted (`u***@example.com`) |
| tenant_id | Internal | Logged for audit; used for RLS context |

---

## 9. Invariant Compliance

| Invariant (HLD §9) | How This Component Complies | Verification |
|---------------------|---------------------------|--------------|
| #2 Auth Required | Middleware rejects all unauthenticated requests before route handlers run | Integration test: request without token → 401 |
| #1 Tenant Isolation | Extracts tenant_id from JWT, passes to tenant middleware for RLS | Integration test: token for tenant A cannot access tenant B data |
| #4 Audit Trail | Provides user_id and tenant_id to audit logger | Unit test: req.user populated after auth |

---

## 10. Observability

### Metrics

| Metric Name | Type | Labels | Alert Threshold |
|-------------|------|--------|----------------|
| `auth.requests.total` | Counter | `status={success,invalid,expired,forbidden}` | N/A |
| `auth.latency.ms` | Histogram | `stage={verify,blocklist,total}` | p99 > 10ms |
| `auth.jwks.fetches` | Counter | `result={hit,miss,error}` | error > 3/hour |
| `auth.blocklist.size` | Gauge | — | > 10,000 (unusual) |

### Logs

| Event | Level | When Emitted |
|-------|-------|-------------|
| `auth.success` | DEBUG | Every successful auth (high volume — sample in prod) |
| `auth.rejected` | WARN | Invalid/expired token |
| `auth.forbidden` | WARN | Valid token, insufficient role |
| `auth.jwks.refresh` | INFO | JWKS cache refreshed |
| `auth.jwks.error` | ERROR | JWKS fetch failed |

### Health Checks

| Check | Endpoint | Criteria |
|-------|----------|----------|
| JWKS reachable | Internal | Keycloak JWKS endpoint responds within 5s |
| Redis connected | Internal | Redis PING succeeds within 100ms |

---

## 11. Validation and Acceptance

### Test Cases

| Test | Input | Expected Output | Type |
|------|-------|-----------------|------|
| Valid token | Valid JWT with all claims | req.user populated, next() called | Unit |
| Expired token | JWT with exp in past | 401 "invalid_token" | Unit |
| Missing tenant_id | JWT without tenant_id claim | 401 "invalid_token" | Unit |
| Wrong audience | JWT with aud != "taskflow-api" | 401 "invalid_token" | Unit |
| Revoked token | JWT with JTI in blocklist | 401 "token_revoked" | Integration |
| Role guard: allowed | Admin accessing admin route | next() called | Unit |
| Role guard: denied | Member accessing admin route | 403 "forbidden" | Unit |
| Cross-tenant access | Token for tenant A, request tenant B data | No data returned (RLS) | Integration |

### Benchmarks

| Benchmark | Pass Criteria |
|-----------|--------------|
| Auth middleware latency (10K requests) | p95 < 5ms |
| Concurrent auth validations | 1,000/sec without errors |

---

## 12. Operational Procedures

### Key Rotation

Keycloak rotates signing keys automatically. When rotation happens:
1. New tokens signed with new key
2. Old key remains in JWKS for grace period (default: 24h)
3. Auth middleware fetches fresh JWKS on unknown key ID
4. No downtime or manual intervention required

### Emergency Token Revocation

To revoke all sessions for a user:
1. Revoke sessions in Keycloak admin console
2. Add affected JTIs to Redis blocklist (automated via Keycloak event listener)
3. Tokens expire naturally within 15 minutes

---

## 13. Risks and Open Questions

| Risk / Question | Severity | Mitigation / Status |
|----------------|----------|---------------------|
| JWKS cache stale during key rotation | Medium | Auto-refresh on key-not-found; max 1 retry |
| Redis failure blocks all auth | High | Circuit breaker pattern; fallback to JWT-only validation (no revocation check) — requires explicit opt-in |
| Token size grows with custom claims | Low | Monitor token size; migrate to realm-per-tenant if > 4KB |
