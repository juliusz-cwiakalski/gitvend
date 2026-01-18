# gitvend — Requirements (FR/NFR)

This document defines functional requirements (FR) and non-functional requirements (NFR) for **gitvend**.

The scope is aligned with `doc/prd.md` and the current decisions:
- Mirror base directory: `${HOME}/.gitvend/`
- Per-mirror lock file: `${HOME}/.gitvend/mirrors/<mirror>.lock.json`
- Vendor Lockfile: next to the manifest, named `<manifest-base>.lock.<ext>` (e.g., `gitvend.lock.yml`)
- Manifests: YAML v1
- Ref resolution: same-branch-first, fallback to per-source configured default branch (`main` by default; legacy repos may specify `master`)
  - Default is configured on Source Repo and can be overridden per Vendor Entry.
- Directory vendoring: `git archive --format=zip` + unzip
- LFS: treated as pointer files (no implicit fetching)
- Git must be present on `PATH`

---

## Functional requirements

### Mirrors and caching

**FR-001 — Mirror home directory**
- gitvend MUST store its local cache under `${HOME}/.gitvend/` by default.
- gitvend SHOULD support overriding this location via configuration (exact mechanism defined in `doc/cli-spec.md` and `doc/manifest-spec.md`).

**FR-002 — Mirror storage structure**
- gitvend MUST store bare mirrors under `${HOME}/.gitvend/mirrors/`.
- Each mirror MUST be a **bare** Git repository at: `${HOME}/.gitvend/mirrors/<mirror>.git`.
- gitvend MAY store mirror metadata alongside the mirror (e.g., `${HOME}/.gitvend/mirrors/<mirror>.meta.json`).

**FR-003 — Stable mirror identity**
- gitvend MUST compute a stable mirror identifier `<mirror>` from the remote repository URL.
- The identifier MUST be stable across runs and environments.
- The mapping algorithm MUST be deterministic, human-readable, and based on a filesystem-safe slug derived from a normalized repository URL (e.g., remove protocol, normalize separators, remove trailing slashes).

**FR-004 — Mirror initialization**
- If a mirror does not exist locally, gitvend MUST initialize it from the remote repository.
- Initialization MUST preserve remote branches and tags required for resolution.

**FR-005 — Mirror update behavior**
- gitvend MUST be able to update an existing mirror via fetching from its remote.
- `sync` MUST fetch mirrors (at most once per mirror per invocation).
- gitvend SHOULD store mirror update timestamps in mirror metadata to enable skipping redundant fetches within a single multi-manifest run.

### Locking (mirrors and Target Repo)

**FR-006 — Per-mirror update locking**
- Mirror updates MUST be protected by a per-mirror lock file:
  - `${HOME}/.gitvend/mirrors/<mirror>.lock.json`
- When the lock is held, concurrent updates of the same mirror MUST be prevented.

**FR-007 — Lock metadata for troubleshooting**
- Lock files SHOULD contain metadata helpful for debugging, at minimum:
  - lock acquisition timestamp
  - owning process id (PID) when meaningful/available
- The lock file format MUST be simple and human-readable JSON.

**FR-008 — Cross-platform locking**
- Lock acquisition and release MUST behave reliably on Windows/macOS/Linux.
- The lock mechanism MUST prevent corruption under concurrent executions.

**FR-009 — Parallelism across mirrors**
- Updates to different mirrors SHOULD be allowed to proceed in parallel.
- Locking MUST be scoped such that only the same `<mirror>` blocks itself.

**FR-010 — Target Repo write safety (repo-level lock)**
- gitvend MUST prevent concurrent writes within the same Target Repo that could race on output paths.
- The mechanism MUST be a repo-level lock file.
- The lock file MUST be added to the Target Repo `.gitignore` automatically (if possible).
- The exact lock filename and lock metadata format MUST be defined in `doc/storage-and-locking.md` (e.g. `.gitvend/target-repo.lock.json`).

### Manifest-driven vendoring

**FR-011 — YAML manifest (v1)**
- gitvend MUST accept a YAML manifest as input to define sources and entries.
- The manifest MUST be versioned (e.g., `version: 1`) to support future evolution.

