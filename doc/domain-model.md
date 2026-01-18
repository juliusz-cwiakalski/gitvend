# gitvend — Domain Model

This document defines the core domain entities, their relationships, the main state transitions during `sync` and `check`, and a stable error taxonomy for v1.

The model is aligned with:
- `doc/glossary.md`
- `doc/prd.md`
- `doc/requirements.md`

---

## Naming and file conventions (v1)

gitvend uses two distinct “lock” concepts:

- **Vendor Lockfile** (determinism/pinning): next to the manifest, named `<manifest-base>.lock.<ext>` (e.g., `gitvend.lock.yml`)
- **Locks** (concurrency): `*.lock.json`

Concrete locations:

- **Mirror Lock** (per Source Repo mirror): `${HOME}/.gitvend/mirrors/<mirror>.lock.json`
- **Target Repo Lock** (per target repository): `<target-repo>/.gitvend/target-repo.lock.json`

---

## 1) Core entities

### 1.1 Manifest

**Manifest** (YAML v1) is the declarative configuration that defines:
- a set of **Source Repos** (remote repositories)
- a set of **Vendor Entries** (vendoring rules)
- optional settings affecting resolution, locking, reporting, and auto-commit.

**Key fields (conceptual):**
- `version`
- `settings` (optional)
- `sources[]`
- `entries[]`

---

### 1.2 Source Repo

A **Source Repo** represents one remote Git repository.

**Attributes (conceptual):**
- `sourceId` (stable name in the manifest)
- `repoUrl`
- `defaultBranch` (optional; defaults to `main`)
- `authHint` (optional; informational only—no secrets)

A Source Repo is materialized locally via a **Mirror**.

---

### 1.3 Vendor Entry

A **Vendor Entry** is one vendoring rule: file or directory copied from a **Resolved Revision** to a **Target Path**.

**Attributes (conceptual):**
- `entryId`
- `sourceId` (reference to Source Repo)
- `type` (`file` | `dir`)
- `fromPath` (path in source repo)
- `toPath` (**Target Path** in target repo, relative to target repo root)
- `refPolicy` (see below)
- `defaultToFromPath` (optional behavior where `toPath` defaults to `fromPath` when omitted)

---

### 1.4 RefPolicy (and Resolved Revision)

**RefPolicy** defines how gitvend chooses the revision to vendor from.

**Supported policies (v1):**
- `same-branch-else-fail`
- `same-branch-else-default`
- `fixed-ref`

**Resolved Revision** is the result of applying the policy:
- `requestedPolicy`
- `targetBranchName` (may be null when detached HEAD)
- `candidateRef` (e.g., `refs/heads/feature/x`)
- `fallbackUsed` (boolean)
- `effectiveRefName` (branch/tag name if applicable)
- `resolvedCommitSha` (full SHA)

Notes:
- For `same-branch-*`, gitvend tries the target repo current branch name first.
- If the same branch does not exist in the source repo:
  - `same-branch-else-fail` fails
  - `same-branch-else-default` falls back to the effective default branch
- Effective default branch is resolved as: entry override (if present) → source default → `main`.

---

### 1.5 Mirror

A **Mirror** is a local bare Git repository that caches a Source Repo.

**Mirror storage:**
- `${HOME}/.gitvend/mirrors/<mirror>.git`

**MirrorId:**
- `<mirror>` is a deterministic, filesystem-safe, human-readable slug derived from a normalized `repoUrl`.

**Mirror metadata (optional, recommended):**
- `${HOME}/.gitvend/mirrors/<mirror>.meta.json`
  - `repoUrl`
  - `createdAt`
  - `lastFetchedAt`
  - `gitVersionObserved`

**Mirror Lock:**
- `${HOME}/.gitvend/mirrors/<mirror>.lock.json` (JSON)

---

### 1.6 Target Repo

A **Target Repo** is the Git repository where vendored content is written.

**Key derived properties:**
- `repoRoot` (directory containing `.git`)
- `currentBranchName` (or null if detached)
- `manifestPath` (explicit, or discovered)

