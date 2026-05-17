# Example Templates & Samples

This directory contains templates and filled-in examples for every document type referenced in the [Team Guide](../TEAM-GUIDE.md).

All examples use a fictional **"TaskFlow"** project — a multi-tenant SaaS task management platform with a web frontend, backend API, and mobile companion app — to demonstrate the patterns without exposing any real project data.

---

## Directory Structure

```
examples/
├── README.md                          # This file
├── VERSIONS.yaml                      # Version pinning file
├── workspace-claude-md/
│   ├── CLAUDE.md                      # Workspace-root agent instructions
│   └── repo-specific-CLAUDE.md        # Per-repo agent instructions
├── design/
│   ├── HLD_TEMPLATE.md                # High-Level Design template
│   ├── SAMPLE_HLD.md                  # Filled-in HLD example
│   ├── DECISION_LOG_TEMPLATE.md       # Design/planning decision log
│   └── llds/
│       ├── LLD_TEMPLATE.md            # Low-Level Design template
│       └── SAMPLE_LLD_01_Authentication.md  # Filled-in LLD example
├── stories/
│   ├── EPIC_TEMPLATE.md               # Epic/story template
│   └── SAMPLE_EPIC_01_User_Auth.md    # Filled-in epic example
└── agents/
    ├── AGENT_TEMPLATE.md              # Agent definition template
    ├── SAMPLE_backend-api-agent.md    # Development agent example
    └── SAMPLE_security-reviewer.md    # Review agent example
```

---

## How Documents Relate

```
HLD (System-level architecture)
 ↓ referenced by section number (§X.X)
LLD (Component-level design, one per major component)
 ↓ referenced by LLD number + line range
Epic (Implementation plan, 2-5 stories per epic)
 ↓ references LLD sections for traceability
Story (What the agent implements in a single session)
```

**Agent definitions** load domain knowledge from HLD/LLD references and enforce quality gates during implementation.

**Instruction files** (shown in this repo as `CLAUDE.md` examples) are loaded into agent context at session start. They reference the design documents and contain the operational rules.

**VERSIONS.yaml** is the single source of truth for all library/framework/tool versions used in the project.

---

## Using These Templates

1. **Copy the template** for the document type you need.
2. **Fill in your project-specific details** using the sample as a reference.
3. **Maintain traceability** — every LLD references HLD sections, every epic references LLD sections with line numbers, and non-obvious decisions are captured with rationale.
4. **Evolve continuously** — add rules to your instruction file when agents make mistakes, update VERSIONS.yaml when dependencies change.

---

## Template vs. Sample

- **`*_TEMPLATE.md`** files contain placeholder fields (`<fill in>`) and structural guidance. Copy these as starting points.
- **`SAMPLE_*.md`** files are filled-in examples showing what a completed document looks like. Use these as reference when filling in templates.
