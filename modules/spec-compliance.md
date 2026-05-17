# Module: Specification Compliance

**Load when:** you're writing integration code that touches a shared message format, API contract, or wire schema.

**Core rule lives in:** [`AGENT-INSTRUCTIONS.md` §8](../AGENT-INSTRUCTIONS.md#8-specification-compliance).

**Silent-plausibility variant fought:** code that compiles and runs but uses field names that differ from the spec — looks correct, breaks integration silently when the consumer parses the message.

---

## The Rule

Before writing ANY integration code (message structures, API schemas, protocol implementations):

1. **Read the relevant spec FIRST.** Find your topic/endpoint in the spec file.
2. **Copy field names EXACTLY.** NEVER guess — copy from spec verbatim.
3. **Add a spec reference comment** in code with file path and line numbers.
4. **Verify before commit.** Diff your field names against the spec.

If the spec appears wrong: do NOT change your implementation to "fix" it. Report to the user.

## Worked Example

```go
// Per specs/mqtt/SYNC_PROTOCOL.md lines 210-222:
// patrol_id, name, waypoint_ids (not full waypoints)
type PatrolPayload struct {
    PatrolID    string   `json:"patrol_id"`   // NOT "id"
    Name        string   `json:"name"`
    WaypointIDs []string `json:"waypoint_ids"` // NOT []Waypoint
}
```

The spec-reference comment matters: when someone reads this code in six months, they need to know whether the field name `patrol_id` was a choice or copied from a spec. The comment is the audit trail.

## Past Violations That Caused Integration Failures

- `id` vs `patrol_id` mismatch — consumer couldn't parse messages because the producer named the field `id` while the spec said `patrol_id`. Producer-side tests passed; consumer-side tests failed; integration broke in production.
- `waypoints` (full objects) vs `waypoint_ids` (references only) — wrong data structure sent, consumer's schema validation rejected the message, retries piled up, queue blocked.
- Status enum case mismatch (`"Active"` vs `"active"`) — looks identical to a casual reader; consumer's parser fails the case-sensitive string match.

All three would have been caught by a code-vs-spec diff at commit time.

## Bidirectional Parity

For specs that have a code-generated companion (OpenAPI from swaggo, protobuf, GraphQL schema), parity is BIDIRECTIONAL:

1. **Spec → code:** the generated client/server matches the spec.
2. **Code → spec:** regenerating the spec from code annotations produces no diff vs the committed spec.

If either direction has drift, integration is fragile. CI should fail on `git diff --exit-code <spec-file>` after a regenerate run.

## Change Process

If a spec needs to change to match what the code already does, that's a separate conversation:

1. Open a spec proposal (under `specs/_proposals/` or your project's equivalent).
2. The spec owner reviews.
3. Both producer and consumer teams approve.
4. Merge spec change.
5. Update implementations to match.

NEVER do it backwards: changing your code to match what you think the spec should say, before the spec change is approved, ships a non-spec-compliant artifact under the cover of "we'll fix the spec later."

## Field Naming Discipline

When the spec is the source of truth:

- **Don't translate field names** between layers. If the spec says `patrol_id`, your DB column is `patrol_id`, your Go struct is `PatrolID` (with `json:"patrol_id"` tag), your TypeScript type is `patrol_id` (or `patrolId` only if your project's TS convention does that translation centrally). Per-place translation drift is how `id` vs `patrol_id` happens.
- **Don't add fields that aren't in the spec.** Untyped `metadata` / `extra` / `attrs` blobs are forbidden — they let consumers misinterpret payloads. If you need a field, propose it through the spec change process.
- **Don't repurpose existing fields.** Using `description` to also carry a serialized JSON blob "for now" is a future bug.