**Target Repo Lock (repo-level):**
- `<target-repo>/.gitvend/target-repo.lock.json` (JSON)

---

### 1.7 Output artifacts

#### Vendor Lockfile (`<manifest-base>.lock.<ext>`)
Records resolved SHAs to support deterministic CI.

Naming rules:
- Manifest `gitvend.yml` -> lockfile `gitvend.lock.yml`
- Manifest `gitvend.yaml` -> lockfile `gitvend.lock.yaml`
- Manifest `.gitvend.yml` -> lockfile `.gitvend.lock.yml`
- Manifest `.gitvend.yaml` -> lockfile `.gitvend.lock.yaml`

**Conceptual content:**
- run metadata (timestamp, tool version)
- per Source Repo / effective ref:
  - `repoUrl`
  - `effectiveRefName`
  - `resolvedCommitSha`

#### Run Report (`<target-repo>/.gitvend/gitvend.report.json`)
Machine-readable JSON report summarizing a run.

If the report already exists, it is rotated to `<target-repo>/.gitvend/gitvend.report-<iso-timestamp>.json` before writing a new one.

**Conceptual content:**
- per Vendor Entry:
  - status: `synced` | `unchanged` | `failed`
  - resolved ref + SHA
  - fallbackUsed
  - timings (optional)
  - error (if failed)

#### Provenance markers (optional)
Optional inline markers or sidecar metadata that trace outputs back to source repo/ref/SHA/path.

---

### 1.8 Auto-commit

If enabled (configurable), after successful sync:
- gitvend stages only managed vendored outputs
- creates a commit with an audit-friendly message containing:
  - affected managed paths (summary)
  - source repo identifiers
  - resolved SHAs used

Behavior with unrelated dirty changes is specified in `doc/cli-spec.md` / `doc/storage-and-locking.md`.

---

## 2) Relationships

- A **Manifest** contains many **Source Repos** and many **Vendor Entries**.
- Each **Vendor Entry** references exactly one **Source Repo**.
- Each **Source Repo** maps to exactly one **Mirror** (via MirrorId).
- Each **Vendor Entry** produces output under a **Target Path** in the **Target Repo**.
- A run produces output artifacts:
   - vendor lockfile (next to the manifest)
   - run report JSON
   - optional provenance
- A run acquires locks:
  - mirror lock during mirror update
  - target repo lock during writing/staging/committing

---

## 3) State transitions

### 3.1 High-level run state machine

A `sync` or `check` invocation transitions through these phases:

1. **Boot**
   - validate `git` availability/version
   - determine `GITVEND_HOME` (or default)

2. **Target Repo discovery**
   - determine repo root (must contain `.git`)
   - discover manifest (unless provided)

3. **Acquire Target Repo lock**
   - acquire repo-level lock (`.gitvend/target-repo.lock.json`)

4. **Load + validate manifest**
   - parse YAML
   - validate schema and paths

5. **Plan**
   - compute effective entries (including defaults)
   - group work by Source Repo

6. **Resolve refs**
   - detect target repo branch name (or detached)
   - resolve effective source ref per Vendor Entry
   - resolve to commit SHA

7. **Ensure mirrors updated (sync/check)**
   - for each required Mirror:
     - acquire mirror lock
     - fetch once per invocation
     - release mirror lock

8. **Execute**
   - `sync`:
     - extract file/dir content
     - perform atomic writes
     - update Vendor Lockfile
     - write Run Report
     - stage/commit (if enabled)
   - `check`:
     - compute expected content
     - compare with target repo
     - emit Run Report
     - exit non-zero on drift

9. **Release Target Repo lock**

10. **Exit**
   - success or error (with stable code)

---

### 3.2 Mirror lifecycle

Mirror states:
- **Absent** → (init) → **Ready**
- **Ready** → (fetch) → **Updated**
- **Updated** → (use) → **Ready**
- **Any** → (detect corruption) → **Corrupt** → (reinit) → **Ready**

A mirror update is always protected by the mirror lock.

