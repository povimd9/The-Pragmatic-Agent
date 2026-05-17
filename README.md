# The Pragmatic Agent

> Battle-tested operating discipline for software engineering with AI coding agents. Built to defeat **silent plausibility** — the failure mode where agent output looks right and is wrong.

AI coding agents don't usually fail by crashing. They fail by being **plausible**.

They claim completion when the code compiles. They swallow errors behind fallback values and call it a fix. They drop a `BulkFoo.tsx` next to `Foo.tsx` — a near-clone with one swapped hook — and every content-level review passes because the file is correctly implemented in isolation. The bug is that the file exists at all. The codebase silently forks. The build stays green.

**The Pragmatic Agent** is a set of rules, templates, and review gates that exist because their absence caused a real bug, a real integration failure, or a real chunk of wasted work. It's for teams shipping serious software with AI agents in the loop — Claude Code, Cursor, Codex, Copilot, or any similar tool — who want the agent's leverage without inheriting its failure modes.

Every rule earned its place the hard way.

---

## What this is

A three-layer operating discipline:

1. **A team playbook** ([`TEAM-GUIDE.md`](./TEAM-GUIDE.md)) — for tech leads and engineering managers setting up the workflow: operating frame, design pipeline (HLD → LLD → Epic → Story), agent roles and authority, review pipelines, model selection and cost budgets, scaling down by diff size, test design, completion verification, escalation framework.

2. **An agent rulebook** ([`AGENT-INSTRUCTIONS.md`](./AGENT-INSTRUCTIONS.md)) — loaded into agent context at session start. Opens with the **silent plausibility** meta-anti-pattern that every downstream rule fights, with a rule-by-rule map of which variant each one targets. Fail-fast rules, no-placeholder policy, repository boundaries, security stance, no-parallel-implementations detection, completion-verification scans, escalation triggers.

3. **Deep-detail modules** ([`modules/`](./modules/)) — load-on-demand depth for the rules that need it: container image selection, hardcoded-config detection, content validation, tenant identification, revocation endpoints, primitive reuse, clean logs, no-parallel-implementations, root-causes-not-symptoms, per-story PR sequence, scan templates, spec compliance. Each module opens with a `Load when:` trigger so the agent pulls only what's relevant to the current task.

Plus templates and worked examples for every artifact the methodology references — HLDs, LLDs, epics, stories, agent definitions, instruction files, version pins, decision logs.

## What this isn't