**FR-012 — Manifest model extensibility**
- gitvend MUST use an internal manifest DTO/model that can be serialized/deserialized from additional formats in the future (e.g., JSON) without changing core behavior.

**FR-013 — Source Repos**
- The manifest MUST support declaring multiple **Source Repos**, each including:
  - a stable local name/id
  - a Git repository URL
  - a configured default branch name (optional; defaults to `main` when not specified)
- The source default branch MAY be overridden per entry (see FR-018).

**FR-014 — Vendor Entries**
- The manifest MUST support multiple **Vendor Entries**, each specifying:
  - entry id
  - source reference
  - entry type (`file` or `dir`)
  - `from` path in Source Repo
  - `to` path in Target Repo
  - ref policy (see below)
- For developer convenience, gitvend SHOULD support a mode where `to` defaults to the same relative path as `from` (exact manifest rules defined in `doc/manifest-spec.md`).

**FR-015 — Path validation**
- gitvend MUST reject manifest entries that would write outside the Target Repo directory.
- Absolute paths and path traversal (e.g., `..`) MUST be disallowed for `to`.
- `from` MUST be validated to prevent injection into Git commands and to remain within repository paths.

**FR-015a — Target path base**
- Relative target paths (`to`) MUST be interpreted relative to the **Target Repo root** (the Git root containing the `.git` directory), not the current working directory.

### Ref resolution policies

**FR-016 — Branch detection (target repo)**
- gitvend MUST detect the current branch name of the target repository (or determine that it is detached HEAD).
- When there is no branch name available, gitvend MUST apply the configured ref policy fallback strategy (typically fallback to the source default branch unless `fixed-ref` is configured).

**FR-017 — Same-branch-first resolution**
- For policies that use same-branch behavior, gitvend MUST:
  1) attempt to resolve a branch in the source repo matching the target repo current branch name
  2) if that branch does not exist in the source repo, apply the configured fallback behavior

**FR-018 — Supported ref policies**
- gitvend MUST support at least the following policies:
  - `same-branch-else-fail`
  - `same-branch-else-default` (fallback to an effective default branch)
  - `fixed-ref` (explicit branch/tag/commit)
- For `same-branch-else-default`, the effective default branch MUST be resolved as:
  - entry-level override if provided
  - otherwise the source-level default branch
  - otherwise `main`.

**FR-019 — Default branch per source**
- Each source MUST have a configurable default branch name.
- If not configured, the default MUST be `main`.
- Legacy repos MUST be able to specify `master` (or other names) explicitly.

**FR-020 — Visibility of fallbacks**
- If a fallback is used (e.g., default branch instead of same-branch), gitvend MUST record that in the run report.

### Extraction and writing

**FR-021 — File extraction**
- For `file` entries, gitvend MUST extract content from the resolved commit using Git object access (e.g., `git show <sha>:<path>`).

**FR-022 — Directory extraction via ZIP archive**
- For `dir` entries, gitvend MUST vendor the directory recursively without requiring a working-tree checkout.
- gitvend MUST use a portable mechanism such as:
  - `git archive --format=zip <sha> <path>`
  - followed by unzip/extraction

**FR-023 — LFS behavior (v1)**
- gitvend MUST NOT implicitly fetch LFS objects in v1.
- If LFS is used in the source repository, vendored content MAY be LFS pointer files.

**FR-024 — Atomic writes**
- gitvend MUST ensure Target Repo writes are atomic and crash-safe:
  - write to a temporary file/directory
  - then move/rename into place
- Partial/incomplete output MUST NOT be left as the final target state.

**FR-025 — Directory swap behavior**
- For `dir` entries, gitvend MUST avoid partially updating a directory.
- A sync MUST either:
  - fully replace the directory contents atomically, or
  - leave the previous contents intact if an error occurs.

**FR-025a — Mirror self-healing**
- If a mirror is detected as corrupted or unusable, gitvend SHOULD recover automatically by re-initializing the mirror from scratch.

### Outputs: determinism, reporting, provenance

