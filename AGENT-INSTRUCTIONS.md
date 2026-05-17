# Agent Instructions

## Rules for AI Agents in Software Development

*This document is loaded into your context at the start of every session. Every rule exists because its absence caused a real bug, integration failure, or wasted work. Follow them exactly.*

*Companion document: [**TEAM-GUIDE.md**](./TEAM-GUIDE.md) (for humans setting up and orchestrating AI-assisted development).*

*Deep detail for specific rules lives in [**`modules/`**](./modules/). Load a module when the topic comes up in your work; the [module map](./modules/README.md) lists the trigger conditions.*

---

## Table of Contents

0. [Silent Plausibility — The Meta-Anti-Pattern](#0-silent-plausibility--the-meta-anti-pattern)
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

## 0. Silent Plausibility — The Meta-Anti-Pattern

**Almost every rule in this document fights one underlying failure mode: silent plausibility.**

Silent plausibility is the family of agent-produced artifacts that look right and are wrong. The output passes a casual read, fails a careful one. Examples: a fallback value that hides a missing config; a parallel file that duplicates an existing anchor; a scan finding dismissed as "matches the YAML so it's safe"; a retry loop wrapped around an intermittent failure; a completion report citing a green build but no content-plausibility check; a reviewer verdict that grades content correctness without checking shape.

Crashes are loud and easy. Silent plausibility is the dangerous mode — it ships, the user trusts it, and the bug surfaces weeks later when a config edit or scope change exposes it.

**Map of rules → which silent-plausibility variant each one fights:**

| Rule | Variant it fights |
|---|---|
| §3 Fail-Fast | Silent fallback values in error paths |
| §3.5 Reviewing Scan Results | Auto-justified scan dismissals |
| §3.6 No Hardcoded Config | "Matches the YAML so it's safe" |
| §3.7 Validate CONTENT | 200-OK-but-NaN responses |
| §4 No Placeholders | TODO stubs that look like working code |
| §10 No Parallel Implementations | New files that duplicate an anchor |
| §11.1 Fix Root Causes Not Symptoms | Retry/timeout/fallback patches that mask the real bug |
| §11.3 Logging Gap ≠ Didn't Happen | Absent log line treated as proof the event didn't occur |
| §12 Reviewer `Shape:` line | Content-correct verdicts on shape-broken diffs |
| §13.2 Verification ≠ HTTP 200 | Green-build claims that say nothing about correctness |

When in doubt: **"Could this output be wrong in a way no one will notice?"** If yes, you're in silent-plausibility territory — apply the relevant rule before shipping.

---

## 1. Session & Context Management

Context compression and new sessions cause you to "forget" rules. Counter that:

**At the START of every session or after context compression:**

1. Re-read this instruction file.
2. Re-read the repo-specific instruction file for the repo you're working in.
3. Check the project's version pin file for current component versions.

**Signs you need to re-read:** starting work in a repo you haven't touched recently; the user says "you're violating the rules again"; context feels "fresh" or you're unsure of project standards.

---

## 2. Version & API Trust

**Your training data is OUTDATED. You MUST verify current versions before writing code.**

Before using ANY library, framework, or tool:

1. Check the project's version pinning file for the pinned version.
2. Use documentation fetching tools to get current docs for that version.
3. Read existing code in the repo to match established patterns.
4. **When in doubt, ASK the user** rather than guessing.

**This has caused real bugs:** using config syntax for version X when the project uses Y; using deprecated API patterns from training data; assuming features exist in the pinned version that were added later.

### 2.1 Container Image Selection (summary)

Pin every container image deterministically — no `latest`, no Docker-Hub-search heuristics. Verify maintainership at the vendor's official source-of-truth site (not Docker Hub alone). Prefer LTS over latest-stable. If not officially maintained, build a local Dockerfile FROM a minimal distro. Record the choice in your version pin file with a notes line citing the URL that confirmed it.

**Full protocol + the web-fetch tool ladder:** [`modules/container-image-selection.md`](./modules/container-image-selection.md).

---

## 3. Fail-Fast Philosophy

Missing config = immediate failure. Error in a required path = raise, don't suppress. Precondition not met = don't start the pipeline.

### 3.1 Forbidden Patterns

**Silent error swallowing, default-value substitution in error paths, silent return on missing data** — each is a silent-plausibility (§0) instance:

```python
# FORBIDDEN
except: pass
except: log.error("failed"); return default_value
config = os.environ.get('REQUIRED_VAR', 'default-value')   # missing-required mask
if data is None: return 0                                  # in required data paths
```

### 3.2 Correct Patterns

```python
# Fail immediately on missing config
config = os.environ['REQUIRED_VAR']  # KeyError if missing

# Explicit validation
if not config: raise ValueError("REQUIRED_VAR required")

# Propagate exceptions
try: result = process()
except SpecificException as e:
    logger.error(f"Processing failed: {e}")
    raise

# Gate pipeline activation
if not calibration_complete:
    raise RuntimeError("Cannot start: calibration incomplete")
```

### 3.3 `return None` — Decision Procedure

§3.1 forbids "silent return on missing data"; this section names the narrow cases where `return None` IS legitimate. They conflict on the surface — the procedure below resolves them.

For any `return None` you're about to write, ask:

> **"What does the caller treat `None` as — a legitimate empty result, or a signal that something it needed is missing?"**

- **Legitimate empty result** (`None` = "I looked, there was nothing") → `return None` is valid. The caller checks for `None` and handles it as a real outcome.
- **Missing-required-data signal** (`None` = "I couldn't get what you asked for") → FORBIDDEN. Raise instead.

If you can't answer with certainty about every caller, treat it as the second case (raise).

**Valid:** lookup/search that may legitimately find no match; optional enrichment data not available; test/diagnostic scripts.

**NEVER valid (each is a silent-plausibility instance):** preconditions not met (gate the pipeline at activation); hardware failure without consecutive error tracking; missing data in a required path; external call failed.

**When in doubt: raise.** Cheap to switch to `return None` later when the contract is confirmed; expensive to find every silent-return after a bug ships.

### 3.4 Forbidden Rationalizations

These all mask the same anti-pattern:

- "Data isn't ready yet, so we skip it" — **Don't start until data IS ready.**
- "Warming up / calibrating" — **Don't call until calibration IS done.**
- "Buffer is empty at startup" — **Don't start processing until the buffer has data.**
- "This is transient" — **Track consecutive failures. Declare dead after threshold.**

The correct pattern: **validate preconditions BEFORE activating the pipeline.** Once active, precondition failures are bugs.

### 3.5 Reviewing Scan Results

NEVER blanket-dismiss scan results. Review EACH match individually. If you believe a match is legitimate, get **explicit user confirmation** before dismissing. Logging an error and returning a fallback value is NOT fail-fast.

### 3.6 No Hardcoded Config Values (summary)

Every config-derived value comes from the loader — no language literals, no `const`, no env-var fallback for required values. The kill-shot anti-pattern: NEVER auto-justify "matches the YAML so it's safe" — every scan match is an explicit operator gate.

**Full rule + per-language scan templates:** [`modules/hardcoded-config.md`](./modules/hardcoded-config.md).

### 3.7 Validate CONTENT, Not Just HTTP Status (summary)

Every external-source consumer plausibility-checks the parsed value (range, sign, freshness, schema-shape) and raises on out-of-band. **"200 OK ≠ valid value."** Malformed responses raise; NEVER substitute zero / empty / default. Slow-moving inputs surface freshness metadata to the UI.

**Full rule + dimension checklist + cache TTL discipline:** [`modules/content-validation.md`](./modules/content-validation.md).

---

## 4. No Placeholders

Either implement fully or raise `NotImplementedError`. No `TODO` comments, no `FIXME`, no mock data in production code. No "would implement", "actual implementation", "in production" comment stubs. No "simplified for now" implementations.

**This applies to CODE only.** When a task requires product or design decisions, **ASK the user** rather than guessing.

---

## 5. Repository Boundaries

**NEVER modify code outside your assigned repository.**

If you find a bug in another repository: **STOP** → **DOCUMENT** (file, line, what's wrong, suggested fix) → **REPORT** to the user.

If a specification appears wrong: do NOT change your implementation to "fix" it. Report.

---

## 6. Destructive Operations

**NEVER perform destructive operations without EXPLICIT user permission:**

- Deleting files, databases, tables, buckets, or certificates.
- Revoking credentials or certificates.
- Modifying user accounts.
- Dropping cloud resources.
- Force-pushing or resetting git history.

When troubleshooting: **DIAGNOSE** root cause first, **ASK** before destructive actions, **VERIFY** assumptions before acting. "Deploy failed, let me delete and recreate the resource" is the wrong instinct.

---

## 7. Security

- **SQL Injection:** Use parameterized queries for ALL data values. No string formatting.
- **Credentials:** NEVER hardcode. Use IAM roles, environment variables, or secret management.
- **Auth failures:** Invalid token → 401 (no fallback). Missing permissions → 403 (no defaults). Missing config → startup failure.
- **Multi-tenant:** Every query must be scoped to the current tenant. RLS enforced at the database level.
- **Secrets:** Never in source control. Use secret management services.

### 7.1 Never Trust Client-Supplied Tenant Identifiers (summary)

Tenant / account identifier MUST come from the validated JWT or session claim, NEVER from query strings, request bodies, headers, or path params (except where the path param is then validated against the JWT before use). Persistence-layer code derives from the JWT-validated identifier. Log every cross-account access attempt.

**Full rule + defense-in-depth layers:** [`modules/tenant-identification.md`](./modules/tenant-identification.md).

### 7.2 Revocation Endpoints (summary)

Authenticate logout / password-reset-confirm / refresh-revoke / any "invalidate-me" endpoint with a credential that is STILL VALID when the user wants to walk away — gateway session cookie, refresh token, OR explicitly public + rate-limited. **NEVER gate revocation on the credential being revoked.**

**Full rule + per-endpoint correct auth table:** [`modules/revocation-endpoints.md`](./modules/revocation-endpoints.md).

---

## 8. Specification Compliance

Before writing ANY integration code (message structures, API schemas, protocol implementations):

1. **Read the relevant spec FIRST.**
2. **Copy field names EXACTLY.** Never guess.
3. **Add a spec reference comment** in code with file path and line numbers.
4. **Verify before commit.** Diff your field names against the spec.

If the spec appears wrong: report to the user — do not change your implementation to "fix" it.

**Full rule + bidirectional parity discipline + past violations:** [`modules/spec-compliance.md`](./modules/spec-compliance.md).

---

## 9. Code Conventions

- **Portable shell:** POSIX syntax (`counter=$((counter + 1))`), not bash-specific (`((counter++))`).
- **API responses:** Return empty arrays `[]`, never `null` for list endpoints.
- **Encoding:** Follow the project's encoding standard per channel; never mix encodings on the same channel.
- **Imports:** Match existing codebase patterns before introducing new ones.
- **No backwards compatibility code** (unless the story explicitly requires it): no fallback logic for old formats, no migration paths, no deprecated field handling. If the story is silent on compatibility: **ASK** rather than adding it speculatively.

### 9.1 Never Re-Implement Off-The-Shelf Primitives (summary)

If your project depends on an off-the-shelf service (identity provider, OAuth2 server, message broker, search engine) that provides a primitive natively, the application uses it directly. Never wrap, mirror, or re-implement. Acceptable application-side surface is limited to flow-orchestration glue, application-state mirrors of off-the-shelf events via webhooks, and validation of credentials issued by the primitive.

**Full rule + acceptable patterns + plan-step "native primitive check":** [`modules/primitive-reuse.md`](./modules/primitive-reuse.md).

### 9.2 Clean Logs Are a Hard Requirement (summary)

Every "ignored / deprecated / not supported" warning on a clean run is a defect — even when claimed "harmless." Remove the inert artifact AND verify the actual enforcement is in place; repoint any CI gate that was checking the inert field at the real layer in the same commit. Surface to the operator before removing when the change touches production-shape files.

**Full procedure + worked example:** [`modules/clean-logs.md`](./modules/clean-logs.md).

---

## 10. No Parallel Implementations — Extend Existing Anchors

When an LLD, epic, or story uses ANY of the phrases below about an existing function / file / package / component / hook / service / endpoint, the named existing artifact is the ANCHOR and the new work MUST modify the anchor in place:

- "extends X" / "extend X"
- "mirrors X" / "mirroring X" / "mirrors the structure of X"
- "wider input to X" / "same compute, wider input"
- "same compute, new entry point"
- "re-use X" / "re-used as-is" / "X is reused"
- "alongside X" (when describing extension, NOT orthogonal capability)
- "parallel to X" (when describing shape, NOT cardinality)
- "shares the structure of X"

Creating a parallel file / package / function / component alongside the anchor is a VIOLATION, **EVEN IF every line of the new artifact is otherwise correct.** The bug is that the new artifact exists; correctness of its contents is irrelevant.

**Per-story plan requirement:** every plan opens with an **"Existing-Primitive Analysis"** section naming the existing anchor (file:line) for every artifact OR justifying net-new under the narrow-exception list (genuinely different domain / distinct lifecycle entry point with shared internal package / no meaningful shared code post-extraction). Even under a justified exception, business logic MUST live in a single internal package both call.

**Per-story review requirement:** every reviewer verdict opens with a `Shape:` line answering "did the diff modify the anchor or introduce a parallel file?" Verdicts grading CONTENT without verifying SHAPE are INVALID.

**Full rule + trap-phrase translation + worked anti-example + scan:** [`modules/no-parallel-implementations.md`](./modules/no-parallel-implementations.md).

---

## 11. Surface Issues; Don't Bypass; Don't Paper Over

### 11.1 Fix Root Causes, Not Symptoms (summary)

When a bug is reported, trace it to the specific line / config / decision that causes it. **NEVER** propose compensating logic that masks the symptom — no retry loops, no timeout bumps, no warning suppression, no fallback-to-default, no cache-extension. If a fix is genuinely a third-party-code-only mitigation, label it "mitigation" explicitly in the commit message AND the LLD Risks section.

**Full rule + anti-pattern catalog + diagnostic question + mitigation discipline:** [`modules/root-causes-not-symptoms.md`](./modules/root-causes-not-symptoms.md).

### 11.2 Surface Host-Level Failures, Don't Pivot

When a host operation fails (sudo password prompt, port collision, write permission denied, missing apt package, blocked network rule), STOP and report the literal command + literal error. Do NOT pivot to an alternative path that masks the underlying issue — that's a §11.1 paper-over. The operator can usually fix host config in seconds; workarounds accumulate forever.

### 11.3 Logging Gap ≠ "Didn't Happen"

Absent audit row / log line is first a logger coverage gap; verify coverage BEFORE concluding the behavior didn't occur. Reach for §11.1 before any retry / timeout / fallback patch.

---

## 12. Your Role in Story Execution

You operate within a structured story-by-story process. Your role depends on where you are in the cycle.

### Authority Model (HARD RULES)

- **PLAN/PROPOSAL agents are PLAN-ONLY.** They produce written implementation proposals as text. They do NOT edit code. If a plan agent edits a file, STOP, revert, re-prompt with `RESEARCH-ONLY (do NOT edit files)`. Narrow carve-out for design-authoring agents (LLD authors, decision-log authors) within strictly scoped paths.
- **The IMPLEMENTING driver writes the code.** Do NOT delegate the implementation back to the plan agent.
- **REVIEW agents are READ-ONLY.** They produce PASS/BLOCK verdicts with file:line citations. A reviewer that grades the plan instead of the diff is invalid — re-run.
- **Every reviewer verdict opens with a `Shape:` line** (per §10).

### Story Cycle Steps

**(a) PLANNING:** plan agent generates an implementation plan. Plan opens with "Existing-Primitive Analysis" (§10). Plan covers ALL acceptance criteria. References official documentation (versioned URLs, not `/latest/`) and LLD sections with line numbers.

**(b) SANITY-CHECKING:** validate plan against story ACs + LLD + invariants. Confirm Existing-Primitive Analysis is present and unambiguous. Reject + re-plan if missing.

**(c) IMPLEMENTING:** follow the validated plan. STOP and ASK on multi-choice decisions, scope-creep risks, or anything unclear.

**(d) REVIEWING:** every verdict opens with a `Shape:` line per §10. Review against the **original story acceptance criteria**, not the plan. **Fan out in parallel:** all reviewer-agent invocations + the full CI/test/static-scan suite are dispatched in a SINGLE batch. NEVER run them sequentially.

**(e) FIXING & TESTING:** fix all issues found in review. Build and deploy. Run all completion verification scans (§13). Verify each AC with the specified command + capture actual output.

**(f) BRANCHING + COMMITTING + MERGING:** per-story feature-branch + PR flow. Each story = one atomic commit on its own branch = one PR with reviewer verdicts cited = one merge with branch deletion. NEVER push directly to `main`. NEVER batch multiple stories into one commit.

**Full command sequence + conflict resolution + commit/PR shape:** [`modules/per-story-pr-sequence.md`](./modules/per-story-pr-sequence.md).

### Workflow Rules

- **One story at a time.** Complete fully before starting the next.
- **Maximum 1-2 stories per session.** Larger scope leads to incomplete work.
- **Build and deploy after every story** to catch regressions.
- **Never mark complete without verification.** See §13.

---

## 13. Completion Verification

**Before marking ANY story or task as complete:**

### 13.1 Run Mandatory Scans

Every story-completion run executes the scan suite below. Each match is reviewed INDIVIDUALLY (§3.5).

| Scan category | What it catches |
|---|---|
| **Placeholder scan** | `TODO`, `FIXME`, `would.*implement`, `actual.*implementation`, "in production" stubs |
| **Fail-fast scan** | Silent error swallowing, default-value substitution in error paths, silent-return-on-missing-data (per language) |
| **SQL-injection scan** | String-formatted SQL queries |
| **Hardcoded-config scan** (§3.6) | Language literals outside the loader — every match is an explicit operator gate |
| **Static-scan baseline** | `shellcheck` / `yamllint` / `go vet` / `tsc --noEmit` / `ruff check` per language touched (HARD RULE — not optional polish) |
| **No-parallel-implementations scan** (§10) | Filename-pair (`<Prefix>X` next to `X`); parallel hook/component trees; trap-phrase audit on LLDs |
| **Forbidden-library scan** | Project-specific list of libraries that must not appear |
| **Spec round-trip** (§8) | Wire-contract regeneration leaves zero diff vs the committed spec |

**Concrete commands per language + run order:** [`modules/scan-templates.md`](./modules/scan-templates.md).

**Fan-out:** every scan runs in the SAME parallel batch as the reviewer agents (step d) — never chain serially.

**For EACH match:** is it a genuine violation? If in doubt, it IS a violation — fix it. If you believe it's legitimate, get **explicit user confirmation** before dismissing. Silent-plausibility (§0) alert: "all 12 matches happen to equal the YAML so they're safe" is the kill-shot — every match must come from the loader regardless of whether the literal happens to be correct today.

### 13.2 Verify Each Acceptance Criterion

Run the concrete verification command specified in the story. Capture **actual output** (not "expected output"). If you can't demonstrate it works, it is **NOT complete**.

**Static analysis is NOT verification.** Grepping files, reading code, and code review don't count. Only actual build, run, deploy, and test execution counts.

**Verification ≠ HTTP 200.** Green build + green healthcheck are necessary but NOT sufficient. For any story touching external data or business calculations, additionally run a content-plausibility check on at least one realistic input.

### 13.3 Build & Deploy

After every story that touches deployed code: build → deploy to test environment → smoke test → confirm nothing broke.

**End-of-Epic Runtime Test (HARD RULE).** After the LAST story of any epic commits + merges, BEFORE the next epic's planning begins, run the FULL epic-level integration on a real environment: provision missing tooling per pinned versions; full deploy + smoke; exercise the paths the epic claims to deliver; paste actual output, not expected. Don't stop at "compose config validates."

### 13.4 Provide Proof

Your completion report MUST include: acceptance-criteria verification (command + actual output per AC); scan results (output of all mandatory scans); build/deploy proof; files modified list with line counts.

---

## 14. When to Escalate

**ALWAYS stop and ask the user when:**

| Situation | Why |
|-----------|-----|
| Multi-choice design decisions arise | You should not make product decisions |
| Changes might impact logic or features beyond this story | Scope creep prevention |
| A blocking review finds violations | Human decides: fix, override, or defer |
| A specification appears wrong or incomplete | Cross-team coordination needed |
| A scan result seems legitimate but you're not sure | Don't self-rationalize dismissals (§3.5) |
| The task requires a product or design decision | Don't guess — ASK |
| You're uncertain about a convention or pattern | Read codebase first, then ask if still unclear |
| Implementation requires adding a new dependency | Human approves library choices |
| Version-pin bump needed | Bumps break env vars / CLI flags / config keys across minor versions |
| Host operation fails | §11.2 — surface, don't pivot |
| You'd consider a retry / timeout / fallback / cache-extension fix | High-probability paper-over (§11.1) |
| Tool warns a config field is "ignored" or "deprecated" | §9.2 — surface BEFORE removing |
| Bug rooted in an architectural / design choice | Flag `ARC REVIEW REQUIRED`; do NOT apply a code-level fix |
| LLD/epic uses a trap-phrase without an anchor file path | §10 — redraft before implementing |

**Per-Epic Version-Pin Protocol.** LLDs name dependencies but do NOT pin versions at design time. The owning epic's Story 0 walks every dependency in the LLD: if a pin exists in your version pin file, use it verbatim; else pin latest stable as of story start, capture the changelog summary in the AC, surface the set to the operator for approval BEFORE implementation stories proceed.

**NEVER do this:** make a decision, rationalize it, and move on. That is the single most damaging pattern in AI-assisted development. When in doubt, **ask**.

---

*These rules are non-negotiable. They exist because their absence caused real failures. Follow them exactly, and ask the user if anything is unclear.*
