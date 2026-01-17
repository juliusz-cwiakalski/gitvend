# gitvend — Acceptance Criteria (AC)

This document defines acceptance criteria for **gitvend v1** and maps them to requirement IDs from `doc/requirements.md`.

---

## Definition of Done (v1)

gitvend v1 is considered “done” when:

1. A developer can run `gitvend sync` in a Git repository and have the tool:
   - discover or load a YAML manifest,
   - update required mirrors safely (with cross-platform locks),
   - resolve refs deterministically using same-branch-first with configured fallback,
   - vendor files and directories using object-level extraction (file: `git show`, dir: `git archive --format=zip`),
   - write/update `gitvend.lock`,
   - produce a JSON report,
   - optionally auto-commit managed changes with an audit-friendly commit message.

2. A CI job can run `gitvend check` and:
   - detect drift between vendored outputs and the resolved sources,
   - exit non-zero on drift,
   - avoid modifying workspace outputs.

3. The tool behaves reliably on Windows/macOS/Linux, including locking and directory extraction.

4. Documentation is sufficient for users and implementers:
   - README provides a correct quickstart.
   - `doc/INDEX.md` points to the canonical reading order.

---

## Acceptance criteria

### AC-001 — Mirror cache initialization
**Statement:** On first use of a source repo, gitvend creates a bare mirror under `${HOME}/.gitvend/mirrors/<mirror>.git` and can subsequently reuse it.

**Mapped requirements:** FR-001, FR-002, FR-003, FR-004, NFR-002

---

### AC-002 — Per-mirror locking is enforced and cross-platform
**Statement:** Concurrent executions attempting to update the same mirror cannot corrupt it; one process waits/fails according to timeout policy, using `${HOME}/.gitvend/mirrors/<mirror>.lock`.

**Mapped requirements:** FR-006, FR-007, FR-008, FR-009, NFR-004, NFR-009

---

### AC-003 — Lock file contains metadata
**Statement:** The lock file is valid JSON and contains at least lock acquisition timestamp and PID when available.

**Mapped requirements:** FR-007

---

### AC-004 — Mirror update behavior
**Statement:** `gitvend sync` fetches each required mirror at most once per invocation and does not redundantly fetch the same mirror when multiple manifests reference it.

**Mapped requirements:** FR-005, FR-033

---

### AC-005 — Mirror self-healing
**Statement:** If a mirror is detected as corrupted/unusable, gitvend can recover by re-initializing the mirror from scratch and completing the sync (assuming the remote is reachable).

**Mapped requirements:** FR-025a, NFR-008

---

### AC-006 — YAML manifest parsing and validation
**Statement:** gitvend accepts a versioned YAML manifest and rejects invalid manifests with a clear error. Path traversal and invalid target paths are rejected.

**Mapped requirements:** FR-011, FR-012, FR-013, FR-014, FR-015, FR-015a, NFR-006, NFR-007, NFR-012

---

### AC-007 — Manifest discovery within a repo
**Statement:** When no manifest path is provided, gitvend searches upward from CWD to the repo root (directory containing `.git`) and uses the first matching manifest per documented precedence; if none found, it fails with a clear error.

**Mapped requirements:** FR-032

---

### AC-008 — Multiple manifests in one invocation
**Statement:** gitvend can accept multiple manifest paths in a single invocation and sync them, reusing mirrors and avoiding redundant fetches.

**Mapped requirements:** FR-033

---

### AC-009 — Ref resolution uses same-branch-first with fallback
**Statement:** If the target repo is on branch `feature/x`, gitvend first attempts to use `feature/x` in the source; if missing, it applies the selected policy (fail or fallback to source default branch).

**Mapped requirements:** FR-016, FR-017, FR-018, FR-019, FR-020

---

### AC-010 — Detached HEAD behavior
**Statement:** When the target repo has no branch name (detached HEAD), gitvend applies the configured ref policy fallback strategy and records the decision in the report.

**Mapped requirements:** FR-016, FR-020

---

### AC-011 — File vendoring uses object-level extraction
**Statement:** For `file` entries, gitvend vendors content using `git show <sha>:<path>` from the resolved commit and writes it to the target path.

**Mapped requirements:** FR-021, FR-024

---

### AC-012 — Directory vendoring is portable and does not require checkout
**Statement:** For `dir` entries, gitvend vendors directories using `git archive --format=zip <sha> <path>` and unzips into the target path; no working-tree checkout is required.

**Mapped requirements:** FR-022, NFR-004

---

### AC-013 — Atomic writes prevent partial outputs
**Statement:** If an error occurs mid-sync, gitvend does not leave partially written final outputs. For directories, gitvend replaces content atomically or leaves the previous content intact.

**Mapped requirements:** FR-024, FR-025, NFR-008

---

### AC-014 — Idempotent sync
**Statement:** Re-running `gitvend sync` without source changes results in no output changes (no-op) and a report indicating unchanged entries.

**Mapped requirements:** FR-036, NFR-001

---

### AC-015 — Golden source overwrite semantics
**Statement:** If a managed vendored file is modified locally, a subsequent `gitvend sync` overwrites it to match the resolved source revision.

**Mapped requirements:** FR-037

---

### AC-016 — Determinism lockfile written
**Statement:** `gitvend sync` writes/updates `gitvend.lock` in the target repo, recording resolved commit SHAs for each source/ref used.

**Mapped requirements:** FR-026, FR-027, NFR-001

---

### AC-017 — JSON report produced
**Statement:** Each run produces a JSON report containing per-entry outcomes, resolved refs/SHAs, fallback usage, and errors when present.

**Mapped requirements:** FR-028, FR-020, NFR-011

---

### AC-018 — Fallback visibility
**Statement:** Whenever a fallback ref is used (e.g., source default branch), the report explicitly marks it; users can audit fallback usage.

**Mapped requirements:** FR-020, FR-028

---

### AC-019 — `check` detects drift and does not modify outputs
**Statement:** `gitvend check` exits non-zero if vendored outputs differ from what the manifest and resolved refs dictate, and does not modify workspace outputs.

**Mapped requirements:** FR-031, FR-027, NFR-001, NFR-003

---

### AC-020 — Git must be available
**Statement:** If `git` is not available on `PATH`, gitvend fails early with a clear error message.

**Mapped requirements:** NFR-003, NFR-004

---

### AC-021 — Automatic initialization of gitvend home
**Statement:** On first run, gitvend creates required directories under `${HOME}/.gitvend/` (e.g., `mirrors/`) without manual setup.

**Mapped requirements:** FR-035

---

### AC-022 — Convenience commands exist for manifest maintenance
**Statement:** gitvend provides commands to add sources and entries to a manifest, validating inputs and producing conforming YAML.

**Mapped requirements:** FR-034

---

### AC-023 — Auto-commit of managed changes
**Statement:** When `sync` changes managed outputs, gitvend stages and commits only those managed paths, using a commit message that includes source repo identifiers and resolved SHAs.

**Mapped requirements:** FR-038

---

### AC-024 — Container image available for CI
**Statement:** A container image exists as a supported distribution channel for CI usage.

**Mapped requirements:** FR-039, NFR-014

---

## Notes

- Any acceptance criteria involving exact filenames/locations (report name, manifest filename precedence) are satisfied once defined in `doc/cli-spec.md`, `doc/manifest-spec.md`, and `doc/output-artifacts.md`.

