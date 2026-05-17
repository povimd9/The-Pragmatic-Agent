# Module: No Parallel Implementations — Extend Existing Anchors

**Load when:** an LLD / epic / story uses any anchor phrase ("extends X", "mirrors X", "wider input to X", "same compute new entry point", "re-use as-is", "alongside X", "parallel to X", "shares the structure of X") OR you're reviewing a diff that adds new files alongside existing ones with similar names.

**Core rule lives in:** [`AGENT-INSTRUCTIONS.md` §10](../AGENT-INSTRUCTIONS.md#10-no-parallel-implementations--extend-existing-anchors).

**Silent-plausibility variant fought:** new files that duplicate an anchor — content-correct in isolation, structurally wrong because the anchor already exists.

---

## The Rule

When an LLD/epic/story names an existing function / file / package / component / hook / service / endpoint as the anchor for new work, that anchor MUST be modified in place. Creating a parallel artifact alongside the anchor is a VIOLATION, **EVEN IF every line of the new artifact is otherwise correct.** The bug is that the new artifact exists; correctness of its contents is irrelevant.

## Forbidden Patterns

- `src/pages/X.tsx` next to `src/pages/<Scope>X.tsx` where the second is a near-copy with a different hook.
- `src/components/foo/` next to `src/components/<scope>/` containing parallel components for the same role.
- `src/api/hooks/foo.ts` next to `src/api/hooks/<scope>/foo.ts` containing parallel hooks.
- `services/<svc>/internal/<pkg>/worker.go` next to `services/<svc>/internal/<pkg>/<scope>.go` containing a second orchestrator that duplicates fan-out / phase / tolerance logic.
- A second router-middleware stack for the same logical endpoint where the only difference is auth shape.
- A second migration sequence / schema / DB-helper for the same logical entity differing only by scope.

## What "Extend In-Place" Concretely Means

1. The anchor takes a new parameter (`scope`, `tenant`, `source`, etc.) OR a new top-level function in the SAME file delegating to the same internal logic.
2. Branching by parameter happens at the SMALLEST possible scope (typically one if/switch at one function's top, NOT a whole new file).
3. Tests cover both branches in the SAME test file.
4. The diff shape is MODIFICATIONS to the anchor + minimal additions, NOT a brand-new file of similar size to the anchor.

## The Narrow Exception List

Parallel artifacts are justified ONLY when:
- The domain is genuinely different (e.g., identity-provider glue vs business primitives).
- The artifact serves a different lifecycle entry point (batch worker vs HTTP handler) — distinct entry points are allowed, but the business logic they share MUST live in a single internal package both call.
- The two artifacts share zero meaningful code after extraction; parametrization would produce a god-function with mutually exclusive branches.

Even under a justified exception, **both artifacts MUST refer to a SHARED INTERNAL function/package for the actual business logic.** Copy-paste of business logic is NEVER acceptable.

## Trap-Phrase Translation

Read EVERY occurrence in an LLD/epic/story this LITERALLY:

| LLD/epic/story English | Concrete instruction |
|---|---|
| "Mirrors existing X" | MODIFY X. Do not create `<Scope>X`. |
| "wider input to existing X" | MODIFY X to accept the wider input. |
| "same compute, new entry point" | ADD a top-level function IN X. Do not create a sibling file. |
| "re-use X as-is" | IMPORT and USE X. Do not re-implement X under a new name. |
| "extends X" | MODIFY X. |
| "alongside X" (extension context) | If the new capability shares any internal logic with X, the shared logic goes in an INTERNAL package both call. NEVER two parallel artifacts containing the same logic. |

If you read a story phrase and your first instinct is "create a new file that looks like X" — STOP. Reread the rule. The default is always to extend X.

## Plan & Review Requirements (HARD RULES)

**Per-story plan-output requirement.** Every implementation plan MUST include an **"Existing-Primitive Analysis"** section as the FIRST substantive section, BEFORE proposing any file. For every artifact:

- Named existing equivalent (file:line if applicable), OR explicit "net-new" with one-paragraph justification per the narrow-exception list.
- For "extends": the diff shape proposed — function-signature change, scope parameter, branch point.
- For "net-new": grep evidence confirming no existing artifact serves the same role.

Plans missing this section are rejected at sanity-check.

**Per-story review requirement.** Every reviewer verdict MUST open with a `Shape:` line answering:
- "Did the diff modify the anchor named in the story, or introduce a parallel file/package?"
- "If parallel: does the story's Existing-Primitive Analysis explicitly justify it, AND does shared logic live in a single internal package both call?"

Verdicts grading CONTENT (placeholders / fail-fast / SQL / decimal / etc.) without verifying SHAPE are INVALID — re-run.

### Shape: Line Templates

The `Shape:` line is the reviewer's first sentence. It either passes the shape check or blocks the merge — content findings come after. Sample shapes:

**PASS — anchor modified in place:**
```
Shape: PASS — anchor `pkg/foo/foo.go::Foo` modified; new entry point `FooBatch` added in same file, delegates to internal `fooCore`. No parallel file introduced.
```

**PASS — narrow exception justified:**
```
Shape: PASS — net-new file `pkg/foo/scheduled/foo.go` introduced; justified per story's Existing-Primitive Analysis as distinct lifecycle entry point (cron worker vs HTTP handler). Shared business logic lives in `pkg/foo/internal/core/` which both call. Narrow-exception argument verified at story plan §X.
```

**BLOCK — parallel file:**
```
Shape: BLOCK — diff introduced `pkg/foo/bar/foo.go` (438 LOC) alongside anchor `pkg/foo/foo.go` (712 LOC). Story's Existing-Primitive Analysis names the anchor as the extension target ("wider input to existing foo.go") but the diff created a sibling instead of extending. Re-implement as `FooBatch` in the anchor; share logic via internal `fooCore`. Content review deferred until shape is correct.
```

**BLOCK — net-new without justification:**
```
Shape: BLOCK — diff introduced `src/components/admin/UserList.tsx`; no Existing-Primitive Analysis row covers this artifact. Either name the existing anchor (likely `src/components/UserList.tsx` per the scope-prefix scan) and extend it, or add a narrow-exception justification to the story plan. Content review deferred.
```

The pattern: one sentence stating PASS or BLOCK + the anchor + the diff shape. Content findings follow in subsequent sections of the verdict.

## Worked Anti-Example

A team's epic said: "Feeds the existing Foo processor ONE wider input vector (same compute, no algorithmic change ... just a wider input)" — anchor `pkg/foo/foo.go`. The implementer read "new entry point" as "new package + parallel file" and produced `pkg/foo/bar/foo.go` (472 LOC) next to `pkg/foo/foo.go` (713 LOC), re-writing the input-validation logic from scratch and silently missing one validation step. Every reviewer passed the new file in isolation because it was correctly implemented in isolation; none flagged that the same function existed 12 lines above. Latent bug shipped to main. **The bug was that the new file existed at all.**

Same pattern on the frontend: a `BarFoo.tsx` (344 LOC) shipped next to `Foo.tsx` (482 LOC) as a near-clone with one swapped hook + a different icon. Six sibling parallel pages + a parallel hook tree + a parallel component dir all landed without a single reviewer asking "should this file exist?"

The right shape would have been: `Foo` and `FooBatch` as two top-level functions in `pkg/foo/foo.go`, each delegating to an internal `fooCore` that accepts a single-or-multi item input. On the frontend, `Foo.tsx` takes a `scope?: "single" | "bulk"` prop, and the route table mounts the same component for both `/foo/:id` and `/foo/bulk`. No parallel files, no copy-pasted logic, one source of truth.

## Detection Scan

See [`scan-templates.md`](./scan-templates.md) for sample shapes (filename-pair scan + trap-phrase audit). Both samples need calibration: extend the prefix list (`Admin`, `Bulk`, `Tenant`, …) to match the scope prefixes your project actually uses, and extend the trap-phrase verb list (`mirrors`, `wider input`, `same compute`, …) to match your LLD authors' vocabulary (`expands`, `broadens`, `another flavor of`, …). The samples teach the technique; the patterns are project-specific.