**FR-026 — Determinism lockfile**
- gitvend MUST write a determinism lockfile (Vendor Lockfile) next to the manifest.
- The lockfile name MUST be derived from the manifest filename (base name + `.lock.` + same extension), e.g. `gitvend.yml` -> `gitvend.lock.yml`.
- The lockfile MUST record resolved commit SHAs for each Source Repo/ref used during the run.

**FR-027 — Lockfile semantics (CI)**
- gitvend MUST support a deterministic CI workflow in which:
  - resolved SHAs can be pinned and re-used
  - changes that would alter resolved SHAs are detectable and reviewable
- The exact behavior (e.g., whether `check` fails on lock drift) MUST be defined in `doc/output-artifacts.md` and `doc/cli-spec.md`.

**FR-028 — Machine-readable report**
- gitvend MUST produce a machine-readable JSON report for each run (default name and location defined later).
- The report MUST include:
  - per-entry outcome (synced/unchanged/failed)
  - resolved ref + resolved SHA per entry (or per source/ref group)
  - whether fallback was used
  - error details when failures occur

**FR-029 — Provenance support**
- gitvend MUST provide provenance information sufficient to trace each vendored output to:
  - source repo URL
  - resolved ref name
  - resolved commit SHA
  - source path
- Provenance MUST be available at least via the report and lockfile.
- Inline provenance markers in vendored files SHOULD be supported as an optional feature (enable/disable via configuration).

### Commands: sync vs check

**FR-030 — `sync` behavior**
- `gitvend sync` MUST:
  - read the manifest
  - resolve refs per policy
  - fetch mirrors (at most once per mirror per invocation)
  - vendor files/dirs into the Target Repo
  - write/update the vendor lockfile (next to the manifest)
  - write a JSON report

**FR-031 — `check` behavior**
- `gitvend check` MUST:
  - verify that vendored outputs match what would be produced from the manifest and resolved refs
  - exit non-zero on drift or configuration errors
  - NOT modify vendored outputs
- `check` MAY update mirrors if needed for verification (but must not alter the workspace outputs).

### Developer experience, initialization, and multi-repo workflows

**FR-032 — Manifest discovery (default behavior)**
- If no manifest path is provided, gitvend MUST search for a manifest starting at the current working directory and traversing upward.
- Traversal MUST stop at the repository root (the directory containing `.git`).
- If a manifest is not found within that boundary, gitvend MUST fail with a clear error.

**FR-033 — Multiple manifests in one invocation**
- gitvend SHOULD support syncing multiple repositories in one invocation by accepting multiple manifest paths.
- When multiple manifests reference the same source repository, gitvend MUST share the same mirror and avoid redundant fetches (within the invocation).

**FR-034 — Convenience commands for manifest maintenance**
- gitvend SHOULD provide convenience commands to create/update manifests, including:
  - adding a source
  - adding an entry (syncing a file/dir from a source into a target path)
- These commands MUST validate inputs and produce a manifest that conforms to `doc/manifest-spec.md`.

**FR-035 — Automatic initialization**
- gitvend MUST create required directories under `${HOME}/.gitvend/` on demand (e.g., `mirrors/`).
- gitvend SHOULD provide an initialization workflow that can create any required per-repo scaffolding (e.g., recommended `.gitignore` entries), defined in `doc/cli-spec.md`.

**FR-036 — Idempotent sync**
- Repeated `sync` runs MUST be safe and idempotent: if sources and resolved SHAs did not change, outputs MUST remain unchanged.

**FR-037 — Golden source overwrite semantics**
- For any file/directory declared in the manifest, the source repository content MUST be treated as the golden source.
- If local modifications exist for vendored outputs, gitvend MUST NOT overwrite them when they are uncommitted; it must fail to prevent work loss.
- If local modifications are already committed, a later `sync` may overwrite them (normal git history applies).

