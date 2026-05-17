# The Pragmatic Agent

Guidance & templates for efficient agentic product development. Battle-tested practices from a multi-repository, safety-critical systems project.

---

## What's Inside

This repository contains two companion documents that work together to set up and run AI-assisted software development effectively:

### [Team Guide](./TEAM-GUIDE.md)

**Audience:** Tech leads, prompt engineers, and engineering managers.

A practical guide for orchestrating AI agents in your development process. Covers:

- Project structure for agentic AI (instruction files, version pinning, spec repos)
- The design-to-implementation pipeline (HLD > LLD > Epic > Story)
- Writing effective agent instructions
- Defining agent roles & authority levels (development, review, validation, research)
- Quality gates & review pipelines
- Story execution orchestration
- Epic & story template design
- Specification ownership & cross-team coordination
- Completion verification framework
- Escalation framework
- Common pitfalls & lessons learned

### [Agent Instructions](./AGENT-INSTRUCTIONS.md)

**Audience:** AI agents (loaded into agent context at session start).

The companion rule set that gets loaded into your AI agent's context window. Covers:

- Session & context management (surviving compression and new sessions)
- Version & API trust (never trust training data for versions)
- Fail-fast philosophy (forbidden patterns, correct patterns, `return None` rules)
- No placeholders policy
- Repository boundaries
- Destructive operation safeguards
- Security rules
- Specification compliance
- Code conventions
- Story execution role definition
- Completion verification scans
- Escalation triggers

### [Examples & Templates](./examples/)

**Audience:** Everyone. Copy these, adapt them, use them.

Ready-to-use templates and filled-in samples for every document type referenced in the guides. All examples use a fictional "TaskFlow" SaaS project to demonstrate the patterns.

```
examples/
├── VERSIONS.yaml                          # Version pinning file
├── workspace-claude-md/
│   ├── CLAUDE.md                          # Workspace-root agent instructions
│   └── repo-specific-CLAUDE.md            # Per-repo agent instructions
├── design/
│   ├── HLD_TEMPLATE.md                    # High-Level Design template
│   ├── SAMPLE_HLD.md                      # Filled-in HLD example
│   ├── DECISION_LOG_TEMPLATE.md           # Decision/rationale template
│   └── llds/
│       ├── LLD_TEMPLATE.md                # Low-Level Design template
│       └── SAMPLE_LLD_01_Authentication.md
├── stories/
│   ├── EPIC_TEMPLATE.md                   # Epic/story template
│   └── SAMPLE_EPIC_01_User_Auth.md        # Filled-in epic example
└── agents/
    ├── AGENT_TEMPLATE.md                  # Agent definition template
    ├── SAMPLE_backend-api-agent.md        # Development agent example
    └── SAMPLE_security-reviewer.md        # Review agent example
```

---

## How to Use

1. **Start with the Team Guide** to understand the overall framework and set up your project structure.
2. **Copy the templates** from `examples/` and fill them in for your project (HLD, LLDs, epics, agent definitions).
3. **Adapt the Agent Instructions** to your project's specific rules, conventions, and tooling.
4. **Place the adapted agent instructions** in the location your coding agent expects (e.g., `CLAUDE.md`, `AGENTS.md`, or `.github/copilot-instructions.md`) so they're loaded into agent context automatically.
5. **Evolve all documents** as you discover new agent failure modes -- every violation is a rule waiting to be written.

### Quick Start

If you want the fastest path to a working setup, start with one instruction file and one sample epic:

1. Copy `examples/workspace-claude-md/CLAUDE.md` into the root of the workspace or repository your agent will edit.
2. If you only have a single repository, keep the shared rules and repo-specific rules in that one file rather than splitting root vs. repo files.
3. Copy `examples/VERSIONS.yaml` and replace the sample components, dependencies, and versions with your real stack.
4. Use `examples/stories/SAMPLE_EPIC_01_User_Auth.md` as the reference for how traceable stories, verification commands, and fail-fast rules should look when filled in.
5. Adapt `AGENT-INSTRUCTIONS.md` to your environment, then place it where your platform auto-loads instructions.

