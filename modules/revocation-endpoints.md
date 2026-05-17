# Module: Revocation Endpoints Don't Require the Credential They Revoke

**Load when:** you're implementing logout, password-reset-confirm, refresh-revoke, or any "invalidate-me" endpoint.

**Core rule lives in:** [`AGENT-INSTRUCTIONS.md` §7.2](../AGENT-INSTRUCTIONS.md#7-security).

**Silent-plausibility variant fought:** revocation endpoints that look secure because they require a bearer token, but in practice can't be called when the user most needs them (after the token is leaked / expired / suspected compromised).

---

## The Rule

Authenticate logout / password-reset-confirm / refresh-revoke / any "invalidate-me" endpoint with a credential that is **STILL VALID** when the user wants to walk away — gateway session cookie OR refresh token OR explicitly public + rate-limited. **NEVER gate revocation on the credential being revoked.**

## Why This Matters

If the access token gates `POST /logout`, anyone whose access token has already expired can't proactively invalidate their refresh token. The refresh token lives until ITS own expiry — that's the lingering session: the access token is dead, but the refresh token can still mint new access tokens until the refresh expiry window closes. The point of logout is to kill the refresh token NOW; gating logout on a credential that's already expired blocks the user from doing exactly that.

Same anti-pattern applies to:

- Password-reset-confirm gated on knowing the password being reset (impossible: the whole point is the user forgot it).
- Refresh-revoke gated on the refresh token being revoked (chicken-and-egg).
- Account-disable gated on the credential the disabled account uses (locks the operator out).

## Correct Patterns

| Endpoint | Authenticate via | Why |
|---|---|---|
| Logout | Gateway session cookie OR refresh token | Both outlive the access token; user holds them as long as the session is "real" |
| Password-reset-init (user requests reset link) | Public, rate-limited per IP + per email | User by definition can't authenticate; rate-limit prevents enumeration |
| Password-reset-confirm (user submits new password) | One-time token from the reset email | Token is short-lived, single-use, scoped to one user |
| Refresh-revoke (single token) | The refresh token being revoked, OR the parent session | Self-revoke is allowed; parent-session revoke covers compromised-refresh case |
| Refresh-revoke-all (all of a user's sessions) | Gateway session cookie OR admin credential | The compromised-token case demands a credential OTHER than the compromised one |
| Account-disable (admin) | Admin's own valid credentials | The disabled account's credentials are about to be invalid |

## Worked Anti-Examples

```go
// FORBIDDEN — bearer-token-gated logout
router.POST("/logout",
    requireBearerJWT(),   // <-- needs access token, but the user is logging out
    func(c *gin.Context) {
        revokeAccessToken(c)
        revokeRefreshToken(c)
        c.JSON(200, gin.H{"status": "logged out"})
    },
)
// Bug: if the access token is already expired (user closed laptop yesterday),
// the user can't log out — they get 401 from the auth middleware before
// the handler runs. The session lingers.

// CORRECT — session-cookie-gated logout
router.POST("/logout",
    requireSessionCookie(),   // gateway-issued, outlives access tokens
    func(c *gin.Context) {
        sessionID := sessionFrom(c)
        revokeAllTokensForSession(sessionID)
        clearSessionCookie(c)
        c.JSON(200, gin.H{"status": "logged out"})
    },
)
```

## When Public-Plus-Rate-Limit Is Right

Some revocation endpoints have no pre-existing credential to authenticate against (e.g., password-reset-init from the login page). Make them explicitly public + rate-limited:

- Rate limit per IP (catches scripted enumeration).
- Rate limit per email or per username (catches targeted attacks that rotate IPs).
- Constant-time response shape regardless of whether the account exists (defeats account-existence enumeration via response timing).
- Audit log every invocation, public or not.

The rate limit IS the security control; without it, "public" becomes "DoS vector + enumeration vector."

## Test the Failure Path

For every revocation endpoint, the test suite includes:
1. Happy path (legitimate revocation succeeds).
2. **Already-revoked credential** (the credential the endpoint USES is expired/revoked — the endpoint still works).
3. Replay (same revocation request twice — the second is idempotent or explicitly rejected, never a 500).
4. Unauthorized (some OTHER credential tries to revoke — rejected).

If test (2) fails, the endpoint has the anti-pattern.
