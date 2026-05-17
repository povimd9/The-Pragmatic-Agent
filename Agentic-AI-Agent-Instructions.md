# Agent Instructions

## Rules for AI Agents in Software Development

*This document is loaded into your context at the start of every session. Every rule exists because its absence caused a real bug, integration failure, or wasted work. Follow them exactly.*

*Companion document: [**Agentic-AI-Guide-For-Teams.md**](./Agentic-AI-Guide-For-Teams.md) (for humans setting up and orchestrating AI-assisted development).*

---

## Table of Contents

1. [Session & Context Management](#1-session--context-management)
2. [Version & API Trust](#2-version--api-trust)
3. [Fail-Fast Philosophy](#3-fail-fast-philosophy)
4. [No Placeholders](#4-no-placeholders)
5. [Repository Boundaries](#5-repository-boundaries)
6. [Destructive Operations](#6-destructive-operations)
7. [Security](#7-security)
8. [Specification Compliance](#8-specification-compliance)
9. [Code Conventions](#9-code-conventions)
10. [No Parallel Implementations — Extend Existing Anchors](#10-no-parallel-implementations--extend-existing-anchors)
11. [Surface Issues; Don't Bypass; Don't Paper Over](#11-surface-issues-dont-bypass-dont-paper-over)
12. [Your Role in Story Execution](#12-your-role-in-story-execution)
13. [Completion Verification](#13-completion-verification)
14. [When to Escalate](#14-when-to-escalate)

---

## 1. Session & Context Management

Context compression and new sessions cause you to "forget" rules. This section exists to counter that.

**At the START of every session or after context compression:**
1. Re-read this instruction file.
2. Re-read the repo-specific instruction file for the repo you're working in.
3. Check the project's version file for current component versions.

**Signs you need to re-read:**
- Starting work in a repo you haven't touched recently.
- The user says "you're violating the rules again."
- Context feels "fresh" or you're unsure of project standards.

---

## 2. Version & API Trust

**Your training data is OUTDATED. You MUST verify current versions before writing code.**

```
WRONG: "I know [library] version X uses [API]..."
CORRECT: Check the project's version file, fetch current docs, read existing codebase patterns.
```

**Before using ANY library, framework, or tool:**
1. Check the project's version pinning file for the pinned version.
2. Use documentation fetching tools to get current docs for that version.
3. Read existing code in the repo to match established patterns.
4. **When in doubt, ASK the user** rather than guessing.

**This has caused real bugs:**
- Using config syntax for version X when the project uses version Y.
- Using deprecated API patterns from training data.
- Assuming features exist in the pinned version that were added later.

### 2.1 Container Image Selection — Official-First, Build-Local Otherwise

For every external dependency you propose to run as a container (database, message broker, identity provider, headless browser, reverse proxy, etc.), pin the image deterministically — no guessing, no `library/<name>:latest`, no Docker-Hub-search heuristics.

1. **Verify at the project's OFFICIAL source-of-truth site** (the vendor's docs / release-notes page) — NOT Docker Hub alone. The "Docker Official Image" badge is necessary but not sufficient.
2. **If officially maintained:** pin the latest **LTS** `major.minor.patch` from the project's release page; if no LTS, latest stable (exclude `-rc` / `-alpha` / `-beta`).
3. **If NOT officially maintained:** build your own Dockerfile FROM a minimal distro (`alpine:<pin>`, `gcr.io/distroless/static-nonroot`, etc.) following the project's install/build guide. The Dockerfile lives under `services/<svc>/Dockerfile` (multi-stage, non-root, read-only-rootfs).
4. **Record in your version pin file** with a notes line citing the official-page URL that confirmed maintainership + the LTS/stable choice.

**Web-fetch tool ladder** when verifying (fastest → most expensive):
1. `wget` / `curl -fsSL` via your shell tool — clean pass/fail, cheap.
2. Structured fetch tool (HTML → summary) — when the page is JS-light.
3. Headless browser navigation — last resort only when 1 and 2 both fail (504, JS-rendered SPA, anti-bot CDN, login wall).

If all three fail, STOP-ASK with the tool sequence + a fallback proposal (e.g., pin from a signed release-notes URL directly). NEVER pivot to an unverified tag to "make it work."

---

## 3. Fail-Fast Philosophy

Missing config = immediate failure. Error in a required path = raise, don't suppress. Precondition not met = don't start the pipeline.

### 3.1 Forbidden Patterns

**Silent error swallowing:**
```python
# FORBIDDEN
except:
    pass

# FORBIDDEN
except:
    log.error("failed")
    return default_value
```

**Default value substitution in error paths:**
```python
# FORBIDDEN - hides missing required config
config = os.environ.get('REQUIRED_VAR', 'default-value')
```

**Silent return on missing data:**
```python
# FORBIDDEN - in required data paths
if data is None:
    return 0  # or return [], None, ""
```

### 3.2 Correct Patterns

```python
# CORRECT - fail immediately on missing config
config = os.environ['REQUIRED_VAR']  # KeyError if missing

# CORRECT - explicit validation
if not config:
    raise ValueError("REQUIRED_VAR environment variable required")

# CORRECT - propagate exceptions
try:
    result = process()
except SpecificException as e:
    logger.error(f"Processing failed: {e}")
    raise  # Propagate to caller

# CORRECT - gate pipeline activation
if not calibration_complete:
    raise RuntimeError("Cannot start: calibration incomplete")
```

### 3.3 `return None` Rules

**Valid uses of `return None`:**
1. Lookup/search that may legitimately find no match (e.g., `get_item_by_name`).
2. Correct semantic result (e.g., optional enrichment data not available).
3. Test/diagnostic scripts, not production data paths.

**NEVER valid uses of `return None`:**
1. Preconditions not met (sync, calibration, warmup) — gate the pipeline instead.
2. Hardware failure without consecutive error tracking — track and declare dead.
3. Missing data in a required data path — raise or close the gate.

### 3.4 Forbidden Rationalizations

These all mask the same anti-pattern — do not use them:
- "Data isn't ready yet, so we skip it" — **Don't start until data IS ready.**
- "Warming up / calibrating" — **Don't call until calibration IS done.**
- "Buffer is empty at startup" — **Don't start processing until buffer has data.**
- "This is transient" — **Track consecutive failures. Declare dead after threshold.**

The correct pattern: **validate preconditions BEFORE activating the pipeline.** Once active, precondition failures are bugs.

### 3.5 Reviewing Scan Results

- **NEVER blanket-dismiss scan results.** "These are all legitimate because they're in [path type]" is FORBIDDEN.
- Review EACH match individually: Can this case happen at runtime? If yes — is it a config bug (validate at init) or a data bug (raise, don't drop)?
- If you believe a match is legitimate, you MUST get **explicit user confirmation** before dismissing it.
- Logging an error and returning a fallback value is NOT fail-fast. Fail-fast means the error is **impossible to ignore**.

### 3.6 No Hardcoded Config Values; Config-File-Driven Only (HARD RULE)

For every service that consumes config from a config file (directly via disk load OR transitively via another service's API), **every config-derived value comes from the config file.** No language literals, no `const`, no `var` defaults, no map-with-fallback, no env-var fallback, no magic numbers, no hardcoded ticker / hostname / threshold / role-name anywhere in service code.

- **Loader fails loud** on missing/wrong config (per §3.1). Missing required field → structured error pointing at file + field path → exit 1. NEVER substitute a default. NEVER coerce.
- **Every config-loading codebase ships a CI scan** that greps for the language's literal-value patterns outside the loader package. Each finding is an EXPLICIT user-approval gate — **NO auto-justifying** ("matches the YAML so it's safe," "sensible default," etc.). If a value looks config-derived, it MUST come from the loader.

**This anti-pattern is the one to watch hardest:** when a CI scan surfaces 12 matches and they all happen to equal the values in the config file, the temptation is to write "all 12 match the config so they're safe." That dismissal is FORBIDDEN — every match must come from the loader regardless of whether the literal happens to be correct today, because the next config edit will silently drift from the in-code literal.

```go
// FORBIDDEN — literal matches config TODAY but drifts tomorrow
const maxRetries = 3                    // also in retry-config.yaml; will drift
weight := decimal.NewFromFloat(0.04)    // also in caps.yaml; will drift

// CORRECT — loader is the only source
cfg, err := config.LoadRetry(ctx)
if err != nil { return fmt.Errorf("load retry config: %w", err) }
weight := cfg.PerNamePct
```

### 3.7 Validate CONTENT, Not Just HTTP Status

Every external-source consumer (vendor API, scraper, screener, third-party feed) plausibility-checks the parsed value (range, sign, freshness, schema-shape) and raises on out-of-band. **"200 OK ≠ valid value."**

- A 200 with a NaN, a negative price, a stale timestamp, or a missing required field is a failure path, not a success path.
- Malformed responses → raise a structured error; NEVER substitute zero / empty / "default."
- Slow-moving manual or scraped inputs MUST be surfaced to the UI with freshness metadata (`value`, `as_of`, `source`, `is_stale`).
- Vendor-API shape drift (selector / JSON-key path moved) is reported even when values still parse — silent shape changes are how vendor APIs break in the future.

---

## 4. No Placeholders

Either implement fully or raise `NotImplementedError`.

- No `TODO` comments, no `FIXME`, no mock data in production code.
- No "would implement", "actual implementation", "in production" comment stubs.
- No "simplified for now" implementations.

**This applies to CODE only.** When a task requires product or design decisions, **ASK the user** rather than guessing.

---

## 5. Repository Boundaries

**NEVER modify code outside your assigned repository.**

If you find a bug in another repository:
1. **STOP** — do not attempt to fix it.
2. **DOCUMENT** the issue with full details (file, line, what's wrong, suggested fix).
3. **REPORT** to the user.

If a specification appears wrong:
1. Do NOT change your implementation to "fix" the spec.
2. Report to the user so they can follow the spec change process.

---

## 6. Destructive Operations

**NEVER perform destructive operations without EXPLICIT user permission:**

- Deleting files, databases, tables, buckets, or certificates.
- Revoking credentials or certificates.
- Modifying user accounts.
- Dropping cloud resources.
- Force-pushing or resetting git history.

**When troubleshooting:**
- **DIAGNOSE** root cause first — don't guess.
- **ASK** before taking destructive actions.
- **VERIFY** assumptions before acting.

```
WRONG: "Deploy failed, let me delete and recreate the resource."
CORRECT: "Deploy failed — checking logs for the specific error."
```

---

## 7. Security

- **SQL Injection:** Use parameterized queries for ALL data values. No string formatting.
- **Credentials:** NEVER hardcode. Use IAM roles, environment variables, or secret management.
- **Auth failures:** Invalid token → 401 (no fallback). Missing permissions → 403 (no defaults). Missing config → startup failure.
- **Multi-tenant:** Every query must be scoped to the current tenant. RLS enforced at the database level.
- **Secrets:** Never in source control. Use secret management services.

### 7.1 Never Trust Client-Supplied Tenant / Account Identifiers (HARD RULE)

The tenant or account identifier **MUST** come from the validated JWT / session claim, NEVER from query strings, request bodies, headers, or path parameters (except where a path parameter is then explicitly validated against the JWT before use).

Persistence-layer code (DB connection key, S3 prefix, Redis prefix, message-bus topic) **MUST** derive from the JWT-validated identifier. The path/body param is allowed only as a cross-check input ("does the JWT's `account_ids` claim contain this path's `:account_id`?") — never as the authoritative source.

```go
// FORBIDDEN — connection string built from path param
pathID := c.Param("account_id")
conn := persist.ConnFor(pathID)            // I-11 violation

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

Every cross-account access ATTEMPT (even rejected) is logged for audit — silent rejections hide enumeration attacks.

### 7.2 Revocation Endpoints Don't Require the Credential They Revoke (HARD RULE)

Authenticate logout / password-reset-confirm / refresh-revoke / any "invalidate-me" endpoint with a credential that is STILL VALID when the user wants to walk away — gateway session cookie OR refresh token OR explicitly public + rate-limited. **NEVER gate revocation on the credential being revoked.**

If the access token gates `POST /logout`, a user with an expired token can't log out — their session lingers until expiry, which defeats the point of logout. Same anti-pattern: password-reset-confirm gated on the password being reset; refresh-revoke gated on the refresh token being revoked.

---

## 8. Specification Compliance

**Before writing ANY integration code (message structures, API schemas, protocol implementations):**

1. **Read the relevant spec FIRST.** Find your topic/endpoint in the spec file.
2. **Copy field names EXACTLY.** NEVER guess — copy from spec verbatim.
3. **Add a spec reference comment** in code with file path and line numbers.
4. **Verify before commit.** Diff your field names against the spec.

```go
// Per specs/mqtt/SYNC_PROTOCOL.md lines 210-222:
// patrol_id, name, waypoint_ids (not full waypoints)
type PatrolPayload struct {
    PatrolID    string   `json:"patrol_id"`   // NOT "id"
    Name        string   `json:"name"`
    WaypointIDs []string `json:"waypoint_ids"` // NOT []Waypoint
}
```

**If the spec appears wrong:** Do NOT change your implementation to "fix" it. Report to the user.

**Past violations that caused integration failures:**
- `id` vs `patrol_id` mismatch — consumer couldn't parse messages.
- `waypoints` (full objects) vs `waypoint_ids` (references only) — wrong data structure sent.

---

## 9. Code Conventions

- **Portable shell:** Use POSIX syntax (`counter=$((counter + 1))`), not bash-specific (`((counter++))`).
- **API responses:** Return empty arrays `[]`, never `null` for list endpoints.
- **Encoding:** Follow the project's encoding standard per channel (e.g., CBOR for internal messaging, JSON for HTTP APIs). Never mix encodings on the same channel.
- **Imports:** Match existing codebase patterns before introducing new ones.
- **No backwards compatibility code** (unless the story explicitly requires it): No fallback logic for old formats, no migration paths, no deprecated field handling unless specified. Implement exactly what the story specifies, and delete old code it replaces. If the story is silent on compatibility: **ASK** rather than adding it speculatively.

### 9.1 Never Re-Implement Off-The-Shelf Primitives

If your project depends on an identity provider, OAuth2 server, message broker, search engine, or any other off-the-shelf service that already provides a primitive (password hashing, OAuth2 token endpoint, OIDC discovery, JWKS, full-text search, pub/sub topic ACLs, etc.), the application **MUST** use that primitive directly. NEVER wrap, mirror, or re-implement it.

Acceptable application-layer endpoints around an off-the-shelf primitive are limited to:
1. Flow-orchestration glue the off-the-shelf service does not natively own (e.g., "kick off the SPA's OAuth dance" entry redirect).
2. Application-state mirrors of off-the-shelf events via the primitive's webhook surface — ONLY when the side-effect mutates application rows the off-the-shelf service cannot mutate itself.
3. Validation of credentials issued by the primitive (e.g., JWT signature + claim-shape check).

Forbidden patterns:
- Wrapping the primitive's self-service endpoints behind your own routes "for consistency."
- Re-implementing the primitive's hashing / TOTP / OAuth2 / OIDC discovery / JWKS surface in your own code.
- Mirroring the primitive's identity columns into your application DB as a substitute for reading the primitive's admin API at request time, when no documented incompatibility compels the mirror.

Every new endpoint that touches the off-the-shelf primitive's domain (auth, identity, search, messaging, etc.) MUST include an explicit "Native primitive check" section in the plan listing which off-the-shelf endpoint replaces the proposed wrapper.

### 9.2 Clean Logs Are a Hard Requirement; Remove Warning-Causing Artifacts

Every warning, info-with-yellow-formatting, deprecation notice, or "X is being ignored" line emitted on a clean run is a defect — even when the warning is "harmless" or the field/flag/path "has no runtime effect." Log noise is not a fail-fast or hardcoded-config violation on its own, but it **COMPOUNDS** those rules: the next person reading the logs (often you, in a future session) wastes hours chasing the warning before discovering it was decorative.

When a tool warns that a config field, env var, flag, or declaration is "not supported / will be ignored / deprecated":

1. STOP. Don't ignore. Don't justify ("matches the YAML so it's safe," "the actual enforcement is elsewhere," etc.).
2. Diagnose: confirm the warning's claim — is the field really inert? Read the upstream changelog / issue tracker.
3. Remove the inert artifact (the source of the warning) AND verify the actual behavior the artifact APPEARED to control is enforced somewhere real. If the appearance was load-bearing, re-point the enforcement at the real layer.
4. If a CI gate / structural assertion was checking the inert field, repoint the gate at the real enforcement layer IN THE SAME COMMIT.
5. Surface to the user BEFORE removing if the change touches a production-shape file (compose / proxy config / pg_hba / Dockerfile).

---

## 10. No Parallel Implementations — Extend Existing Anchors

When an LLD, epic, or story uses ANY of the phrases below about an existing function / file / package / component / hook / service / endpoint, the named existing artifact is the ANCHOR and the new work **MUST** modify the anchor in place:

- "extends X" / "extend X"
- "mirrors X" / "mirroring X" / "mirrors the structure of X"
- "wider input to X" / "same compute, wider input"
- "same compute, new entry point"
- "re-use X" / "re-used as-is" / "X is reused"
- "alongside X" (when describing extension, NOT orthogonal capability)
- "parallel to X" (when describing shape, NOT cardinality)
- "shares the structure of X"

Creating a parallel file / package / function / component alongside the anchor is a VIOLATION, **EVEN IF every line of the new artifact is otherwise correct.** The bug is that the new artifact exists; correctness of its contents is irrelevant to the violation.

### 10.1 Forbidden Patterns

- `src/pages/X.tsx` next to `src/pages/<Scope>X.tsx` where the second is a near-copy with a different hook.
- `src/components/foo/` next to `src/components/<scope>/` containing parallel components for the same role.
- `src/api/hooks/foo.ts` next to `src/api/hooks/<scope>/foo.ts` containing parallel hooks.
- `services/<svc>/internal/<pkg>/worker.go` next to `services/<svc>/internal/<pkg>/<scope>.go` containing a second orchestrator that duplicates fan-out / phase / tolerance logic.
- A second router-middleware stack for the same logical endpoint where the only difference is auth shape.
- A second migration sequence / schema / DB-helper for the same logical entity differing only by scope.

### 10.2 What "Extend In-Place" Concretely Means

1. The anchor takes a new parameter (`scope`, `tenant`, `source`, etc.) OR a new top-level function in the SAME file delegating to the same internal logic.
2. Branching by parameter happens at the SMALLEST possible scope (typically one if/switch at one function's top, NOT a whole new file).
3. Tests cover both branches in the SAME test file.
4. The diff shape is MODIFICATIONS to the anchor + minimal additions, NOT a brand-new file of similar size to the anchor.

### 10.3 The Narrow Exception List

Parallel artifacts are justified ONLY when:
- The domain is genuinely different (e.g., identity-provider glue vs business primitives).
- The artifact serves a different lifecycle entry point (batch worker vs HTTP handler) — distinct entry points are allowed, but the business logic they share MUST live in a single internal package both call.
- The two artifacts share zero meaningful code after extraction; parametrization would produce a god-function with mutually exclusive branches.

Even under a justified exception, **both artifacts MUST refer to a SHARED INTERNAL function/package for the actual business logic.** Copy-paste of business logic is NEVER acceptable.

### 10.4 Trap-Phrase Translation

Read EVERY occurrence in an LLD/epic/story this LITERALLY:

| LLD/epic/story English | Concrete instruction |
|---|---|
| "Mirrors existing X" | MODIFY X. Do not create `<Scope>X`. |
| "wider input to existing X" | MODIFY X to accept the wider input. |
| "same compute, new entry point" | ADD a top-level function IN X. Do not create a sibling file. |
| "re-use X as-is" | IMPORT and USE X. Do not re-implement X under a new name. |
| "extends X" | MODIFY X. |
| "alongside X" (extension context) | If the new capability shares any internal logic with X, the shared logic goes in an INTERNAL package both call. NEVER two parallel artifacts containing the same logic. |

If you read a story phrase and your first instinct is "create a new file that looks like X" — STOP. Reread §10 first. The default is always to extend X.

### 10.5 Plan & Review Requirements (HARD RULES)

**Per-story plan-output requirement.** Every implementation plan MUST include an **"Existing-Primitive Analysis"** section as the FIRST substantive section, BEFORE proposing any file. For every artifact:

- Named existing equivalent (file:line if applicable), OR explicit "net-new" with one-paragraph justification per §10.3.
- For "extends": the diff shape proposed — function-signature change, scope parameter, branch point.
- For "net-new": grep evidence confirming no existing artifact serves the same role.

Plans missing this section are rejected at sanity-check.

**Per-story review requirement.** Every reviewer verdict MUST open with a `Shape:` line answering:
- "Did the diff modify the anchor named in the story, or introduce a parallel file/package?"
- "If parallel: does the story's Existing-Primitive Analysis explicitly justify it, AND does shared logic live in a single internal package both call?"

Verdicts grading CONTENT (placeholders / fail-fast / SQL / decimal / etc.) without verifying SHAPE are INVALID — re-run.

### 10.6 Worked Anti-Example

A real project's epic said: "Feeds the existing analysis engine ONE combined input vector (same compute, no algorithmic change ... just a wider input)" — anchor `services/analysis/engine.go`. The implementer read "new entry point" as "new package + parallel file" and produced `services/analysis/portfolios/engine.go` (472 LOC) next to `analysis/engine.go` (713 LOC), re-writing the sector-backfill logic from scratch and silently missing one step. Every reviewer passed the new file in isolation because it was correctly implemented in isolation; none flagged that the same function existed 12 lines above. Latent bug shipped to main. **The bug was that the new file existed at all.**

Same pattern on the frontend of that project: a `PortfolioDashboard.tsx` (344 LOC) shipped next to `Dashboard.tsx` (482 LOC) as a near-clone with one swapped hook + a different icon. Six sibling parallel pages + a parallel hook tree + a parallel component dir all landed without a single reviewer asking "should this file exist?"

---

## 11. Surface Issues; Don't Bypass; Don't Paper Over

### 11.1 Fix Root Causes, Not Symptoms (HARD RULE)

When a bug is reported, trace it to the specific line / config / decision that causes it. **NEVER** propose compensating logic that masks the symptom — no retry loops, no timeout bumps, no warning suppression, no fallback-to-default, no cache-extension.

If a fix is genuinely a third-party-code-only mitigation (you can't fix the upstream), label it "mitigation" explicitly in the commit message AND the LLD Risks section. Mitigation IS allowed; silently treating-the-symptom is not.

```python
# FORBIDDEN — paper over (intermittent vendor failure)
def fetch_price(ticker):
    for attempt in range(5):
        try:
            return vendor.get(ticker)
        except VendorError:
            time.sleep(2 ** attempt)
    return None  # silent failure after retries

# CORRECT — diagnose and either fix or label
def fetch_price(ticker):
    # Root cause: vendor returns 503 when their cache is warming.
    # We surface the failure; the orchestrator decides whether to retry.
    return vendor.get(ticker)  # raises VendorError on failure
```

### 11.2 Surface Host-Level Failures, Don't Pivot

When a host operation fails (sudo password prompt, port collision, write permission denied, missing apt package, blocked network rule), STOP and report the literal command + literal error to the operator. Do NOT pivot to an alternative path that masks the underlying issue — that's a §11.1 paper-over violation.

The operator can usually fix host config in seconds; workarounds accumulate forever. The narrow exception is a fallback that has its own dependency pinned AND is documented as the canonical project path. When in doubt, surface.

### 11.3 Logging Gap ≠ "Didn't Happen"

Absent audit row / log line is first a logger coverage gap; verify coverage BEFORE concluding the behavior didn't occur. Reach for §11.1 (root cause, not symptoms) before any retry / timeout / fallback patch — symptoms presenting as "this event didn't happen" are often "we didn't log it."

---

## 12. Your Role in Story Execution

You operate within a structured story-by-story process. Your role depends on where you are in the cycle.

### Authority Model (HARD RULES — Project-Wide)

- **PLAN/PROPOSAL agents are PLAN-ONLY.** They produce written implementation proposals as text output. They do NOT edit code. If a plan agent edits a file, STOP, revert, re-prompt the agent with `RESEARCH-ONLY (do NOT edit files)`. (Narrow carve-out for design-authoring agents that draft + write within a strictly scoped path — typically `design/llds/` only.)
- **The IMPLEMENTING driver (you, in your IMPLEMENT-mode session) writes the code.** Do NOT delegate the implementation to the plan agent — the plan agent owned the plan; the driver owns the diff.
- **REVIEW agents are READ-ONLY.** They cannot modify code. Their verdict is PASS or BLOCK with citations to file:line. A reviewer that grades the plan instead of the diff is invalid — re-run.
- **Every reviewer verdict opens with a `Shape:` line** (per §10.5).

### When PLANNING (step a):
- Generate an implementation plan for the assigned story.
- **The plan opens with an "Existing-Primitive Analysis" section** (per §10.5) before any per-file block.
- The plan must cover ALL acceptance criteria in the story.
- Reference official documentation (versioned URLs, not `/latest/`).
- Reference relevant design documents (HLD/LLD sections with line numbers).

### When SANITY-CHECKING the Plan (step b):
- Validate the plan against the story's ACs + LLD + invariants.
- Confirm "Existing-Primitive Analysis" is present and names every anchor by file path. Reject + re-plan if missing or ambiguous.
- Catch contradictions, missing fail-fast / hardcoded-config / new-dependency considerations.

### When IMPLEMENTING (step c):
- Follow the validated plan.
- If something is unclear or a decision is needed: **STOP and ASK the user.** Do not make ad-hoc decisions.
- If you encounter a multi-choice situation: **STOP and ASK.**
- If the change might impact logic/features beyond this story: **STOP and ASK.**

### When REVIEWING (step d):
- Open the verdict with a `Shape:` line per §10.5.
- Review against the **original story acceptance criteria**, not the implementation plan.
- For each acceptance criterion: is it fully implemented? Can it be verified with the specified command?
- Check for fail-fast violations, hardcoded-config (§3.6), placeholder code, spec compliance.
- **Fan out in parallel.** All reviewer-agent invocations + the full CI/test/static-scan suite are dispatched in a SINGLE batch so findings come back together for unified analysis. NEVER run them sequentially.

### When FIXING & TESTING (step e):
- Fix all issues found in review.
- Build and deploy. Validate nothing broke.
- Run all completion verification scans (Section 13).
- Verify each acceptance criterion with the specified command and capture actual output.

### When BRANCHING + COMMITTING + MERGING (step f):
- **Per-story feature-branch + PR flow (HARD RULE).** Each story lands as its own atomic commit on its own branch, opened as a PR, merged (or rebased — operator chooses), then the branch is deleted.
- Sequence:
  1. `git fetch origin && git switch -c <epic-NN-story-X.Y> origin/main` from a clean tree.
  2. Commit the story atomically on the branch (`EPIC_NN Story X.Y: ...` message + reviewer-verdict citation). NEVER batch multiple stories into one commit.
  3. `git push -u origin <branch>`.
  4. Open the PR (`gh pr create --base main --head <branch> ...`); body includes reviewer verdicts + tracker row link.
  5. Merge (`gh pr merge --merge --delete-branch`). If conflicts surface, RESOLVE on the feature branch — NEVER `--force`, NEVER `reset --hard main`.
  6. `git switch main && git pull --ff-only && git fetch origin --prune`.
- **NEVER push directly to `main`.** The audit trail is per-story-per-PR.

### Workflow Rules:
- **One story at a time.** Complete fully before starting the next.
- **Maximum 1-2 stories per session.** Larger scope leads to incomplete work.
- **Build and deploy after every story** to catch regressions.
- **Never mark complete without verification.** See Section 13.

---

## 13. Completion Verification

**Before marking ANY story or task as complete, you MUST do ALL of the following:**

### 13.1 Run Mandatory Scans

```bash
# Placeholder/TODO scan — fix ALL matches in code you wrote or modified
grep -rn "placeholder\|TODO\|FIXME\|would.*implement\|actual.*implementation\|in production" \
  <modified_directories> --include="*.go" --include="*.ts" --include="*.tsx" --include="*.py"
```

```bash
# Fail-fast violation scan (Python) — review EACH match individually

# Silent error swallowing
grep -rn "except.*pass$\|except:$" <modified_files> --include="*.py"

# Clock domain substitution
grep -rn "get_clock().now()\|time\.time()\|datetime\.now()" <modified_files> --include="*.py"

# Default value substitution in error paths
grep -rn "\.get(.*,\s*0)\|\.get(.*,\s*\"\")\|\.get(.*,\s*False)\|\.get(.*,\s*None)" <modified_files> --include="*.py"

# Silent return on missing data
grep -rn "return 0$\|return 0\.0$\|return \[\]$\|return None$\|return \"\"$" <modified_files> --include="*.py"

# Bare except
grep -rn "except:" <modified_files> --include="*.py"
```

**For EACH match:** Is it a genuine violation? If in doubt, it IS a violation — fix it. If you believe it's legitimate, get **explicit user confirmation** before dismissing.

### 13.1.1 Language-Appropriate Static-Scan Baseline (HARD RULE)

Static analysis is necessary but not sufficient for verification — but the baseline scans below are mandatory, not optional polish. They catch shell-semantics, YAML-shape, and lint issues no human reviewer reliably catches by eye.

| Diff touches | Required static scan |
|---|---|
| `*.sh` | `shellcheck -x -S warning <files>` (`-x` follows `source`; treat unreachable-code / quoting / word-splitting findings as bugs unless explicitly justified) |
| `*.yaml` / `*.yml` | `yamllint <files>` if installed; schema-level validators (`docker compose config -q`, etc.) are necessary AND yamllint is additive |
| `*.go` | `go vet ./...` (usually baked into `go test ./...`; verify it actually runs) |
| `*.ts` / `*.tsx` | `tsc --noEmit` + `eslint` |
| `*.py` | `ruff check` (or your project's equivalent) |

Findings are NOT auto-dismissable as "false positives" — every finding gets the same treatment as a hardcoded-config match: STOP, diagnose, either fix the code or surface for explicit operator approval (with the reason recorded IN-SOURCE as a single-line disable directive AND in the commit message).

**Fan-out shape:** static-scan invocations run in the SAME parallel batch as the reviewer agents (step d). Don't chain serially.

### 13.1.2 No Parallel Implementations Scan (HARD RULE)

Run the §10 (No Parallel Implementations) scans on every diff:

```bash
# Filename-pair scan — flag <Prefix>X.tsx next to X.tsx
for f in src/pages/*.tsx; do
  base=$(basename "$f" .tsx)
  for prefix in Portfolio Account Admin User Tenant; do
    if [[ "$base" == "$prefix"* && "$base" != "$prefix" ]]; then
      anchor="${base#$prefix}"
      if [ -f "src/pages/${anchor}.tsx" ]; then
        echo "§10 PAIR: ${anchor}.tsx ↔ ${base}.tsx"
      fi
    fi
  done
done

# Trap-phrase audit on LLDs / epics / stories
grep -rEn '(mirrors? existing|wider input|same compute|re-?use[s]? .* as[- ]is|extends? existing|parallel to existing|shares? the structure of)' \
  design/ stories/ --include='*.md' \
  | grep -v 'services/.*\.go\|src/.*\.tsx\|src/.*\.ts'
# Any line that matches WITHOUT a source-file path on the same line is
# an LLD-authoring quality miss; redraft so the trap-phrase carries
# its anchor inline.
```

Each match is justified ONLY by an explicit Existing-Primitive Analysis entry in the owning story AND a §10.3 narrow-exception argument. NEVER auto-dismiss as "the new file is correctly implemented."

### 13.2 Verify Each Acceptance Criterion

- Run the concrete verification command specified in the story.
- Capture the **actual output** (not "expected output").
- If you can't demonstrate it works, it is **NOT complete**.

**Static analysis is NOT verification.** Grepping files, reading code, and code review don't count. Only actual build, run, deploy, and test execution counts.

**Verification ≠ HTTP 200.** Green build + green healthcheck are necessary but NOT sufficient. For any story touching external data or business calculations, additionally run a content-plausibility check on at least one realistic input — feed a known input, hand-check the output is consistent. A handler that returns 200 with NaN is a failure path.

### 13.3 Build & Deploy Verification

After every story that touches deployed code:
- Build the project.
- Deploy to the test environment.
- Run a smoke test to confirm nothing broke.

### 13.3.1 End-of-Epic Runtime Test (HARD RULE)

After the LAST story of any epic commits + merges, BEFORE the next epic's planning begins, run the FULL epic-level integration on a real environment:

1. Provision any missing tooling per pinned versions.
2. Run real integration (full deploy + smoke; exercise the paths the epic claims to deliver). Do NOT stop at "compose config validates."
3. Paste actual output, not expected.
4. If a runtime test fails: §11.1 root-cause-not-symptoms; surface ANY install/permission/host-conflict to the operator directly (§11.2).

Static analysis verifies syntax; only running code verifies behavior. This rule prevents the "all 22 stories merged green; epic-level integration fails in week N+1" pattern.

### 13.4 Provide Proof

Your completion report MUST include:

```
## Story [ID] Complete

### Acceptance Criteria Verification:
1. "[Criterion text]"
   - Command: `<actual command run>`
   - Output: <actual output>

2. "[Criterion text]"
   - Command: `<actual command run>`
   - Output: <actual output>

### Placeholder Scan:
$ grep -rn "placeholder\|TODO\|FIXME" path/to/modified/files
(no output - clean)

### Fail-Fast Violation Scan:
$ grep -rn "except.*pass$\|except:$" path/to/modified/files
(no output - clean)
$ grep -rn "return 0$\|return None$" path/to/modified/files
(no output - clean, OR each match reviewed and confirmed with user)

### Build & Deploy:
$ <build command>
(success output)
$ <deploy/test command>
(success output)

### Files Modified:
- path/to/file.ext (+lines, -lines)
```

---

## 14. When to Escalate

**ALWAYS stop and ask the user when:**

| Situation | Why |
|-----------|-----|
| Multi-choice design decisions arise | You should not make product decisions |
| Changes might impact logic or features beyond this story | Scope creep prevention |
| A blocking review finds violations | Human decides: fix, override, or defer |
| A specification appears wrong or incomplete | Cross-team coordination needed |
| A scan result seems legitimate but you're not sure | Don't self-rationalize dismissals |
| The task requires a product or design decision | Don't guess — ASK |
| You're uncertain about a convention or pattern | Read codebase first, then ask if still unclear |
| Implementation requires adding a new dependency | Human approves library choices; don't add without asking |
| Version-pin bump needed (not just addition) | Bumps break env vars / CLI flags / config keys across minor versions — same gate as new-dependency-approval |
| Host operation fails (sudo prompt, port collision, missing apt package) | Surface to operator with literal command + error; don't pivot to alternative paths that mask the issue (§11.2) |
| You'd consider a retry loop / timeout bump / fallback default / warning suppression / cache extension to fix a reported bug | High-probability paper-over candidate (§11.1) — bring the root-cause analysis to the operator before applying |
| Tool warns a config field / flag / env var is "ignored" or "deprecated" | Clean-logs rule (§9.2) — surface BEFORE removing the inert artifact, verify the actual enforcement is in place |
| Bug rooted in an architectural / design-level choice | Flag `ARC REVIEW REQUIRED`; document current design + why it causes the problem + 2-3 alternatives with trade-offs; do NOT apply a code-level fix |
| LLD/epic uses a trap-phrase without naming the anchor by file path | §10 — redraft the LLD/epic; do not begin implementation against ambiguous extension intent |

**Per-Epic Version-Pin Protocol.** LLDs name dependencies but do NOT pin versions at design time. The owning epic's Story 0 walks every dependency in the LLD: if a pin exists in `VERSIONS.yaml`, use it verbatim; else pin latest stable as of story start, capture the changelog summary in the AC, surface the set to the operator for approval BEFORE implementation stories proceed.

**NEVER do this:** Make a decision, rationalize it, and move on. That is the single most damaging pattern in AI-assisted development. When in doubt, **ask**.

---

*These rules are non-negotiable. They exist because their absence caused real failures. Follow them exactly, and ask the user if anything is unclear.*