Expected outcome: the agent starts each session with explicit fail-fast rules, known version pins, and story-level verification commands instead of guessing.

---

## Scope

This is **Claude-Code-shaped operational discipline.** The concepts (design-pipeline, story-cycle, fail-fast, no-parallel-implementations, silent-plausibility as the meta-anti-pattern, etc.) port to any agent platform; the tooling assumptions (`.claude/agents/`, `CLAUDE.md`, multi-agent fan-out via subagents, the model-tier naming) are written for the Anthropic stack.

If you're on a different platform (Codex, Cursor, Aider, Copilot CLI, Gemini CLI, custom orchestration on the Anthropic SDK), the framework applies — you'll just translate `.claude/agents/` to whatever your platform uses for agent definitions, `CLAUDE.md` to your platform's instruction-file convention, and "subagents" to whatever spawning primitive your runtime exposes. Don't expect platform parity out of the box; do expect the concepts to land.

### Verifying Your Adaptation

Use this checklist after copying the templates into your own project:

- [ ] Your instruction file explicitly tells the agent to re-read instructions at session start or after compression.
- [ ] Your fail-fast section includes concrete scan commands for the patterns you consider forbidden.
- [ ] Your `VERSIONS.yaml` matches the versions actually pinned in your codebase and deployment tooling.
- [ ] Every LLD includes an **Existing-Primitive Map** (§2.4) — every artifact has a named existing anchor (file:line) OR explicit narrow-exception justification.
- [ ] Every epic story carries an **Existing-Primitive Analysis** table filled in before implementation.
- [ ] Every reviewer agent's verdict requires an opening `Shape:` line (parallel-implementation detection).
- [ ] Your CI scan suite includes filename-pair detection (e.g., `<Prefix>Foo.tsx` next to `Foo.tsx`) and a trap-phrase audit on LLDs/epics ("mirrors", "wider input", "same compute", "re-use as-is" without a file-path anchor).
- [ ] Your CI runs the language-appropriate static-scan baseline (shellcheck / yamllint / `go vet` / `tsc --noEmit` / linter) on every PR — these are mandatory, not optional polish.
- [ ] Every epic references the relevant LLD sections with line numbers.
- [ ] Every story acceptance criterion includes a runnable verification command with observable output.
- [ ] Design or epic decisions that required human judgment are recorded with rationale instead of being left implicit.

---

## Core Philosophy

These guides are built on principles learned the hard way:

1. **Design before implementation.** Agents execute plans well. They make architectural decisions poorly.
2. **One story at a time.** Agents lose coherence across large scopes.
3. **Verify everything.** "It compiles" is necessary but insufficient.
4. **Fail-fast, never fail-silent.** The most damaging bugs are silent fallbacks, not crashes.
5. **Escalate decisions, not problems.** Agents implement; humans decide.
6. **Extend existing anchors; never fork parallel implementations.** When an LLD/epic says "extends X" or "mirrors X" or "same compute, wider input", the work modifies X in place. Parallel files alongside an anchor are violations even when otherwise correct.
7. **Reviewers must grade shape, not just content.** A reviewer that asks "is this file correctly implemented?" passes parallel duplicates every time. Every reviewer verdict opens with a `Shape:` line.
8. **Fix root causes, not symptoms.** No retry loops, no timeout bumps, no fallback defaults, no cache extensions to mask reported bugs. If you can't fix the root cause, label the fix "mitigation" explicitly.
9. **Surface issues; don't bypass.** Host operation failures, deprecation warnings, "ignored" config fields — every one is a defect. Workarounds accumulate forever; the operator can fix host config in seconds.

---

## Contributing

These documents are living guides. If you've found patterns that work (or don't) in your own agentic development workflow, contributions are welcome.

---

## License

This project is open source. See individual files for details.
