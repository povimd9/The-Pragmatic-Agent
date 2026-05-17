# Module: Fix Root Causes, Not Symptoms

**Load when:** a bug is reported AND you're considering a retry loop, timeout bump, fallback default, warning suppression, or cache-extension as the fix.

**Core rule lives in:** [`AGENT-INSTRUCTIONS.md` §11.1](../AGENT-INSTRUCTIONS.md#11-surface-issues-dont-bypass-dont-paper-over).

**Silent-plausibility variant fought:** patches that look like reliability engineering but actually mask the underlying bug — the symptom goes away from view, the problem stays.

---

## The Rule

When a bug is reported, trace it to the specific line / config / decision that causes it. **NEVER** propose compensating logic that masks the symptom — no retry loops, no timeout bumps, no warning suppression, no fallback-to-default, no cache-extension.

If a fix is genuinely a third-party-code-only mitigation (you can't fix the upstream), label it "mitigation" explicitly in the commit message AND the LLD Risks section. Mitigation IS allowed; silently treating-the-symptom is not.

## The Anti-Pattern Catalog

| Anti-pattern | Looks like | Actually is |
|---|---|---|
| Retry loop around an intermittent failure | "Improved reliability" | Hides a vendor outage, a flaky network path, a race condition |
| Timeout bump | "We were too aggressive" | Hides a slow query, a missing index, an unbounded loop |
| Fallback-to-default on parse error | "Graceful degradation" | Hides a schema drift, a vendor API change, malformed input the producer should fix |
| Warning suppression (`# noqa`, `// nolint`, `# shellcheck disable`) without an in-source justification | "Clean output" | Hides a real defect the linter caught |
| Cache TTL extension to mask vendor instability | "Tuning for our usage" | Hides upstream rate-limit hits, vendor planning gaps |
| `try { ... } catch { return defaultValue; }` | "Defensive coding" | Hides a contract violation upstream |
| Re-running a flaky test in CI on retry | "Flaky test handling" | Hides a real race condition or timing dependency |

If your fix is on this list, STOP. Find the root cause first.

## The Diagnostic Question

Before applying any fix, answer:

> **"What is the failure mode that this fix prevents from being observed?"**

If the answer is "the failure mode doesn't go away — we just don't see it" — that's a paper-over. Find what triggers the failure, fix that.

If the answer is "the failure mode can't happen anymore because we eliminated the cause" — that's a root-cause fix.

## Worked Example

```python
# FORBIDDEN — paper over (intermittent vendor 503)
def fetch_price(ticker):
    for attempt in range(5):
        try:
            return vendor.get(ticker)
        except VendorError:
            time.sleep(2 ** attempt)
    return None   # silent failure after retries — silent-plausibility instance

# DIAGNOSE FIRST
# Why is vendor.get() raising VendorError? Three branches:
#   (a) Our request rate exceeds the vendor's quota.
#   (b) The vendor's cache is warming after a deploy.
#   (c) We're calling the wrong endpoint for the asset class.
# Each has a different fix.

# (a) → CORRECT: respect the rate limit at the caller
async def fetch_price(ticker):
    await rate_limiter.acquire()  # wait until our quota allows
    return vendor.get(ticker)     # raises on 503; orchestrator decides

# (b) → CORRECT: mitigation, EXPLICITLY LABELLED
# MITIGATION (vendor-side, can't fix upstream): vendor's cache warms
# for ~30s after their deploys. Retry up to 3x with backoff; if all
# fail, raise (do NOT substitute None). Logged in LLD Risks §X.Y.
def fetch_price(ticker):
    for attempt in range(3):
        try:
            return vendor.get(ticker)
        except VendorError as e:
            if e.status != 503:
                raise   # not the warming case
            time.sleep(2 ** attempt)
    raise VendorWarmingError(ticker, attempts=3)
```

The difference between (a)+(b) and the original: each branch fixes A SPECIFIC root cause. The original tried to handle ALL failures with one retry loop, masking the difference between rate-limit and vendor-503 and wrong-endpoint.

## ARC REVIEW REQUIRED

When the bug touches an architectural / design-level choice (e.g., the underlying contract between two services is wrong), the fix isn't a code-level retry — it's a design change. Flag the bug as `ARC REVIEW REQUIRED`:

- Document the current design.
- Document why it causes the problem (failure mode + reproduction steps).
- List 2-3 alternatives with trade-offs.
- Do NOT apply a code-level fix.

The operator picks the direction; that becomes a new decision-log entry; implementation follows. Skipping the ARC REVIEW step and applying a code-level fix is the worst paper-over kind — it ships a "fix" that disguises the architectural problem so it gets harder to address later.

## Mitigation Discipline

When a fix IS a third-party-code-only mitigation:

1. Label it `MITIGATION` in the commit message subject.
2. The commit body cites WHY the upstream can't be fixed (closed-source vendor, deprecated library, etc.).
3. The relevant LLD's Risks section gets a new entry documenting the mitigation + the conditions under which it should be revisited.
4. The mitigation has bounded scope — wraps only the specific failure mode, raises on everything else.

The mitigation/fix distinction is not bureaucracy — it's so the next person reading the code knows whether the "weird-looking" code is intentional (mitigation) or accidental (paper-over).

## Logging Gap ≠ "Didn't Happen"

A related anti-pattern: a bug report says "X didn't happen" and the first instinct is to add retry logic to make sure X happens. But maybe X DID happen and the log line that would have shown it was missing. Verify logger coverage BEFORE concluding the behavior didn't occur. Reach for §11.1 (this rule) before any retry / timeout / fallback patch — symptoms presenting as "this event didn't happen" are often "we didn't log it."
