# [EPIC] LLD #<XX.Y>: <LLD Title> — <Scope Name>

**Epic ID:** EPIC_<XX.Y>
**File Name:** `EPIC_<XX.Y>_<Scope_Name>.md`
**Implementation Domain:** <Frontend | Backend | Mobile | Infrastructure | CI-CD>
**Assigned Team:** <Frontend Team | Backend Team | Mobile Team | Platform Team | DevOps>
**LLD Reference:** [LLD #<XX> — <LLD Title>](../design/llds/LLD_<XX>_<filename>.md) (lines XXX-YYY)
**Spec Reference (REQUIRED for integrations/APIs/messaging):** [specs/api/<SPEC>.md](../../specs/api/<SPEC>.md) (section X.Y, lines XXX-YYY) OR `No spec yet; user approval required before implementation.`
**PR Scope:** <Brief description of what this epic covers>
**Estimated Complexity:** <Small/Medium/Large>
**Story Count:** <2-5 stories (MAXIMUM)>
**Dependencies:** <List of blocking LLDs or epics>

---

## Critical Implementation Principles

**These principles are NON-NEGOTIABLE and MUST be followed:**

### 1. NO Fallbacks or Default Values
- **NEVER provide fallback values when configuration is missing.**
- Fail fast and fail loud with clear error messages.
- If a required value is missing, throw an error immediately.

```typescript
// WRONG — silent fallback
const dbUrl = process.env.DATABASE_URL || "postgres://localhost/default"; // FORBIDDEN

// CORRECT — fail fast
const dbUrl = process.env.DATABASE_URL;
if (!dbUrl) throw new Error("DATABASE_URL environment variable is required");
```

### 2. NO Placeholders or Mock Data
- Either implement fully or throw `NotImplementedError`.
- NEVER return mock/dummy data. NEVER use TODO comments in place of implementation.

```typescript
// WRONG — placeholder
async function analyzeUsage(data: UsageData) {
  // TODO: Implement proper analysis
  return { score: 0.5, status: "ok" }; // FORBIDDEN
}

// CORRECT — explicit error or full implementation
async function analyzeUsage(data: UsageData) {
  throw new Error("analyzeUsage not yet implemented");
}
```

### 3. NO Error Obfuscation
- NEVER catch and suppress errors to "make things work."
- NEVER use empty catch blocks.
- Let errors propagate with full context.

```typescript
// WRONG — swallowed error
try { await riskyOperation(); } catch (e) { /* ignore */ } // FORBIDDEN

// CORRECT — propagate after logging
try {
  await riskyOperation();
} catch (error) {
  logger.error("Operation failed", { error });
  throw error;
}
```

### 4. NO Simplified Implementations
- Full implementation or nothing. No "simplified version for now."

### 5. MANDATORY Verification Commands
- Every acceptance criterion MUST have a concrete verification command.
- The command MUST be runnable and produce observable proof.
- "Trust but verify" — if you can't prove it works, it doesn't work.

### 6. NO Migration or Backwards Compatibility
- This is an MVP — breaking changes are acceptable.
- NO migration paths, NO compatibility layers, NO version bridging.

### 7. MANDATORY Spec Compliance
**If this epic involves API endpoints, WebSocket events, or messaging/integration contracts:**
- [ ] **Spec Reference Filled In:** The field above links to the relevant spec
- [ ] **No Silent Guessing:** If no approved spec exists, stop and get user approval before implementation
- [ ] **Field Names Copied from Spec:** Every field name in code matches spec exactly
- [ ] **Spec Comment in Code:** Each handler has a comment citing spec file and lines

### 8. ALL DECISIONS MUST BE IN EPIC (Traceability)
- Make ALL technical decisions during planning and document them here.
- NO "Option A / Option B" left unresolved — pick one and document WHY.
- During implementation: follow the epic exactly. If unclear: STOP > ASK > UPDATE EPIC > CONTINUE.

### 9. NO PARALLEL IMPLEMENTATIONS — Extend Existing Anchors (HARD RULE)
- Every story in this epic that produces ANY artifact (file / function / package / component / hook / endpoint / migration) MUST fill out the per-story **Existing-Primitive Analysis** table below.
- Trap-phrases ("mirrors X", "wider input to X", "same compute new entry point", "re-use X as-is", "extends X", "alongside X", "parallel to X", "shares the structure of X") MUST name the anchor X **by file path** on the same line.
- The default action is **extend the anchor in place**. Parallel files / packages / components / hooks alongside an existing anchor are violations EVEN IF the new artifact is otherwise correctly implemented — the bug is that the new artifact EXISTS.
- Net-new artifacts MUST be justified under the narrow-exception list (genuinely different domain / distinct lifecycle entry point / no meaningful shared code post-extraction). Even under a justified exception, business logic MUST live in a single internal package both call — NEVER copy-pasted.
- Plans missing the Existing-Primitive Analysis table OR reviewer verdicts missing a `Shape:` line are rejected.

---

## Domain Separation Requirement

**Each epic MUST target ONLY ONE implementation domain.**

Valid domains: Frontend, Backend, Mobile, Infrastructure, CI-CD.

Multi-domain epics must be split. Different teams work on different domains with different review processes and testing environments.

---

## Epic Summary

<2-3 sentence summary of what this epic delivers, focusing on business/technical value.>

### Scope

**In Scope:**
- <Major component/feature 1>
- <Major component/feature 2>

**Out of Scope:**
- <Explicitly excluded items to avoid scope creep>

### Success Criteria

- [ ] All user stories completed and acceptance criteria met
- [ ] Unit tests achieve >80% coverage for new code
- [ ] Integration tests validate end-to-end workflows
- [ ] Documentation updated (inline comments, README if applicable)
- [ ] Observability instrumented (metrics, logs)

---

## User Stories

### Story 1: <Story Title>

**As a** <role/component>
**I want** <capability>
**So that** <business value>

**Existing-Primitive Analysis (HARD RULE — fill BEFORE implementation):**

For every artifact this story produces (file / function / package / component / hook / endpoint / migration), name its existing anchor — OR justify net-new under the narrow-exception list. Trap-phrases without a file-path anchor are violations.

| Proposed Artifact | Action | Existing Anchor (file:line) | Diff Shape / Net-new Justification |
|-------------------|--------|-----------------------------|------------------------------------|
| `src/<path>/<file>` (or function name) | extend / extend-in-place / net-new | `src/<path>/<anchor>:L<line>` OR "net-new" | "Adds `<param>` at function entry; branches at L<line>" / "Net-new: <one-paragraph narrow-exception justification>; shared logic lives in `src/<path>/<core>/`" |

**Acceptance Criteria (with Verification Commands):**

| # | Criterion | Verification Command | Expected Output |
|---|-----------|---------------------|-----------------|
| 1 | <Specific, testable criterion> | `<command to prove it works>` | <what success looks like> |
| 2 | <Specific, testable criterion> | `<command to prove it works>` | <what success looks like> |

**Implementation Tasks:**
- [ ] Task 1.1: <Specific technical task>
- [ ] Task 1.2: <Specific technical task>

**LLD Traceability:**
- Section: §<X.X> <Section name>
- File: `LLD_<XX>_<filename>.md` (lines XXX-YYY)

**Estimated Effort:** <hours or story points>

---

### Story 2: <Story Title>

**As a** <role/component>
**I want** <capability>
**So that** <business value>

**Existing-Primitive Analysis (HARD RULE — fill BEFORE implementation):**

For every artifact this story produces (file / function / package / component / hook / endpoint / migration), name its existing anchor — OR justify net-new under the narrow-exception list. Trap-phrases without a file-path anchor are violations.

| Proposed Artifact | Action | Existing Anchor (file:line) | Diff Shape / Net-new Justification |
|-------------------|--------|-----------------------------|------------------------------------|
| `src/<path>/<file>` (or function name) | extend / extend-in-place / net-new | `src/<path>/<anchor>:L<line>` OR "net-new" | "Adds `<param>` at function entry; branches at L<line>" / "Net-new: <one-paragraph narrow-exception justification>; shared logic lives in `src/<path>/<core>/`" |

**Acceptance Criteria (with Verification Commands):**

| # | Criterion | Verification Command | Expected Output |
|---|-----------|---------------------|-----------------|
| 1 | <Specific, testable criterion> | `<command to prove it works>` | <what success looks like> |
| 2 | <Specific, testable criterion> | `<command to prove it works>` | <what success looks like> |

**Implementation Tasks:**
- [ ] Task 2.1: <Specific technical task>
- [ ] Task 2.2: <Specific technical task>

**LLD Traceability:**
- Section: §<X.X> <Section name>
- File: `LLD_<XX>_<filename>.md` (lines XXX-YYY)

**Estimated Effort:** <hours or story points>

---

## Technical Notes

### Architecture Decisions
<Key architectural decisions specific to this epic. Reference LLD sections.>

### Interface Contracts
<Critical API/message contracts from LLD §3.>

### Performance Requirements
<Latency/throughput budgets from LLD §6.>

### Failure Modes
<Failure scenarios and mitigations from LLD §7.>

---

## Testing Strategy

### Unit Tests
- [ ] Test suite for <component 1>
- [ ] Test suite for <component 2>
- [ ] Edge case coverage: <specific edge cases>

### Integration Tests
- [ ] End-to-end workflow: <scenario 1>
- [ ] End-to-end workflow: <scenario 2>
- [ ] Failure injection: <failure scenario>

### Validation Criteria

| Metric | Pass Criteria |
|--------|--------------|
| Latency | <target> |
| Throughput | <target> |
| Error rate | <target> |

---

## Observability

### Metrics
- `<metric_name>`: <description, units, thresholds>

### Logs
- `<log_event>`: <when emitted, severity>

---

## Dependencies & Blockers

**Upstream Dependencies:**
- [ ] LLD #<XX>: <dependency> — <what we need from it>

**External Dependencies:**
- [ ] Library/framework: <name, version from VERSIONS.yaml>

**Known Blockers:**
- <Blocker and mitigation plan>

---

## Definition of Done

### PRE-COMPLETION CHECKLIST (MANDATORY)

**Before marking this epic/story complete, you MUST complete these steps:**

#### Runtime Verification Declaration

**You MUST explicitly state what you DID and DID NOT verify at runtime.**

- [ ] Application/service actually runs (not just compiles)
- [ ] Feature-specific verification commands from acceptance criteria executed

**Static analysis is NOT verification.** Grepping files, reading code, and code review don't count. Only actual runtime execution counts.

---

- [ ] **Placeholder Scan Passed:**
  ```bash
  grep -rn "placeholder\|TODO\|FIXME\|would.*implement\|actual.*implementation" \
    <modified_directories> --include="*.ts" --include="*.tsx" --include="*.js"
  ```
  **Result:** _paste output here (should be empty)_

- [ ] **Fail-Fast Violation Scan Passed:**
  ```bash
  # Empty catch blocks
  grep -rn "catch.*{}" <modified_files> --include="*.ts" --include="*.js"
  # Silent fallback defaults
  grep -rn '|| ""' <modified_files> --include="*.ts" --include="*.js"
  grep -rn "?? null" <modified_files> --include="*.ts" --include="*.js"
  ```
  **For each match:** Review INDIVIDUALLY. If you believe it's legitimate, get explicit user confirmation.
  **Result:** _paste output and per-match justification_

- [ ] **All Verification Commands Executed:**
  | Criterion | Command Run | Output/Result |
  |-----------|-------------|---------------|
  | AC-1 | `<command>` | <result> |
  | AC-2 | `<command>` | <result> |

- [ ] **Files Modified Summary:**
  ```
  <file_path> (+XX, -YY lines)
  ```

### Spec Compliance Verification (API Epics)

**If this epic involves API endpoints, verify:**

- [ ] **Spec Reference Verified:** Spec section exists and matches implementation
- [ ] **Field Names Match Spec:** All request/response field names match exactly
- [ ] **Endpoint Registered:** Route matches spec path and HTTP method

### Standard Definition of Done

- [ ] All user stories completed with acceptance criteria met
- [ ] All verification commands executed with proof recorded
- [ ] Spec compliance verified (for API epics)
- [ ] Code reviewed
- [ ] Unit tests pass with >80% coverage
- [ ] Integration tests pass
- [ ] Performance benchmarks met
- [ ] Documentation updated
- [ ] Merged to main branch

### Session Scope Limits

**Maximum 1-2 stories per session.** If this epic has more stories, split implementation across multiple sessions.

---

## References

- **LLD:** [LLD #<XX> — <Title>](../design/llds/LLD_<XX>_<filename>.md)
- **HLD:** [HLD v1.0.0](../design/SAMPLE_HLD.md) §<X.X>
- **Related Epics:** <Links to dependent or related epics>
