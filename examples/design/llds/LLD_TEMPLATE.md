# LLD #<XX>: <Component Title>

**Version:** <X.Y.Z>
**Status:** Draft | Review | Approved
**Last Updated:** <YYYY-MM-DD>
**Owner:** <Team or individual>
**HLD Reference:** §<X.X> <Section Title>

---

## 1. Purpose and Scope

### What This Component Implements
<Brief description of what this component does.>

### What It Does NOT Implement
<Explicit exclusions to prevent scope creep.>

### HLD References
- §<X.X> <Section Title> — <what aspect>
- §<X.X> <Section Title> — <what aspect>

---

## 2. Context and Dependencies

### Upstream Dependencies

| Component | What It Provides | Interface |
|-----------|-----------------|-----------|
| <Component> | <Data/service it provides> | <API/protocol> |

### Downstream Consumers

| Component | What It Consumes | Interface |
|-----------|-----------------|-----------|
| <Component> | <Data/service it uses from us> | <API/protocol> |

### External Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| <Library/service> | <version from VERSIONS.yaml> | <why needed> |

### 2.4 Existing-Primitive Map (No Parallel Implementations — HARD RULE)

**Every artifact this LLD adds MUST be named with its anchor** — the
existing file/function/package/component it extends. The agent rule
"No Parallel Implementations" (see Agent Instructions §10) translates
to a documentation requirement here: trap-phrases ("mirrors X", "wider
input to X", "same compute new entry point", "re-use X as-is",
"extends X", "alongside X", "parallel to X", "shares the structure
of X") MUST name the anchor BY FILE PATH on the same line.

Net-new artifacts MUST be justified under the narrow-exception list
(genuinely different domain / distinct lifecycle entry point with a
shared internal package / no meaningful shared code post-extraction)
AND state where the shared business logic lives if ANY logic is
shared with an existing artifact.

LLDs missing this table — OR using trap-phrases without an anchor
file path — should be rejected at LLD review. The implementer-agent
that later picks up the epic inherits this table as the basis for its
own per-story Existing-Primitive Analysis; if this table is missing
or ambiguous, the downstream implementation plan is impossible to
write correctly.

| Proposed Artifact | Action | Existing Anchor (file:line) | Diff Shape / Net-new Justification |
|-------------------|--------|-----------------------------|------------------------------------|
| `src/path/file.ext` (or function name) | extend / extend-in-place / net-new | `src/path/anchor.ext:L123` OR "net-new" | "Adds `scope` parameter at function entry; branches at line L45" / "Net-new: distinct lifecycle entry point (HTTP handler vs batch worker); shared logic lives in `src/path/core/`" |

**Worked example:**

A "bulk Foo" feature adds an aggregated entry point that processes
many Foos in one call. The LLD says "Feeds the existing Foo processor
ONE wider input vector (same compute, just a wider input)."

| Proposed Artifact | Action | Existing Anchor | Diff Shape |
|-------------------|--------|-----------------|------------|
| Bulk Foo processor entry | extend-in-place | `src/processor/foo.ts:155` | Add `processFooBatch(items)` top-level function in SAME file delegating to internal `processCore` which accepts a single-or-multi item input; `processFoo(item)` and `processFooBatch(items)` both call `processCore`. NO parallel `bulk-foo.ts`. |
| Bulk Foo page render | extend-in-place | `src/web/pages/Foo.tsx:200` | Add `scope?: "single" \| "bulk"` prop; branch the hook selection at the top of the component; route table mounts the same component for both `/foo/:id` and `/foo/bulk`. NO new file. |

**Forbidden shape** (what real projects ship when this table is
missing): a separate `BulkFoo.tsx` next to `Foo.tsx` (near-clone with
one swapped hook), a separate `bulk-foo.ts` next to `foo.ts` (parallel
processor re-implementing the validation + fan-out logic from
scratch). Every reviewer that grades the new file in isolation passes
it because the file IS correctly implemented — the bug is that it
exists at all.

---

## 3. Interfaces and Contracts

### 3.1 API Endpoints (if applicable)

```
<HTTP method> <path>
  Request:  <schema or TypeScript type>
  Response: <schema or TypeScript type>
  Errors:   <error codes and meanings>
```

### 3.2 Events / Messages (if applicable)

```
Event: <event_name>
  Payload: <schema>
  Frequency: <rate>
  Publisher: <who sends>
  Subscribers: <who receives>
```

### 3.3 Error Model

<How errors are represented, propagated, and handled at boundaries.>

---

## 4. Data Models and Persistence

### 4.1 Database Schema

```sql
-- <Table name and purpose>
CREATE TABLE <table_name> (
    <columns with types and constraints>
);
```

### 4.2 Indexes

| Table | Index | Columns | Purpose |
|-------|-------|---------|---------|
| <table> | <index_name> | <columns> | <why needed> |

### 4.3 Retention and Lifecycle

| Data Type | Retention | Archive Strategy |
|-----------|-----------|------------------|
| <type> | <period> | <what happens after> |

---

## 5. Detailed Design

### 5.1 Architecture

```
<Component-internal architecture diagram.
Show internal modules, data flow, key abstractions.>
```

### 5.2 Algorithms / State Machines

<Describe key algorithms, state transitions, or decision logic.>

### 5.3 Resource Budgets

| Resource | Budget | Measurement |
|----------|--------|-------------|
| CPU | <target> | <how measured> |
| Memory | <target> | <how measured> |
| Disk | <target> | <how measured> |
| Network | <target> | <how measured> |

### 5.4 Concurrency Model

<Thread/process model, connection pooling, backpressure strategy.>

### 5.5 Configuration

| Config Key | Type | Default | Description |
|------------|------|---------|-------------|
| <key> | <type> | <value or REQUIRED> | <purpose> |

---

## 6. Timing and Performance

### 6.1 Latency Budgets

| Operation | Target (p50) | Target (p95) | Target (p99) |
|-----------|-------------|-------------|-------------|
| <operation> | <ms> | <ms> | <ms> |

### 6.2 Throughput

| Metric | Steady State | Peak |
|--------|-------------|------|
| <metric> | <rate> | <rate> |

---

## 7. Reliability and Failure Handling

### 7.1 Failure Modes

| Failure | Detection | Impact | Mitigation |
|---------|-----------|--------|------------|
| <what can fail> | <how detected> | <user impact> | <recovery action> |

### 7.2 Retry and Backoff

<Retry strategy for transient failures. Which operations are idempotent?>

---

## 8. Security and Privacy

### 8.1 Authentication / Authorization

<How this component validates identity and permissions.>

### 8.2 Data Classification

| Data Element | Classification | Handling |
|-------------|---------------|----------|
| <field> | <PII / Confidential / Public> | <encryption, masking, etc.> |

---

## 9. Invariant Compliance

Map to HLD §<X> Critical Invariants:

| Invariant | How This Component Complies | Verification |
|-----------|---------------------------|--------------|
| <invariant from HLD> | <how enforced here> | <test or check> |

---

## 10. Observability

### Metrics

| Metric Name | Type | Labels | Alert Threshold |
|-------------|------|--------|----------------|
| <metric> | Counter/Gauge/Histogram | <labels> | <when to alert> |

### Logs

| Event | Level | When Emitted |
|-------|-------|-------------|
| <event> | INFO/WARN/ERROR | <condition> |

### Health Checks

| Check | Endpoint | Criteria |
|-------|----------|----------|
| Liveness | <path> | <what it checks> |
| Readiness | <path> | <what it checks> |

---

## 11. Validation and Acceptance

### Test Cases

| Test | Input | Expected Output | Type |
|------|-------|-----------------|------|
| <test> | <input> | <expected> | Unit/Integration |

### Benchmarks

| Benchmark | Pass Criteria |
|-----------|--------------|
| <benchmark> | <threshold> |

---

## 12. Operational Procedures

### Maintenance

<Recurring maintenance tasks: log rotation, cache cleanup, etc.>

### Runbook

<Key operational procedures: restart, failover, data recovery.>

---

## 13. Risks and Open Questions

| Risk / Question | Severity | Mitigation / Status |
|----------------|----------|---------------------|
| <risk> | High/Medium/Low | <mitigation or "Open"> |

---

## Appendices

<Diagrams, sequence charts, example payloads, glossary.>