---

### 3.3 Vendor Entry processing lifecycle (sync)

Each Vendor Entry progresses through:
- **Planned**
- **Resolved** (effective ref + SHA)
- **Extracted** (file via `git show`, dir via `git archive --format=zip`)
- **StagedWrite** (temp file/dir written)
- **CommittedWrite** (atomic rename into final Target Path)
- **Reported** (entry status recorded)

If no content changes are detected, Vendor Entry may short-circuit to:
- **Unchanged**

---

### 3.4 Vendor Entry processing lifecycle (check)

Each Vendor Entry progresses through:
- **Planned**
- **Resolved**
- **ComputedExpected**
- **Compared**
- **Reported**

If drift is detected, Vendor Entry result is **Drifted** (reported) and the overall run exits non-zero.

---

## 4) Domain invariants

The following invariants must hold for correct behavior:

1. **Golden source invariant**
   - For paths declared in the manifest, the source repository is the single source of truth.
   - Local edits to managed vendored paths are overwritten on `sync`.

2. **No escape invariant**
   - Vendored outputs must never be written outside the target repo root.

3. **Determinism invariant**
   - Given the same resolved SHAs and manifest, outputs are identical.

4. **Mirror integrity invariant**
   - Mirror corruption is detected and healed (by reinit) rather than propagated.

5. **Idempotency invariant**
   - Re-running `sync` without upstream changes results in no output change.

---

## 5) Error taxonomy

### 5.1 Error model

Errors are emitted with a stable, machine-parseable code.

**Error (conceptual):**
- `code` (stable, e.g., `GV_REF_NOT_FOUND`)
- `message` (human-readable)
- `category` (see below)
- `retryable` (boolean)
- `context` (structured fields useful for debugging)
  - e.g., `repoUrl`, `mirrorId`, `manifestPath`, `entryId`, `fromPath`, `toPath`, `ref`, `sha`
- `cause` (optional nested error)

### 5.2 Categories

- `BOOT` — environment/tooling prerequisites
- `MANIFEST` — manifest parsing/validation
- `TARGET_REPO` — target repo discovery/lock/dirty state
- `MIRROR` — mirror init/fetch/health
- `LOCK` — lock acquisition/release/timeout
- `REF` — ref resolution / branch/tag lookup
- `EXTRACT` — retrieving objects/archives from Git
- `IO` — filesystem operations and atomic writes
- `REPORT` — vendor lockfile/run report/provenance outputs
- `GIT` — git command failures not otherwise classified
- `AUTH` — authentication/authorization failures

### 5.3 Stable error codes (v1)

#### BOOT
- `GV_GIT_NOT_FOUND` — `git` not available on PATH
- `GV_GIT_VERSION_UNSUPPORTED` — Git version < minimum supported
- `GV_HOME_INIT_FAILED` — cannot create `${HOME}/.gitvend` structure

#### MANIFEST
- `GV_MANIFEST_NOT_FOUND` — manifest discovery failed within repo boundary
- `GV_MANIFEST_READ_FAILED` — cannot read manifest file
- `GV_MANIFEST_PARSE_FAILED` — YAML parse error
- `GV_MANIFEST_VERSION_UNSUPPORTED` — unsupported `version`
- `GV_MANIFEST_INVALID` — schema validation failed
- `GV_SOURCE_ID_UNKNOWN` — entry references missing source
- `GV_DUPLICATE_ENTRY_ID` — duplicate entry ids
- `GV_DUPLICATE_TARGET` — two entries write to the same target path
- `GV_TARGET_OUTSIDE_REPO` — `to` resolves outside repo root
- `GV_PATH_TRAVERSAL` — `to` contains traversal patterns
- `GV_FROM_PATH_INVALID` — invalid `from` path (unsafe or malformed)
- `GV_REF_POLICY_INVALID` — ref policy invalid or unsupported

