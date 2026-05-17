# Module: Clean Logs Are a Hard Requirement

**Load when:** a tool, framework, or runtime emits "ignored", "deprecated", or "not supported" warnings on a clean run — even when the warning claims "no runtime effect."

**Core rule lives in:** [`AGENT-INSTRUCTIONS.md` §9.2](../AGENT-INSTRUCTIONS.md#9-code-conventions).

**Silent-plausibility variant fought:** "harmless" log noise that hides real problems by training readers to skim warnings.

---

## The Rule

Every warning, info-with-yellow-formatting, deprecation notice, or "X is being ignored" line emitted on a clean run is a defect — even when the warning is "harmless" or the field/flag/path "has no runtime effect." Log noise is not a fail-fast or hardcoded-config violation on its own, but it **COMPOUNDS** those rules: the next person reading the logs (often you, in a future session) wastes hours chasing the warning before discovering it was decorative.

## The Procedure

When a tool warns that a config field, env var, flag, or declaration is "not supported / will be ignored / deprecated":

1. **STOP.** Don't ignore. Don't justify ("matches the YAML so it's safe," "the actual enforcement is elsewhere," etc.).
2. **Diagnose:** confirm the warning's claim — is the field really inert? Read the upstream changelog / issue tracker.
3. **Remove the inert artifact** (the source of the warning) AND verify the actual behavior the artifact APPEARED to control is enforced somewhere real. If the appearance was load-bearing (the field WAS supposed to enforce something and isn't), re-point the enforcement at the real layer.
4. **Repoint any CI gate** that was checking the inert field at the real enforcement layer IN THE SAME COMMIT.
5. **Surface to the operator BEFORE removing** if the change touches a production-shape file (compose / proxy config / pg_hba / Dockerfile) — even when removal looks safe.

## Why "Harmless" Is the Trap Word

The warning sits in the log because the tool's maintainers thought it was worth flagging. The reader who skims it gets two messages:

1. "This warning is OK to ignore." (the action they take this time)
2. "Warnings in this log are OK to ignore." (the habit they form)

The next warning — a real one — gets the same treatment because the reader has been trained that warnings here are decorative. The cost of leaving the warning in place isn't zero today; it's borrowed against the next incident.

## Worked Example

A team's compose file had `secrets.uid: 100`, `secrets.gid: 100`, `secrets.mode: 0400` on a service definition. The compose runtime warned "secrets uid/gid/mode are ignored; secrets are mounted as the container user." The team had been ignoring the warning for months because "the actual perms are enforced by the entrypoint chmod, this is just decorative."

Three things wrong with that reasoning:

1. The CI gate that was asserting "secrets are 0400" was looking at the inert compose fields, not the entrypoint chmod. The CI gate passed on a check that meant nothing.
2. When a new engineer joined the team, they read the inert fields as load-bearing and didn't realize the entrypoint script was the real enforcement — adding a new secret without the corresponding entrypoint chmod.
3. The warning cluttered every `docker compose up` run, making real warnings (like "image not found locally, pulling…") harder to spot.

The fix: removed the inert compose fields, repointed the CI gate at the entrypoint's `chmod 0400` call, surfaced to the operator before removing (compose is production-shape). One commit, three problems resolved, log noise gone.

## CI Gates Following the Real Behavior

When the inert artifact is removed, the CI gate that was asserting against it MUST be repointed at the real enforcement layer. Otherwise the next time the actual behavior breaks, no test catches it.

This is the root-cause-not-symptom rule applied to test infrastructure: the CI gate was a symptom of "we care about this behavior"; the real cause is the actual chmod call. Fix the test to look at the real cause.

## What This Module Does NOT Cover

- **Genuine errors in logs** — those are §3 fail-fast violations. Use that section.
- **Per-request informational logs** — these are normal traffic, not warnings.
- **Deprecation notices for upcoming breaking changes** — these are warnings on a future-clean-run baseline; address them BEFORE the upstream removal, but they're not "log noise" today.

The scope of this module is the "decorative" warnings on a clean run — the ones that exist because something in your config is no longer load-bearing but you haven't removed it yet.
