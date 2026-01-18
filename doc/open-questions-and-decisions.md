# gitvend — Open Questions & Decisions Log

This document captures open documentation questions and decisions that were previously ambiguous, contradictory, or not fully specified.

Status legend:
- Open: needs a decision
- Proposed: suggested resolution, not yet ratified
- Decided: agreed and reflected across canonical docs

---

## Decisions

### OQ-001 — docs/ vs doc/ directory naming
- Status: Decided
- Decision: Use `doc/` as the canonical documentation directory.
- Action: Update all cross-links and references from `docs/` to `doc/`.

### OQ-002 — Ref policy naming: same-branch-else-main vs same-branch-else-default
- Status: Decided
- Decision: Use `same-branch-else-default` as the canonical policy identifier.

### OQ-003 — Default branch field location and naming
- Status: Decided
- Decision:
  - Sources define a default branch (defaulting to `main`).
  - Entries MAY override the default branch used for fallback.
  - Effective behavior for `same-branch-else-default`:
    - use entry override if present
    - else use source default
    - else `main`

### OQ-004 — Output artifact names and locations (report + lockfile)
- Status: Decided
- Decision:
  - Default Run Report path: `<target-repo>/.gitvend/gitvend.report.json`.
  - If the default report already exists, gitvend rotates it to `<target-repo>/.gitvend/gitvend.report-<iso-timestamp>.json` before writing the new report.
  - Vendor Lockfile location: next to the manifest file.
  - Vendor Lockfile name is derived from the manifest filename (base name + `.lock.` + same extension):
    - Manifest `gitvend.yml` -> lockfile `gitvend.lock.yml`
    - Manifest `gitvend.yaml` -> lockfile `gitvend.lock.yaml`
    - Manifest `.gitvend.yml` -> lockfile `.gitvend.lock.yml`
    - Manifest `.gitvend.yaml` -> lockfile `.gitvend.lock.yaml`.
  - The lockfile is also used to improve diagnostics:
    - if managed paths are dirty, gitvend should log a warning that this indicates local edits to vendored content and recommend editing at the source repo instead.

### OQ-005 — Auto-commit behavior and safety
- Status: Decided
- Decision:
  - Auto-commit exists and only commits files changed/managed by gitvend.
  - Auto-commit is enabled by default for `gitvend sync`.
  - Default dirty-policy (when auto-commit is enabled): `allow-unrelated`.
  - Optional dirty-policy: `allow-unrelated`.
    - Uncommitted changes in unmanaged paths are allowed.
    - Uncommitted changes in managed vendored paths are not allowed (gitvend must fail to prevent work loss).

### OQ-006 — Target Repo locking mechanism
- Status: Decided
- Decision:
  - Use a repo-level (Target Repo) lock.
  - The lock file is located at `<target-repo>/.gitvend/target-repo.lock.json`.
  - `sync` and `check` both acquire the Target Repo lock.
  - The lock file is automatically added to `.gitignore`.

### OQ-007 — Mirror ID algorithm: slug vs stable hash
- Status: Decided
- Decision:
  - Prefer a stable, human-readable slug derived from a normalized repo URL.
  - URL normalization includes removing protocol and trailing slashes (and other pre-normalization as needed).
- Still open:
  - Collision strategy if two different URLs normalize to the same slug (must be specified).

### OQ-008 — Manifest discovery: default filenames and precedence
- Status: Decided
- Decision:
  - Recognized manifest filenames (in precedence order):
    - `gitvend.yml`
    - `gitvend.yaml`
    - `.gitvend.yml`
    - `.gitvend.yaml`
  - If more than one recognized manifest exists in the same directory, gitvend fails with a manifest error (no implicit winner).

### OQ-009 — Minimum supported Git version
- Status: Decided
- Decision: Git `2.x.y` is supported (i.e., any Git v2 is acceptable for v1).

### OQ-010 — Container image UX (CI-friendly)
- Status: Decided
- Decision:
  - The container image must be usable as a CI image (e.g., GitLab CI) to run arbitrary shell scripts.
  - Image entrypoint should be a shell (e.g., `bash`), not `gitvend`.

### OQ-011 — Exit code granularity
- Status: Decided
- Decision:
  - Current exit-code granularity is sufficient.
  - More detailed diagnostics belong in structured stderr/log messages (and the JSON report when enabled).


### OQ-012 — Target Repo lock filename, format, and stale-lock recovery
- Status: Decided
- Decision:
  - Filename: `<target-repo>/.gitvend/target-repo.lock.json`.
  - Format: JSON (timestamp, pid, manifest path).
  - Timeout/stale-lock policy to be defined in storage spec.

### OQ-013 — Mirror id slug normalization and collision handling
- Status: Open
- Question:
  - Define the exact normalization rules and allowed character set.
  - How do we handle collisions deterministically (e.g., append `-<shortHash>` only when collision detected)?
