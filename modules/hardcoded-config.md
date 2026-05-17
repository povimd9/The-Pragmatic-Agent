# Module: No Hardcoded Config Values — Config-File-Driven Only

**Load when:** you're writing or reviewing code that touches a config-derived value (timeout, threshold, retry count, host name, role name, feature flag, business constant — anything that COULD live in a config file).

**Core rule lives in:** [`AGENT-INSTRUCTIONS.md` §3.6](../AGENT-INSTRUCTIONS.md#3-fail-fast-philosophy).

**Silent-plausibility variant fought:** language literals that happen to match the config file today and silently drift tomorrow.

---

## The Rule

For every service that consumes config (directly from disk OR transitively via another service's API), **every config-derived value comes from the config file.** No language literals, no `const`, no `var` defaults, no map-with-fallback, no env-var fallback, no magic numbers, no hardcoded hostnames / thresholds / role names anywhere in service code.

- **Loader fails loud** on missing/wrong config. Missing required field → structured error pointing at file + field path → exit 1. NEVER substitute a default. NEVER coerce.
- **Every config-loading codebase ships a CI scan** that greps for the language's literal-value patterns outside the loader package. Each finding is an EXPLICIT operator-approval gate — **NO auto-justifying.**

## The Kill-Shot Anti-Pattern

The single most dangerous dismissal:

> "All 12 scan matches happen to equal the values in the YAML, so they're safe."

This is FORBIDDEN. Every match must come from the loader regardless of whether the literal happens to be correct today, because the next config edit will silently drift from the in-code literal. The scan's job is to catch the dual-source-of-truth bug BEFORE it manifests; auto-dismissing on "they match" defeats the scan.

## Examples

```go
// FORBIDDEN — literal matches config TODAY but drifts tomorrow
const maxRetries = 3                    // also in retry-config.yaml
weight := decimal.NewFromFloat(0.04)    // also in caps.yaml
host := "api.internal"                  // also in hosts.yaml

// CORRECT — loader is the only source
cfg, err := config.LoadRetry(ctx)
if err != nil { return fmt.Errorf("load retry config: %w", err) }
maxRetries := cfg.MaxRetries
weight := cfg.PerNamePct
host := cfg.APIHost
```

## What Counts as "Config-Derived"

A value is config-derived if any of these are true:
- It's named in a config file your service loads.
- It's a tunable that an operator might want to change without a code deploy.
- It's a business constant that lives in domain-policy (thresholds, caps, time windows).
- It's an environment-shape value (hostnames, ports, credentials, file paths).

NOT config-derived (and OK to have as literals):
- Mathematical constants (π, e, 2, etc.) — these don't drift.
- Pure-implementation constants (buffer sizes calibrated to CPU cache lines, etc.) — not operator-tunable.
- Test fixtures inside `_test.go` / `*.test.tsx` / etc. — loader doesn't apply to test data.

When in doubt: config-derived. Cheaper to refactor a one-off back to a literal than to find every parallel literal after a config edit drifts.

## Per-Language Scan Templates

See [`scan-templates.md`](./scan-templates.md) for the hardcoded-config detection greps per language.