**FR-038 — Automatic commit of vendored changes**
- When `sync` modifies vendored outputs, gitvend MUST be able to automatically create a Git commit in the target repo.
- Auto-commit MUST be configurable in the manifest and MUST be overridable by environment variable.
- If enabled, gitvend MUST stage and commit only the vendored outputs it manages (not unrelated workspace changes).
- Default dirty-policy for auto-commit: `allow-unrelated`.
  - Uncommitted changes outside managed paths do not block auto-commit.
  - Uncommitted changes inside managed paths MUST block the run (to prevent overwriting local work).
- The commit message MUST include audit information, including:
  - source repository identifiers
  - resolved commit SHAs used
  - a summary of managed paths changed (or count).

### Distribution and CI execution modes

**FR-039 — Containerized execution (optional distribution channel)**
- gitvend SHOULD provide a container image suitable for CI usage.
- The image SHOULD be published to GitHub Container Registry for the project repository.

---

## Non-functional requirements

**NFR-001 — Determinism**
- Given the same manifest and the same resolved SHAs, gitvend MUST produce identical outputs.
- CI verification MUST be stable and repeatable.

**NFR-002 — Performance (mirror-first)**
- gitvend SHOULD avoid full repository clones and working-tree checkouts.
- gitvend SHOULD leverage bare mirrors and object-level extraction (`git show`, `git archive`).

**NFR-003 — CI friendliness**
- gitvend MUST be easy to run in common CI systems (GitHub Actions, GitLab CI).
- The tool SHOULD provide clear exit codes and machine-readable reports.

**NFR-004 — Portability**
- gitvend MUST support Windows/macOS/Linux.
- The implementation MUST avoid platform-specific dependencies where possible.
- Directory extraction MUST be portable (ZIP-based approach is the baseline).

**NFR-005 — Security: no secrets in manifests**
- Manifests MUST NOT contain credentials.
- gitvend MUST avoid logging sensitive tokens (if provided via env/CI).

**NFR-006 — Security: safe filesystem writes**
- gitvend MUST prevent writing outside the target repository.
- gitvend MUST handle untrusted manifest inputs safely (path traversal prevention).

**NFR-007 — Security: command execution safety**
- If gitvend invokes external `git`, arguments MUST be passed without shell interpolation.
- Inputs from manifests MUST be validated/escaped to prevent command injection.

**NFR-008 — Reliability: crash safety**
- Interruption during sync MUST not leave corrupted mirrors or partially written outputs.
- Atomic write strategies MUST be used for both files and directories.

**NFR-009 — Reliability: lock timeout and stale-lock recovery**
- gitvend MUST support a lock timeout.
- gitvend SHOULD provide a safe stale-lock recovery strategy (defined in `doc/storage-and-locking.md`).

**NFR-010 — Observability: logs**
- gitvend MUST provide readable logs suitable for local use.
- Logs SHOULD include key resolution decisions and fallback usage.

**NFR-011 — Observability: report**
- A JSON report MUST be produced with sufficient detail for CI diagnostics.

**NFR-012 — Backward compatibility and versioning**
- Manifest schema MUST be versioned.
- gitvend SHOULD preserve backward compatibility within a major version (v1.x).

**NFR-013 — Maintainability**
- The codebase SHOULD be structured to keep:
  - manifest parsing
  - ref resolution
  - mirror management
  - extraction
  - workspace writes
  - reporting/lockfiles
  as clearly separated modules.

**NFR-014 — Self-contained CI execution**
- gitvend SHOULD offer a self-contained distribution option for CI (container image) to reduce environment variability.

---

## Clarifications that may be needed (not blocking this document)

The requirements above are implementable, but the following details should be explicitly decided in later documents to remove remaining ambiguity:

1. **Lockfile/report locations**: Paths are decided in `doc/cli-spec.md` and `doc/open-questions-and-decisions.md`; remaining questions belong to `doc/output-artifacts.md` (field schema, lifecycle).
2. **Inline provenance defaults**: default off vs default on (and which file types are eligible).
3. **Git dependency**: minimal supported Git version (Decided: Git v2).
4. **Auto-commit toggles**: field/env naming (if any) and any additional policy knobs beyond the decided default dirty-policy (`allow-unrelated`).
5. **Manifest discovery**: default manifest filenames and precedence if multiple are found.
