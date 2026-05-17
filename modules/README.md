# Agent Instruction Modules

These are **optional deep-detail modules** referenced from the core
[`AGENT-INSTRUCTIONS.md`](../AGENT-INSTRUCTIONS.md).
The core file is always loaded; modules are loaded on demand when a
specific topic comes up in your work.

Each module file opens with a **"Load when:"** trigger that tells you
the conditions under which you should pull that module into context.
If none of a module's triggers match your current task, skip it.

## Module Map

| Module | Load when |
|---|---|
| [`container-image-selection.md`](./container-image-selection.md) | You're proposing or pinning a new container image |
| [`hardcoded-config.md`](./hardcoded-config.md) | You're writing or reviewing code that touches a config-derived value |
| [`content-validation.md`](./content-validation.md) | You're integrating an external data source (vendor API, scraper, screener, feed) |
| [`tenant-identification.md`](./tenant-identification.md) | You're writing or reviewing a multi-tenant request handler (auth, persistence, audit) |
| [`revocation-endpoints.md`](./revocation-endpoints.md) | You're implementing logout, password-reset-confirm, refresh-revoke, or any "invalidate-me" endpoint |
| [`spec-compliance.md`](./spec-compliance.md) | You're writing integration code that touches a shared message format, API contract, or wire schema |
| [`primitive-reuse.md`](./primitive-reuse.md) | You're integrating with an off-the-shelf service (identity provider, OAuth2 server, message broker, search engine) and considering wrapping its primitives |
| [`clean-logs.md`](./clean-logs.md) | A tool emits "ignored", "deprecated", or "not supported" warnings on a clean run |
| [`no-parallel-implementations.md`](./no-parallel-implementations.md) | An LLD/epic/story uses any anchor phrase ("extends X", "mirrors X", "wider input to X", "same compute new entry point", "re-use as-is", "alongside X", "parallel to X", "shares the structure of X") OR you're reviewing a diff that adds new files alongside existing ones with similar names |
| [`root-causes-not-symptoms.md`](./root-causes-not-symptoms.md) | A bug is reported AND you're considering a retry loop, timeout bump, fallback default, warning suppression, or cache-extension fix |
| [`per-story-pr-sequence.md`](./per-story-pr-sequence.md) | You're at step (f) of the story cycle — branching, committing, and merging a completed story |
| [`scan-templates.md`](./scan-templates.md) | You're running completion-verification scans (every story before marking complete) |

## Why this structure

The core agent instructions file is built around two reading modes:

1. **Session-start refresher:** load the core file at the start of every session and after context compression. The core gives you every rule's WHAT and a 1-2 sentence WHY plus a pointer to the module. The agent reads ~250 lines and has the full rule set in mind.

2. **Just-in-time deep detail:** when a topic actively applies to your current diff, load the matching module from this directory. The module has the worked examples, anti-pattern catalogs, decision procedures, and grep commands for the rule.

This split keeps the core lean (the must-know rules fit on one screen) and pushes per-topic depth into files you load only when relevant. Modern context windows can fit everything, but loading on demand still helps: it keeps the rule active in attention and avoids burying it in 800 lines of mixed material.

If your platform supports project memory (a directory of always-loaded markdown files), point that memory at this `modules/` directory and the core file together — all rules in context, indexed for fast retrieval.