- **Not an agent framework.** No code to install. No LangChain successor. No runtime.
- **Not [12-Factor Agents](https://github.com/humanlayer/12-factor-agents).** 12FA is a methodology for *building* LLM applications. The Pragmatic Agent is a methodology for *using* AI agents to build software. The two are complementary; apply both to the same project, in different layers.
- **Not prompt engineering tips.** No "prompts that 10x your productivity." This is operational discipline at the SDLC level — design pipelines, review gates, verification frameworks.
- **Not platform-neutral marketing.** The patterns port to any AI coding agent. The examples are Claude-Code-shaped because that's where the rules were paid for. We're honest about that — see [Scope](#scope) below.

## Who it's for

- **Tech leads and engineering managers** standing up AI-assisted development workflows on real codebases.
- **Senior engineers** who've watched plausible AI-generated diffs ship subtle bugs and want a structural defense.
- **Prompt engineers** writing system instructions that need to survive across long sessions and across team members.
- **Teams adopting Claude Code, Cursor, Codex, or similar** for production work where "looks correct" isn't enough.

If a regression means a customer call, a security review, or a production rollback, this is the discipline.

---

## The core idea

AI agents fail at being plausible, not at being wrong. Every rule in this repo points at one anti-pattern: **silent plausibility** — output that compiles, looks right, and matches the surface shape of what was asked, while quietly violating an invariant the agent never knew was load-bearing.

The diagnostic question for any agent-produced artifact:

> **"Could this output be wrong in a way no one will notice?"**

If yes, you're in silent-plausibility territory — apply the relevant rule before shipping. [`AGENT-INSTRUCTIONS.md` §0](./AGENT-INSTRUCTIONS.md#0-silent-plausibility--the-meta-anti-pattern) maps each rule to the variant it fights.

The discipline rests on three commitments:

1. **Design before implementation.** The agent drafts; the human decides. HLD → LLD → Epic → Story. All decisions captured *before* a single file is touched.
2. **Verify everything at runtime.** Static analysis is not verification. Every acceptance criterion has a runnable command that produces observable proof.
3. **Fail loud or don't fail.** No fallback defaults, no silenced exceptions, no `return None` when something feels off. Missing precondition is a bug; surface it.

---

## The principles

Ten operating principles, all subordinate to the silent-plausibility meta-rule above. Each one comes in the rulebook with FORBIDDEN / CORRECT code examples and the specific scan command that catches violations.

1. **Re-read your instructions.** Context compression and new sessions erase rules. Survive that by re-reading at the start of every session.
2. **Pin everything, trust nothing.** Training data is outdated. `VERSIONS.yaml` is the source of truth. Read the actual codebase before writing new code.
3. **Fail fast, never fail silent.** Missing config crashes on startup. Errors propagate. `return None` on a precondition failure is a bug, not a graceful degrade.
4. **No placeholders.** Implement fully or raise `NotImplementedError`. No `TODO`, no "would implement," no mock data in production paths.
5. **Extend existing anchors.** When an LLD says "extends X" or "mirrors X" or "same compute, wider input," X is the anchor and the work modifies X *in place*. Parallel files alongside an anchor are violations even when otherwise correct. The bug is that the new file exists.
6. **Reviewers grade shape AND content.** A reviewer that asks "is this file correctly implemented?" passes parallel duplicates every time. Every verdict opens with a `Shape:` line.
7. **Fix root causes, not symptoms.** No retry loops, no timeout bumps, no fallback defaults, no cache extensions to mask a reported bug. If you can't fix the root cause, label the fix "mitigation" explicitly.
8. **Escalate decisions, not problems.** Multi-choice situations stop and ask the human. The agent implements; the human decides.
9. **Verify with runtime proof.** "It compiles" and "tests pass" are necessary but insufficient. Show the command, show the actual output, show the deploy.
10. **Evolve the rules from violations.** Every new agent failure mode is a rule waiting to be written. The instruction file grows after every sprint.

---

## What's inside

```
.
├── README.md                       ← you are here
├── TEAM-GUIDE.md                   ← operational playbook for tech leads
├── AGENT-INSTRUCTIONS.md           ← rulebook loaded into agent context
├── modules/                        ← deep-detail modules (load on demand)
│   ├── README.md                   ← module map with `Load when:` triggers
│   ├── no-parallel-implementations.md
│   ├── hardcoded-config.md
│   ├── content-validation.md
│   ├── tenant-identification.md
│   ├── revocation-endpoints.md
│   ├── root-causes-not-symptoms.md
│   ├── scan-templates.md
│   └── ...
└── examples/                       ← templates + filled-in samples
    ├── VERSIONS.yaml               ← version pinning file
    ├── workspace-claude-md/        ← instruction file examples (root + per-repo)
    ├── design/
    │   ├── HLD_TEMPLATE.md         ← + SAMPLE_HLD.md
    │   ├── DECISION_LOG_TEMPLATE.md
    │   └── llds/                   ← LLD template + auth sample
    ├── stories/                    ← epic + story template + auth sample
    └── agents/                     ← agent definitions (dev + reviewer)
```

The fictional "TaskFlow" SaaS used throughout the samples is a multi-tenant task management platform — chosen because it's complex enough to demonstrate every pattern (multi-tenancy, RLS, real-time, mobile, auth) without exposing any real project data.

**What's prescriptive vs illustrative:**

- **Rule statements** in `AGENT-INSTRUCTIONS.md` and module preambles **travel as-is.** They're the load-bearing content — the WHAT and WHY of each rule.
- **Everything else** — `examples/` (TaskFlow domain, version pins, file paths, agent definitions, sample acceptance criteria) and the literal greps / path patterns inside `modules/` — is **illustrative shape to calibrate.** Treating any sample as drop-in will either over-constrain you to TaskFlow's shape or leave gaps where TaskFlow's shape doesn't match yours.

See per-file headers and `examples/README.md` for what's prescriptive vs illustrative at each layer.

---

## Quick start

1. **Skim the team guide** end-to-end to understand the framework (~25 minutes).
2. **Copy the core instruction file** (`AGENT-INSTRUCTIONS.md`) into your agent's expected location — `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, `.github/copilot-instructions.md`, depending on platform. Adapt to your stack.
3. **Point your agent's project memory at `modules/`** (or copy the relevant modules alongside your instruction file). The core is always in context; modules load on demand when their `Load when:` triggers match.
4. **Copy `examples/VERSIONS.yaml`** and replace the sample versions with what you actually run.
5. **Adopt the design pipeline gradually.** Start with the epic + story template for your next multi-file feature. Add LLDs when you hit a domain that needs deeper design. The HLD comes when the system is big enough to need one.
6. **Adapt the scan templates in [`modules/scan-templates.md`](./modules/scan-templates.md) to your stack and run them in CI from day one.** The greps in that module are illustrative shapes — calibrate the literals, paths, and language coverage to your project before relying on them.

Expected outcome: agents that fail loudly instead of plausibly, work that ships with runtime proof instead of "looks right," and a discipline that compounds — every new failure mode becomes a new rule.

---

## Scaling down

The framework is calibrated for multi-file feature work, but it scales down. See [`TEAM-GUIDE.md` §7.5](./TEAM-GUIDE.md#75-rules-by-diff-size--scaling-down) for the full diff-size tier table:

| Tier | Examples | Ceremony |
|---|---|---|
| **Trivia** | typo fix, version bump pre-approved in VERSIONS.yaml | Direct commit, no plan agent, no `Shape:` line |
| **Small** | bug fix in one function, helper addition | Plan optional in PR description; one reviewer; `Shape:` line if new function |
| **Multi-file feature** | full story implementation | Full ceremony per the story cycle |
| **Epic-level** | new component, large feature, migration | Full ceremony + design-pipeline gate + end-of-epic runtime test |

The language-level invariants (no placeholders, no silent fallbacks, spec compliance, fail-fast) apply at every tier. What scales down is the *process*, not the *invariants*.

---

## Scope

This is **Claude-Code-shaped operational discipline.** The concepts — design pipeline, story cycle, fail-fast, no-parallel-implementations, silent plausibility as the meta-anti-pattern — port to any agent platform. The tooling assumptions (`.claude/agents/`, `CLAUDE.md`, multi-agent fan-out via subagents, the model-tier naming) are written for the Anthropic stack.

If you're on a different platform (Codex, Cursor, Aider, Copilot CLI, Gemini CLI, custom orchestration on the Anthropic SDK), the framework applies — translate `.claude/agents/` to whatever your platform uses for agent definitions, `CLAUDE.md` to your platform's instruction-file convention, and "subagents" to whatever spawning primitive your runtime exposes. Don't expect platform parity out of the box; do expect the concepts to land.

Scan commands in the rulebook are mixed-language by default (Python, Go, TypeScript, Rust, shell); pick the ones that apply to your stack. The [static-scan baseline](./modules/scan-templates.md#static-scan-baseline-hard-rule--run-per-language-touched) lists what's mandatory by file type.

---

## FAQ

**How is this different from 12-Factor Agents?**

12-Factor Agents (Dex Horthy / HumanLayer) is a methodology for building LLM-powered applications — the agent code itself. The Pragmatic Agent is a methodology for using AI coding agents to build software — the SDLC discipline around the agent. The two are complementary; you can apply both to the same project, in different layers.

**Is this just for Claude Code?**

The patterns are platform-agnostic. The examples are Claude-coded because that's where the rules were paid for. See [Scope](#scope).

**My team is small. Do I need HLDs and LLDs and epics for everything?**

No. See [Scaling down](#scaling-down). The discipline is the operating posture; the documents are the artifacts.

**Why is so much of this about reviewing?**

Because the highest-cost AI agent failures are the ones that pass review. Silent plausibility is invisible to readers who grade content correctness without checking structure. Most of the rules around shape-checking, existing-primitive analysis, and the `Shape:` reviewer line exist to defend against this single failure mode.

**Can I use this with AI-coding-skeptical teammates?**

Yes — this is arguably *more* useful for skeptics, because it gives them concrete failure modes to watch for and concrete verification gates to demand. The discipline doesn't require trusting the agent; it assumes the agent will produce confident plausible incorrectness and gates accordingly.

**Why split the rules into core + modules instead of one big file?**

[`TEAM-GUIDE.md` §4.5](./TEAM-GUIDE.md#45-evolving-your-instruction-files) covers the three storage patterns (monolithic / per-rule memory / core-plus-modules) and which to pick for your context budget. This repo demonstrates Pattern C: lean core (always loaded) + on-demand modules. If your project is smaller, Pattern A (monolithic) is simpler; if you spawn many narrow-scope sub-agents, Pattern B (per-rule memory) lets each load just what it needs.

---

## Contributing

This repo is a living document. The fastest way to contribute is to add a rule that came from a real failure you saw, including:

1. The specific anti-pattern (FORBIDDEN code example).
2. The correct pattern (CORRECT code example).
3. The scan command that would have caught it.

Open a PR or issue. Rules without a paid-for incident behind them tend to get rejected — the discipline only stays sharp if every entry earned its place.

## Acknowledgments

This is the SDLC-discipline sibling to [12-Factor Agents](https://github.com/humanlayer/12-factor-agents) by Dex Horthy. Where 12FA argues that production LLM applications are mostly deterministic code with strategic LLM augmentation, The Pragmatic Agent argues that production software *built with* AI agents is mostly disciplined SDLC with strategic agent augmentation. Same shape, one layer up.

The name and tone owe a debt to *The Pragmatic Programmer* (Hunt & Thomas) — practical, opinionated, hard-won, humble.

## License

MIT License — see [LICENSE](./LICENSE).
