# Module: Container Image Selection Protocol

**Load when:** you're proposing or pinning a new container image, or auditing your project's existing image pins.

**Core rule lives in:** [`AGENT-INSTRUCTIONS.md` §2.1](../AGENT-INSTRUCTIONS.md#2-version--api-trust).

**Silent-plausibility variant fought:** `latest` tags or unverified Docker-Hub images that compile and run fine today but ship malware or break unpredictably tomorrow.

---

## The Protocol

For every external dependency you propose to run as a container (database, message broker, identity provider, headless browser, reverse proxy, etc.), pin the image deterministically — no guessing, no `library/<name>:latest`, no Docker-Hub-search heuristics.

1. **Verify at the project's OFFICIAL source-of-truth site** (the vendor's docs / release-notes page) — NOT Docker Hub alone. The "Docker Official Image" badge is necessary but not sufficient.
2. **If officially maintained:** pin the latest **LTS** `major.minor.patch` from the project's release page; if no LTS, latest stable (exclude `-rc` / `-alpha` / `-beta`).
3. **If NOT officially maintained:** build your own Dockerfile FROM a minimal distro (`alpine:<pin>`, `gcr.io/distroless/static-nonroot`, etc.) following the project's install/build guide. The Dockerfile lives under `services/<svc>/Dockerfile` (multi-stage, non-root, read-only-rootfs).
4. **Record in your version pin file** with a notes line citing the official-page URL that confirmed maintainership + the LTS/stable choice.

## Web-Fetch Tool Ladder

When verifying maintainership and the right pin, work from cheapest to most expensive:

1. **`wget` / `curl -fsSL`** via your shell tool — clean pass/fail, cheap. Works for static HTML or JSON release-notes endpoints.
2. **Structured fetch tool** (HTML → summary) — when the page is JS-light enough that fetching + parsing gives you what you need.
3. **Headless browser navigation** — last resort only when 1 and 2 both fail (504 errors, JS-rendered SPA release-notes pages, anti-bot CDN, login wall).

If all three fail, **STOP-ASK** with the tool sequence + a fallback proposal (e.g., pin from a signed release-notes URL directly). NEVER pivot to an unverified tag to "make it work" — that's a silent-plausibility instance dressed as productivity.

## What "Officially Maintained" Means

The container image is officially maintained when:

- The project's documentation site lists it as the canonical install method, OR
- The project's release-notes page references that specific image as the supported deployment shape.

NOT sufficient on its own:
- The Docker-Official-Image badge (Docker Hub's badge isn't the project's endorsement).
- A high star count on Docker Hub.
- A namespace that looks official (`postgres/`, `nginx/`) — anyone can name an image namespace anything.

When the official site points to a community image, the community image is acceptable IF the official site explicitly endorses it. Otherwise, build locally.

## When to Build Locally Instead

Build local Dockerfiles when:

- No official image exists (project is small / new).
- The official image bundles tools you don't need and you want a tighter attack surface.
- The official image runs as root and you're enforcing non-root containers project-wide.
- The project's install guide is "apt-get install foo" and you want to control the OS layer.

The local Dockerfile pattern:

```dockerfile
# Multi-stage; build in one, run in distroless
FROM alpine:3.21 AS build
RUN apk add --no-cache <build-deps>
COPY . /src
RUN <build steps>

FROM gcr.io/distroless/static-nonroot:nonroot
COPY --from=build /src/binary /usr/local/bin/binary
USER nonroot:nonroot
ENTRYPOINT ["/usr/local/bin/binary"]
```

This is the right shape because: pinned base distro, non-root by default, minimal runtime surface, multi-stage so build deps don't ship.

## Version Pin Record

In your `VERSIONS.yaml` (or equivalent):

```yaml
postgres:
  image: postgres:18.1
  notes: |
    LTS per https://www.postgresql.org/support/versioning/ (verified 2026-04-12).
    Major 18 is in active support through 2030; minor 18.1 is the latest stable.
```

The `notes:` line is the audit trail. Without it, the next person doesn't know whether `18.1` was deliberate or copy-pasted.

## Version Bump Discipline

Bumping a container pin is the same gate as adding a new dependency — explicit operator approval required. Bumps aren't "just change the tag"; environment variables, CLI flags, YAML schema keys, and config field names break across minor versions. Before changing any pin:

1. Read the vendor changelog and config schema diff for the version range you're crossing.
2. Capture the diff in the relevant LLD or decision log.
3. Surface to the operator with a recommended path (proceed / hold / revert) before applying.
