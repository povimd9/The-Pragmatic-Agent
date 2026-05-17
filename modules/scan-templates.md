# Module: Completion-Verification Scan Templates

**Load when:** you're running completion-verification scans (every story before marking complete) OR setting up your project's CI pipeline.

**Core rule lives in:** [`AGENT-INSTRUCTIONS.md` §13.1](../AGENT-INSTRUCTIONS.md#13-completion-verification).

**Silent-plausibility variant fought:** "all scans clean" claims that ran the wrong scans, missed a language, or auto-dismissed findings.

---

## These Are Sample Shapes, Not Drop-in Scans

Every grep below **demonstrates the technique for a scan category**; the literal patterns will produce noise on any real codebase without calibration. Before adopting any scan:

- **Replace path globs** (`<files>`, `<modified_directories>`, `services/<svc>/`) with your project's actual paths.
- **Tighten the regex** against your codebase's idioms — e.g., the Python `return None$` grep will fire on every legitimate optional return; narrow it to error-path proximity or replace with an AST-aware tool (`ast-grep`, `ruff` custom rules, `semgrep`).
- **Add your project's domain-specific literals** (ticker symbols, role names, threshold values) to the hardcoded-config scan.
- **Strip language sections you don't use.**

A scan that produces hundreds of matches gets dismissed within a week — which is exactly the failure mode §3.5 warns against. Calibrate before deploying.

The mandate that comes from the core rule is that every **category** runs in CI; the specific commands implementing each category are project-specific. Treat this module as a sample-shape catalog you'll fork once and then maintain alongside your codebase.

---

## Universal Discipline

- Run ALL scan categories every story-completion run; missing categories produce false-clean reports.
- Review EACH match INDIVIDUALLY. Never blanket-dismiss ("they're all in the test dir," "they all match the YAML").
- If you believe a match is legitimate, get **explicit operator confirmation** before dismissing, AND record the dismissal IN-SOURCE as a single-line disable directive AND in the commit message.
- Fan-out: scans run in the SAME parallel batch as reviewer agents (step d of the story cycle). Don't chain serially.

## Placeholder Scan (universal)

```bash
grep -rn "placeholder\|TODO\|FIXME\|would.*implement\|actual.*implementation\|in production" \
  <modified_directories> \
  --include="*.go" --include="*.ts" --include="*.tsx" --include="*.py" \
  --include="*.sh" --include="*.java" --include="*.rs"
```

Every match is a violation. Either implement fully, or replace the comment with a `raise NotImplementedError("<what>")` / equivalent.

## Fail-Fast Scans

*Sample patterns — calibrate to your codebase's actual error idioms.*

### Python

```bash
# Silent error swallowing
grep -rn "except.*pass$\|except:$" <files> --include="*.py"

# Bare except
grep -rn "except:" <files> --include="*.py"

# Default-value substitution in error paths
grep -rn "\.get(.*,\s*0)\|\.get(.*,\s*\"\")\|\.get(.*,\s*False)\|\.get(.*,\s*None)" <files> --include="*.py"

# Silent return on missing data (§3.3 decision procedure applies)
grep -rn "return 0$\|return 0\.0$\|return \[\]$\|return None$\|return \"\"$" <files> --include="*.py"
```

> **Caveat on the silent-return grep.** This shape fires on every legitimate optional return. Use it only as a first-pass filter feeding human or LLM-reviewer judgment per §3.3, or replace with an AST-aware check that scopes to error-handling branches (`ast-grep`, `semgrep`, `ruff` custom rules). The **category** (silent-return-on-missing-data) is mandatory; this literal grep is not.

### Go

*Sample patterns — calibrate to your codebase's actual error idioms.*

```bash
# Suppressed panics
grep -rn 'recover()' <files> --include='*.go'

# Silent return on error
grep -rn 'if err != nil { return [^e]' <files> --include='*.go' \
  | grep -E 'return (nil|0|"")'

# Underscore-assignment of errors (excluding test files)
grep -rn '_ = ' <files> --include='*.go' | grep -v '_test.go'
```

### TypeScript / JavaScript

*Sample patterns — calibrate to your codebase's actual error idioms.*

```bash
# Empty catch blocks
grep -rn 'catch.*{}' <files> --include="*.ts" --include="*.tsx" --include="*.js"

# Silent fallback defaults
grep -rn '|| ""' <files> --include="*.ts" --include="*.tsx"
grep -rn '?? null' <files> --include="*.ts" --include="*.tsx"
```

### Rust

*Sample patterns — calibrate to your codebase's actual error idioms.*

```bash
# .unwrap() / .expect() in non-test files (each match needs review)
grep -rn '\.unwrap()\|\.expect(' <files> --include="*.rs" | grep -v 'test'

# Silent panic suppression
grep -rn 'catch_unwind' <files> --include="*.rs"
```

## SQL-Injection Scan (any language)

```bash
grep -rEn 'fmt\.Sprintf\(.*"(SELECT|INSERT|UPDATE|DELETE)' <files>     # Go
grep -rEn 'f["\x27].*\b(SELECT|INSERT|UPDATE|DELETE)\b' <files>         # Python f-strings
grep -rEn '`.*\$\{.*\}.*\b(SELECT|INSERT|UPDATE|DELETE)\b' <files>      # JS template literals
grep -rEn '\.format\(.*\b(SELECT|INSERT|UPDATE|DELETE)\b' <files>       # Python .format()
```

## Hardcoded-Config Scan (see [`hardcoded-config.md`](./hardcoded-config.md))

Per-service: substitute `<svc>` with the service directory; substitute the patterns with your project's domain-specific literals.

```bash
# Decimal literals that look like thresholds / weights / caps
grep -rEn 'decimal\.(NewFromString|RequireFromString|NewFromFloat)\("?0?\.[0-9]' \
  services/<svc>/ --include='*.go' \
  | grep -v '_test\.go' \
  | grep -v 'internal/.*loader'

# Hardcoded business constants outside the loader package
grep -rEn '"(your-domain-specific-tickers|host-names|role-names)"' \
  services/<svc>/ --include='*.go' \
  | grep -v '_test\.go' \
  | grep -v 'internal/.*loader'

# Time literals (durations, intervals)
grep -rEn '[0-9]+\s*\*\s*time\.\(Second\|Minute\|Hour\|Day\)' \
  services/<svc>/ --include='*.go' \
  | grep -v 'internal/.*loader' \
  | grep -v '_test\.go'
```

Every match is an EXPLICIT operator-approval gate. NEVER auto-dismiss as "matches the YAML." See [`hardcoded-config.md`](./hardcoded-config.md) for the rule and worked anti-example.

## Static-Scan Baseline (HARD RULE — run per language touched)

| Diff touches | Required static scan |
|---|---|
| `*.sh` | `shellcheck -x -S warning <files>` (or your project's shell linter of choice) |
| `*.yaml` / `*.yml` | `yamllint <files>` + schema-level validators (`docker compose config -q`, etc.; or your project's YAML toolchain) |
| `*.go` | `go vet ./...` (or `staticcheck`, `golangci-lint`, etc. — verify whichever you pick actually runs as part of `go test ./...`) |
| `*.ts` / `*.tsx` | `tsc --noEmit` + `eslint` (or `biome`, `oxlint`, etc. per project convention) |
| `*.py` | `ruff check` (or `flake8` + `mypy`, `pylint`, etc. per project convention) |
| `*.rs` | `cargo clippy -- -D warnings` (or your project's lint toolchain) |
| `*.java` | `spotbugs` or `errorprone` (or `pmd`, `checkstyle`, etc. per project convention) |

Findings are NOT auto-dismissable as "false positives" — every finding gets the same treatment as a hardcoded-config match: STOP, diagnose, either fix the code or surface for explicit operator approval (with the reason recorded IN-SOURCE as a single-line disable directive AND in the commit message).

## No-Parallel-Implementations Scan (universal — see [`no-parallel-implementations.md`](./no-parallel-implementations.md))

```bash
# Filename-pair scan — flag <Prefix>X.tsx next to X.tsx
# Substitute the scope prefixes your project actually uses
# (e.g. Admin, Bulk, Tenant, Org, User, etc.).
for f in src/pages/*.tsx; do
  base=$(basename "$f" .tsx)
  for prefix in Admin Bulk Tenant Org User; do
    if [[ "$base" == "$prefix"* && "$base" != "$prefix" ]]; then
      anchor="${base#$prefix}"
      if [ -f "src/pages/${anchor}.tsx" ]; then
        echo "PAIR: ${anchor}.tsx ↔ ${base}.tsx"
      fi
    fi
  done
done

# Trap-phrase audit on LLDs / epics / stories.
# The exclusion filter removes lines that DO carry a file-path anchor;
# the remaining matches are unanchored trap-phrases. The path pattern
# is language-agnostic — it matches any token shaped like
# "<segment>/<segment>.<extension>" (e.g., src/foo.go, lib/Bar.tsx,
# pkg/baz/qux.py, internal/widget.rs). Tighten or extend the pattern
# for your project if it false-positives on prose that looks like a
# path, or if your codebase uses extension-less files as anchors.
grep -rEn '(mirrors? existing|wider input|same compute|re-?use[s]? .* as[- ]is|extends? existing|parallel to existing|shares? the structure of)' \
  design/ stories/ --include='*.md' \
  | grep -vE '[A-Za-z0-9_./-]+/[A-Za-z0-9_.-]+\.[A-Za-z0-9]+'
# Any line that matches WITHOUT a source-file path on the same line is
# an LLD-authoring quality miss; redraft so the trap-phrase carries
# its anchor inline.
```

Each match is justified ONLY by an explicit Existing-Primitive Analysis entry in the owning story AND a narrow-exception argument. NEVER auto-dismiss as "the new file is correctly implemented."

## Forbidden-Library Scan (project-specific list)

Every project has a list of libraries that MUST NOT appear in its code (broker SDKs for read-only projects, deprecated frameworks being phased out, license-incompatible dependencies). Encode it as a grep:

```bash
grep -rEn '<lib1>|<lib2>|<lib3>' services/ specs/
# Expected: no output. Each match = a forbidden-library violation.
```

## Spec Round-Trip (when generators are involved)

```bash
# OpenAPI / swagger / protobuf round-trip
<regen command>                             # e.g., swag init, buf generate
git diff --exit-code <generated files>      # must produce zero diff
```

Drift in either direction is fragile integration. CI fails if the round-trip leaves a non-empty diff.

## Run Order (Per Story)

```
1. Static-scan baseline per language touched     # cheap, fail fast
2. Placeholder scan                              # cheap
3. Fail-fast scans                               # cheap
4. SQL-injection scan                            # cheap
5. Hardcoded-config scan                         # cheap
6. No-parallel-implementations scan              # cheap
7. Forbidden-library scan                        # cheap
8. Spec round-trip                               # moderate
9. Build + unit tests                            # moderate
10. Integration tests                            # expensive
11. Manual content-plausibility check            # expensive
```

Steps 1-7 run in a single parallel batch (alongside the reviewer agents). Steps 8-11 chain only if step 1-7 are clean, but that's a triage-order convention, not a structural dependency.