#### TARGET_REPO
- `GV_NOT_A_GIT_REPO` — `.git` not found / not a repo
- `GV_REPO_ROOT_NOT_FOUND` — cannot determine repo root
- `GV_BRANCH_DETECT_FAILED` — cannot determine current branch
- `GV_TARGET_REPO_LOCK_TIMEOUT` — repo-level lock timeout
- `GV_TARGET_REPO_LOCK_STALE` — stale repo-level lock detected (policy-defined)
- `GV_DIRTY_WORKSPACE_UNMANAGED` — unrelated dirty changes prevent safe auto-commit (policy-defined)

#### MIRROR / LOCK
- `GV_MIRROR_ID_COLLISION` — slug collision detected and cannot be resolved deterministically
- `GV_MIRROR_LOCK_TIMEOUT` — per-mirror lock timeout
- `GV_MIRROR_LOCK_STALE` — stale per-mirror lock detected (policy-defined)
- `GV_MIRROR_INIT_FAILED` — failed to initialize bare mirror
- `GV_MIRROR_FETCH_FAILED` — `git fetch` failed
- `GV_MIRROR_CORRUPT` — mirror unusable/corrupt detected
- `GV_MIRROR_HEAL_FAILED` — failed to reinitialize a corrupt mirror

#### AUTH / REMOTE
- `GV_AUTH_FAILED` — authentication failed (SSH/HTTPS)
- `GV_REMOTE_UNREACHABLE` — network unreachable / DNS / timeout
- `GV_REMOTE_FORBIDDEN` — insufficient permissions

#### REF
- `GV_REF_NOT_FOUND` — required ref not found in source repo
- `GV_REF_AMBIGUOUS` — ref resolves ambiguously
- `GV_REF_POLICY_VIOLATION` — policy requires same-branch but fallback attempted/forbidden
- `GV_FIXED_REF_INVALID` — fixed ref specified but invalid

#### EXTRACT
- `GV_SOURCE_PATH_NOT_FOUND` — `from` path not found at resolved SHA
- `GV_GIT_SHOW_FAILED` — file extraction via `git show` failed
- `GV_GIT_ARCHIVE_FAILED` — directory archive creation failed
- `GV_ARCHIVE_EXTRACT_FAILED` — unzip/extract failed
- `GV_LFS_POINTER_ONLY` — LFS content remains pointers (warning-level code; not fatal unless configured)

#### IO
- `GV_TEMP_PATH_CREATE_FAILED` — cannot create temp file/dir
- `GV_IO_READ_FAILED` — filesystem read failed
- `GV_IO_WRITE_FAILED` — filesystem write failed
- `GV_IO_PERMISSION_DENIED` — permission issue
- `GV_ATOMIC_RENAME_FAILED` — atomic move/rename failed
- `GV_DISK_FULL` — insufficient space

#### REPORT
- `GV_VENDOR_LOCKFILE_READ_FAILED` — cannot read the vendor lockfile
- `GV_VENDOR_LOCKFILE_PARSE_FAILED` — vendor lockfile format invalid
- `GV_VENDOR_LOCKFILE_WRITE_FAILED` — cannot write the vendor lockfile
- `GV_REPORT_WRITE_FAILED` — cannot write run report JSON
- `GV_PROVENANCE_WRITE_FAILED` — provenance marker/sidecar write failed
- `GV_DRIFT_DETECTED` — `check` found drift

#### GIT (generic)
- `GV_GIT_CMD_FAILED` — an invoked git command failed (non-specific)

---

## 6) Warnings vs errors

gitvend distinguishes:
- **Errors**: fail the command and return non-zero.
- **Warnings**: recorded in the JSON report but do not fail by default.

In v1:
- `GV_LFS_POINTER_ONLY` is a warning unless a strict mode is enabled.

Strict-mode semantics and exit codes are specified in `doc/cli-spec.md`.

---

## 7) Open alignment points (tracked elsewhere)

This domain model intentionally does not finalize:
- run report filename and default location
- exact mirror-id collision strategy

These remain tracked in `doc/open-questions-and-decisions.md` and will be resolved in `doc/storage-and-locking.md` and `doc/output-artifacts.md`.

