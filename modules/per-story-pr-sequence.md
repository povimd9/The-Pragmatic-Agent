# Module: Per-Story Branch + PR + Merge Sequence

**Load when:** you're at step (f) of the story cycle — branching, committing, and merging a completed story.

**Core rule lives in:** [`AGENT-INSTRUCTIONS.md` §12](../AGENT-INSTRUCTIONS.md#12-your-role-in-story-execution) (step f).

**Silent-plausibility variant fought:** "I'll commit it all together at the end" batched commits that bury individual story changes in a too-big diff no human reviewer parses carefully.

---

## The Sequence

```bash
# 0. Start from a clean tree
git status   # should be clean before you begin

# 1. Fetch + branch from latest origin/main
git fetch origin
git switch -c <epic-NN-story-X.Y> origin/main

# 2. Implement + verify the story
#    (run completion-verification scans before committing)

# 3. Commit atomically with the story ID + reviewer-verdict citation
git add <specific files>           # not `git add .` — explicit set
git commit -m "EPIC_NN Story X.Y: <one-line summary>

<body: what changed, why, reviewer verdicts cited>"

# 4. Push to origin
git push -u origin <epic-NN-story-X.Y>

# 5. Open the PR with reviewer verdicts + tracker row link in the body
gh pr create --base main --head <epic-NN-story-X.Y> \
  --title "EPIC_NN Story X.Y: <one-line summary>" \
  --body "<reviewer verdicts + tracker row link>"

# 6. Merge cleanly with --delete-branch
gh pr merge --merge --delete-branch
# Use --rebase only if the operator explicitly asks.
# If conflicts surface, RESOLVE on the feature branch.
# NEVER --force. NEVER reset --hard main.

# 7. Return to main; prune stale refs the merge left behind
git switch main
git pull --ff-only
git fetch origin --prune
```

## Hard Rules

- **NEVER push directly to `main`.**
- **NEVER batch multiple stories into one commit.** One story = one atomic commit on its own branch = one PR.
- **NEVER `--force` push or `reset --hard main`.** If a branch is wedged, resolve in place; ask if needed.
- **Use `--rebase` only when the operator explicitly asks.** Default is `--merge` so the audit trail preserves the per-story PR.
- **The audit trail is per-story-per-PR.** This matters when a regression has to be tracked back to its source.

## Commit Message Shape

```
EPIC_NN Story X.Y: <one-line summary, imperative mood, < 70 chars>

<paragraph explaining what changed and why — 2-3 sentences>

<paragraph citing reviewer verdicts, e.g.>
Reviewer verdicts:
- fail-fast-reviewer: PASS (Shape: anchor modified; no parallel file)
- security-reviewer: PASS (tenant-ID from JWT verified; audit log present)
- strategy-compliance-reviewer: PASS (I-3 sector cap respected)

Co-Authored-By: <agent or human attribution if applicable>
```

The body matters. A commit message that's just the title is a future incident waiting to happen — when something breaks in week N+4 and someone runs `git log --oneline` looking for the culprit, the body is what tells them whether to look here or keep searching.

## PR Body Shape

```markdown
## Summary

<2-3 sentence what + why>

## Changes

- <file>: <one-line description of what changed>
- <file>: <one-line description>

## Reviewer Verdicts

- **fail-fast-reviewer:** PASS — Shape: anchor `<file>` modified; no parallel file. Content scans clean.
- **security-reviewer:** PASS — tenant-ID derives from JWT, cross-account audit log verified.
- **strategy-compliance-reviewer:** PASS — invariant I-X respected.

## Tracker

- Marks Story X.Y ✅ done in IMPLEMENTATION_TRACKER.md at <link>.

## Test Plan

- [ ] AC-1 verified: `<command>` → `<actual output>`
- [ ] AC-2 verified: `<command>` → `<actual output>`
```

The "Reviewer Verdicts" block isn't decorative — it's the audit trail showing this story passed every required gate before merge.

## Conflict Resolution

If conflicts surface during merge:

1. **DO NOT** `git reset --hard origin/main`.
2. **DO NOT** force-push.
3. **DO** rebase your feature branch on top of latest origin/main (or merge origin/main INTO your feature branch).
4. **DO** resolve conflicts locally with each conflict treated as its own decision.
5. **DO** re-run completion verification scans after resolving conflicts (a conflict resolution can re-introduce a bug the scans were meant to catch).
6. **DO** push the resolved branch; the PR updates automatically.

## Why Per-Story PRs Matter

Two stories landed as one PR means:

- The PR diff is twice as big; reviewers (human or agent) parse less carefully.
- A regression caused by story 1 + revealed by story 2 looks like "the PR broke things" instead of "story 1 broke things, story 2 surfaced it."
- The revert path is "revert both" instead of "revert one."
- The audit trail conflates two completion reports into one PR body.

Per-story PRs cost a few extra `gh pr create` invocations and pay every time something needs to be tracked back through history.
