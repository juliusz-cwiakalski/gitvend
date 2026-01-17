# gitvend — Product Requirements Document (PRD)

## Overview

gitvend is a cross-platform CLI tool that **vendors selected files and folders from remote Git repositories** into a target repository in a **deterministic, auditable, and CI-friendly** way.

The tool is designed for multi-repo systems where shared artifacts (API contracts, schemas, reference documentation, templates) must be consumed across repositories without introducing live links (symlinks) or implicit coupling (submodules), and without manual copy/paste.

gitvend maintains **local bare Git mirrors** under `${HOME}/.gitvend/` (configurable) to reduce bandwidth and accelerate repeated sync operations.

---

## Problem statement

In multi-repository development, shared artifacts frequently drift:

- **API contracts diverge** between the producer service and consumers.
- **Central or platform documentation becomes stale**, causing incorrect implementation decisions and rework.
- CI pipelines repeatedly download the same repositories, wasting **time, bandwidth, and cost**.
- Multi-repo feature development is slowed by lack of a consistent mechanism to:
  - work on a shared feature branch across repositories,
  - keep shared artifacts aligned until work is complete,
  - and verify that vendored content matches its source.

Existing Git mechanisms do not fully address this need:
- Submodules introduce workflow friction and coupling.
- Symlinks are fragile and non-portable across OS/CI.
- Manual copy/paste is error-prone and hard to audit.

---

## Target outcomes

gitvend must enable a workflow where:

1. A target repository declares a **manifest** of external artifacts to vendor.
2. gitvend obtains those artifacts from a **deterministically resolved source revision**, preferably matching the target repo’s current branch name.
3. Vendored outputs are:
   - reproducible in CI,
   - reviewable in Git diffs,
   - traceable back to source repo/ref/SHA/path,
   - and safe to regenerate.

---

## Primary personas

### 1) Solo founder / lead developer
- Maintains multiple repositories (frontend + multiple services).
- Needs a low-friction workflow to keep shared docs/contracts aligned.
- Prefers fast iteration and deterministic results without operational overhead.

### 2) Platform engineer / architecture owner
- Maintains platform-level documentation and shared contracts.
- Needs clear provenance, governance boundaries, and compatibility discipline.
- Wants an explicit configuration (manifest) as the source of truth.

### 3) CI maintainer (GitHub Actions / GitLab CI)
- Needs deterministic builds and simple installation.
- Wants a `check` mode that fails on drift.
- Requires minimal credentials and secure defaults.

---

## Key use cases

### Use case A — Central documentation repository
- A central “system docs” repo vendors:
  - OpenAPI specs from service repos,
  - key ADRs or architecture diagrams,
  - shared runbooks.
- Sync is deterministic and reviewable.
- CI prevents publishing stale docs.

### Use case B — Contract vendoring
- Consumer repos vendor:
  - OpenAPI definitions,
  - JSON Schema,
  - protobuf/IDL files.
- During feature work, if the consumer is on branch `feature/X`, gitvend first tries to pull from `feature/X` in the source repo; if not present, it falls back to a configured main/default branch.

### Use case C — Platform assembly / curated view
- A platform repo vendors a curated set of artifacts from many repos (docs, schemas, templates).
- A lockfile records the exact SHAs used, enabling reproducible CI and auditing.

---

## Explicit non-goals

gitvend intentionally does **not** aim to:

- Provide symlink-based linking of documentation or other content.
- Replace Git submodules or manage submodule workflows.
- Provide “continuous syncing” or background daemons that update files implicitly.
- Provide atomic multi-repo commits/releases.
- Encourage editing vendored files in the consumer repo (the source repository remains the owner).

---

## Success criteria and measurable outcomes

### Functional success (v1)
- **Selective vendoring** works for files and directories.
- **Branch-aware resolution** works:
  - same-branch-first (target branch name),
  - configurable fallback (e.g., main),
  - and explicit failure modes.
- **Deterministic CI** supported via lockfile and `gitvend check`.
- **Mirror cache** reduces repeated network usage.

### Measurable outcomes
- CI and local sync operations show:
  - Reduced clone/fetch bandwidth compared to full cloning.
  - Repeatable results when using lockfile-resolved SHAs.
- Drift prevention:
  - `gitvend check` reliably fails when vendored content differs from source.
- Traceability:
  - Every synced entry can be traced to repo + resolved SHA + source path.

---

## Constraints

### Offline-ish operation via mirrors
- gitvend should minimize network usage by maintaining local bare mirrors.
- Full offline operation is not required, but the tool should work reliably when network access is intermittent (e.g., only fetch when needed).

### Reproducibility
- CI runs must be deterministic when using lockfile-resolved SHAs.
- Fallback decisions must be visible via a machine-readable report.

### Security
- No credentials stored in manifests.
- Support least-privilege read-only tokens in CI.
- Avoid unsafe filesystem writes (path traversal, overwriting outside repo).

### Portability
- Must run on Windows/macOS/Linux.
- v1 distribution target: **fat JAR + thin wrappers** (with Java 21+ recommended).
- Prefer extraction mechanisms that avoid OS-specific tooling differences (e.g., directory vendoring via `git archive --format=zip` + unzip).

---

## Open questions (explicitly tracked)

These are not blockers for writing specs, but must be decided before implementation finalization:

1. Lockfile naming: `gitvend.lock` vs `sync.lock`.
2. Manifest format: YAML (default) vs JSON/TOML.
3. Default fallback branch name: `main` vs repo default branch detection.
4. LFS behavior: treat LFS pointers as content vs optionally fetching LFS objects.

