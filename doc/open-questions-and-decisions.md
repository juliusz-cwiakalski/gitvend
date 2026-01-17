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
- Status: Partially decided
- Decision:
  - Lockfile name: `gitvend.lock` (in the target repository, at repo root).
- Still open:
  - Canonical report name/location (README currently suggests `gitvend.report.json`, but this is not yet finalized in a spec).

### OQ-005 — Auto-commit behavior and safety
- Status: Decided
- Decision:
  - Auto-commit exists and only commits files changed/managed by gitvend.
  - Auto-commit is configurable in the manifest and overridable by an environment variable.
- Still open:
  - The exact manifest field name and env var name (to be defined in `doc/manifest-spec.md` / `doc/cli-spec.md`).
  - Behavior when the repository has unrelated dirty changes (fail vs proceed) needs to be specified.

### OQ-006 — Workspace-level locking mechanism
- Status: Decided
- Decision:
  - Use a repo-level (target repo) workspace lock.
  - The lock file is created next to the manifest file being used, to make active syncing visible.
  - The lock file is automatically added to `.gitignore`.
- Still open:
  - Exact lock filename, lock metadata format, timeout/stale-lock policy (to be specified in `doc/storage-and-locking.md`).

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
  - Default manifest filename precedence:
    - `gitvend.yml`
    - `gitvend.yaml`
    - `.gitvend.yml`
    - `.gitvend.yaml`

### OQ-009 — Minimum supported Git version
- Status: Decided
- Decision: Minimum supported Git version is `2.20`.

---

## Follow-up questions (new)

### OQ-010 — Report filename/location and lifecycle
- Status: Open
- Question:
  - What is the canonical report filename and where is it written by default?
  - Is it always written, or only with a flag?
  - Should `check` write a report too (even on success), or only on drift?

### OQ-011 — Workspace lock filename, format, and stale-lock recovery
- Status: Open
- Question:
  - What is the exact filename (e.g., `<manifest>.lock`, `.gitvend.lock`, etc.)?
  - Is it JSON like mirror locks (timestamp, pid, manifest path), and do we reuse the same lock structure?
  - What is the timeout/stale-lock recovery policy?

### OQ-012 — Mirror id slug normalization and collision handling
- Status: Open
- Question:
  - Define the exact normalization rules and allowed character set.
  - How do we handle collisions deterministically (e.g., append `-<shortHash>` only when collision detected)?
