# CLAUDE.md — Workspace Root

> **This file is a SAMPLE.** It demonstrates the shape of a workspace-root instruction file using the fictional TaskFlow project. The repo names, paths, build commands, architecture details, and rule examples below are illustrative — replace them with your project's actuals before using this as your real `CLAUDE.md`.

This file provides guidance to AI agents working across all repositories in this workspace.
Repository-specific instruction files reference this file for shared rules.

---

## Mandatory First Actions

**At the START of every session or after context compression:**
1. Re-read this file.
2. Re-read the repo-specific CLAUDE.md for the repo you're working in.
3. Check `VERSIONS.yaml` for current component versions.

**Signs you need to re-read:**
- Starting work in a repo you haven't touched recently.
- The user says "you're violating the rules again."
- Context feels "fresh" or you're unsure of standards.

---

## Workspace Overview

| Repository | Purpose | CLAUDE.md |
|------------|---------|-----------|
| **taskflow-web/** | React SPA frontend, Tailwind CSS, React Router v6 | Yes |
| **taskflow-api/** | Node.js/Express REST API, Prisma ORM, PostgreSQL | Yes |
| **taskflow-mobile/** | React Native companion app, push notifications | Yes |
| **taskflow-infra/** | AWS CDK stacks, CI/CD pipelines | Yes |
| **taskflow-specs/** | Shared API contracts, WebSocket event schemas | Yes |

**Always read the repository-specific CLAUDE.md before working in that codebase.**

---

## Cross-Repository Standards

### MVP — No Backwards Compatibility

This is an MVP. There is:
- **NO** production data to preserve
- **NO** legacy systems to support
- **NO** fallback logic needed
- **NO** phased migrations

Implement the new way directly. Delete old code completely.

### Version & API Trust

**Your training data is OUTDATED. You MUST verify current versions before writing code.**

Before using ANY library, framework, or tool:
1. Check `VERSIONS.yaml` for the pinned version.
2. Use documentation fetching tools to get current docs for that version.
3. Read existing code in the repo to match established patterns.
4. When in doubt, ASK the user rather than guessing.

### Security (Mandatory)

- **SQL Injection:** Use parameterized queries or ORM. No string interpolation in queries.
- **XSS:** Sanitize all user input rendered in HTML. Use framework defaults (React escapes by default).
- **Auth failures:** Invalid token = 401. Missing permissions = 403. Missing config = startup crash.
- **Multi-tenant:** Every database query scoped to current tenant. RLS enforced at database level.
- **Secrets:** Never in source control. Use environment variables or secret management.

### Code Quality

- **No placeholders in code.** Either implement fully or raise an error.
- **API responses:** Return empty arrays `[]`, never `null` for list endpoints.
- **Error handling:** Fail fast. Missing config = immediate failure. No silent fallbacks.

### Workflow

- Work incrementally to completion — finish each story before starting another.
- Never mark complete without testing — build, run, and verify functionality.
- Maximum 1-2 stories per session.

---

## Shared Specifications

**All cross-service API contracts are in `taskflow-specs/`.**

| Need to... | Go to |
|------------|-------|
| Find a REST endpoint | `taskflow-specs/api/REST_ENDPOINTS.md` |
| Understand WebSocket events | `taskflow-specs/events/WEBSOCKET_EVENTS.md` |
| Check notification payload | `taskflow-specs/events/NOTIFICATION_SCHEMA.md` |
| Propose a spec change | `taskflow-specs/_proposals/README.md` |

**Never modify a spec you don't own without going through the proposal process.**

---

## Component Versioning

All component versions tracked in `VERSIONS.yaml` at workspace root.

| Change Type | Version Bump |
|-------------|--------------|
| Breaking API/protocol change | MAJOR |
| New feature | MINOR |
| Bug fix | PATCH |

---

## Completion Verification

Before marking ANY story as complete:

### 1. Placeholder Scan
```bash
grep -rn "placeholder\|TODO\|FIXME\|would.*implement\|actual.*implementation" \
  <modified_directories> --include="*.ts" --include="*.tsx" --include="*.py" --include="*.js"
```

### 2. Fail-Fast Violation Scan
```bash
# Silent error swallowing
grep -rn "catch.*{}" <modified_files> --include="*.ts" --include="*.js"

# Empty catch blocks
grep -rn "catch.*{\s*}" <modified_files> --include="*.ts" --include="*.js"

# Default value substitution in error paths
grep -rn "|| ''" <modified_files> --include="*.ts" --include="*.js"
grep -rn "?? null" <modified_files> --include="*.ts" --include="*.js"
```

### 3. Verify Each Acceptance Criterion
Run the verification command. Show the output. If you can't demonstrate it works, it's NOT complete.

### 4. Provide Proof
```
## Story X Complete

### Acceptance Criteria Verification:
1. "Feature X works"
   - Command: `curl localhost:3000/api/feature`
   - Output: <actual output>

### Scans: (clean)
### Files Modified:
- path/to/file.ts (+45, -12)
```

---

## Quick Commands

### Web Frontend (taskflow-web)
```bash
npm run dev          # Start dev server
npm run build        # Production build
npm run test         # Run tests
npm run lint         # ESLint + Prettier
```

### Backend API (taskflow-api)
```bash
npm run dev          # Start with hot reload
npm run build        # TypeScript compilation
npm run test         # Jest tests
npx prisma migrate dev  # Run migrations
```

### Infrastructure (taskflow-infra)
```bash
cdk synth            # Generate CloudFormation
cdk diff             # Show pending changes
cdk deploy --all     # Deploy all stacks
```
