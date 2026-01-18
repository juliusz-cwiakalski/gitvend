# gitvend — Resolution and Sync Algorithm (v1)

This document defines the **deterministic resolution** and **sync/check execution algorithm** for gitvend v1.

It is a normative spec for implementers: where this document uses **MUST/SHOULD/MAY**, that language is intentional.

---

## 0) Goals

- Deterministically resolve each **Vendor Entry** to a **Resolved Revision** (full commit SHA).
- Vendor files and directories **without checking out** a working tree of Source Repos.
- Guarantee safety:
  - no writes outside the **Target Repo**
  - no partial outputs on failure (atomic writes)
  - robust cross-platform locking
- Ensure “fast on repeat” behavior:
  - per-run **mirror update deduplication**
  - reuse local **Mirror Repos**

---

## 1) Inputs

### 1.1 Invocation context

A single `gitvend` invocation operates on one or more **Manifest Contexts**.

A **Manifest Context** is:
- `targetRepoRoot` — filesystem path
- `manifestPath` — filesystem path
- `mode` — `sync` or `check`
- effective CLI options (e.g., `--strict`, `--lock-timeout`, `--report`, `--ci`, `--fail-on-fallback`, `--use-lockfile`, commit/dirty policy)

### 1.2 CLI/Env override (Resolution Override)

The resolution algorithm supports an optional **Resolution Override** input:

- **Override Branch Name**: a branch name treated as the Target Repo “current branch name” for resolution purposes.

**Status:** if this override is not explicitly present in `doc/cli-spec.md`, it MUST be treated as **not implemented in v1** (see Open Questions).

### 1.3 Canonical file conventions (recap)

- Vendor Lockfile (determinism): next to the manifest, `<manifest-base>.lock.<ext>` (e.g., `gitvend.lock.yml`)
- Locks (concurrency): `*.lock.json`
  - Mirror Lock: `${HOME}/.gitvend/mirrors/<mirror>.lock.json`
  - Target Repo Lock: `<target-repo>/.gitvend/target-repo.lock.json`

---

## 2) High-level flow

For each Manifest Context:

1. **Boot**
   - Verify `git` is available on `PATH`.
   - Resolve `GITVEND_HOME` (default `${HOME}/.gitvend`).

2. **Discover Target Repo and Manifest**
   - Determine `targetRepoRoot` (directory containing `.git`).
   - Determine `manifestPath` (explicit `--manifest` or discovery).

3. **Acquire Target Repo Lock**
   - Acquire `<target-repo>/.gitvend/target-repo.lock.json`.

4. **Load and validate manifest**
   - Parse YAML.
   - Validate schema and paths.
   - Expand entries (normalize defaults, expand `paths` bundles to concrete operations).

5. **Ensure Mirror Repos are ready**
   - For each referenced Source Repo:
     - compute `mirrorId` and paths
     - initialize bare mirror if absent

6. **Update mirrors (deduplicated)**
   - Fetch each required mirror at most once per invocation.

7. **Resolve revisions (per entry)**
   - Detect Target Repo “current branch name”.
   - Apply precedence rules and policies to obtain a **Resolved Revision**.

8. **Execute**
   - `sync`: extract + atomic write + vendor lockfile + report + optional auto-commit
   - `check`: compute expected + compare + report; never modify vendored outputs

9. **Release Target Repo Lock**

For multi-manifest invocations:
- The tool MAY process Target Repos sequentially (v1) but MUST still deduplicate mirror fetches across contexts.

---

## 3) Target Repo branch detection

### 3.1 Repo root

`targetRepoRoot` is the directory containing `.git`.

Recommended detection (from current working directory):

- Determine repo root:
  - `git rev-parse --show-toplevel`

If this fails, treat as `GV_NOT_A_GIT_REPO`.

### 3.2 Branch name detection

gitvend MUST detect the Target Repo current branch name as follows:

1. `git rev-parse --abbrev-ref HEAD`
   - If result is not `HEAD`, that value is `currentBranchName`.

2. If result is `HEAD`, the repo is in detached HEAD; set `currentBranchName = null`.

3. If an **Override Branch Name** is implemented and provided, it MUST replace `currentBranchName`.

### 3.3 Detached HEAD handling

If `currentBranchName = null`:
- For `same-branch-else-fail`: MUST fail resolution.
- For `same-branch-else-default`: MUST fall back to the effective default branch (see §5.3) and mark `fallbackUsed=true` with `fallbackReason=DETACHED_HEAD`.
- For `fixed-ref`: unaffected.

---

## 4) Mirror update and branch existence checks

### 4.1 Mirror initialization

For each Source Repo:

- Derive a deterministic `mirrorId` from `repoUrl` (per OQ-013 rules).
- Mirror Repo path:
  - `${GITVEND_HOME}/mirrors/<mirrorId>.git`

If mirror repo does not exist:
- Initialize bare mirror:
  - `git clone --mirror <repoUrl> <mirrorPath>`

### 4.2 Mirror update (fetch)

A mirror update MUST be guarded by the **Mirror Lock**:
- `${GITVEND_HOME}/mirrors/<mirrorId>.lock.json`

Mirror update command (recommended baseline):
- `git -C <mirrorPath> fetch --prune --tags`

Notes:
- A mirror MUST be fetched at most once per process invocation per mirrorId.
- If mirror is detected as corrupted (git commands fail in ways indicating corruption), gitvend SHOULD attempt self-heal by deleting and recreating the mirror.

### 4.3 How “branch exists in source” is checked

After mirror fetch, branch existence MUST be checked via mirror refs:

- A branch `<b>` exists iff `refs/heads/<b>` exists in the mirror.

Recommended command:
- `git -C <mirrorPath> show-ref --verify --quiet refs/heads/<b>`

If exit code is 0 → exists.

Tag existence check (for fixed-ref if user supplies a tag):
- `git -C <mirrorPath> show-ref --verify --quiet refs/tags/<t>`

If the user supplies a general “commit-ish”:
- Resolve using:
  - `git -C <mirrorPath> rev-parse --verify <commitish>^{commit}`

---

## 5) Resolved Revision algorithm

A **Resolved Revision** is the result of applying the ref policy for a Vendor Entry:
- `requestedPolicy`
- `targetBranchName` (may be null)
- `candidateRef` (e.g., `refs/heads/feature/x`)
- `fallbackUsed` (boolean)
- `fallbackReason` (enum)
- `effectiveRefName` (string: branch/tag name or commit-ish)
- `resolvedCommitSha` (full SHA)

### 5.1 Resolution precedence order (per entry)

Resolution MUST follow this precedence:

1. **Explicit override** (flag/env) — if implemented
   - When set, treat override as the Target Repo branch name.

2. **Policy-driven same-branch** (if policy is `same-branch-*`)
   - If `targetBranchName` is non-null and exists in Source Repo, use it.

3. **Fallback policy** (only for `same-branch-else-default`)
   - Use effective default branch.

4. **Default branch**
   - When no other fallback is configured, use `main`.

### 5.2 Policy: `same-branch-else-fail`

Inputs:
- `targetBranchName` (must be non-null)

Algorithm:
1. If `targetBranchName` is null → fail with `GV_REF_POLICY_VIOLATION`.
2. If `refs/heads/<targetBranchName>` exists in Source Repo mirror:
   - `effectiveRefName = <targetBranchName>`
   - `resolvedCommitSha = rev-parse refs/heads/<targetBranchName>^{commit}`
   - `fallbackUsed = false`
3. Else fail with `GV_REF_NOT_FOUND`.

### 5.3 Policy: `same-branch-else-default`

Inputs:
- `targetBranchName` (may be null)
- `defaultBranchOverride` (optional, entry-level)
- `sourceRepo.defaultBranch` (optional, defaults to `main`)

Effective default branch resolution order:
1. `vendorEntry.defaultBranchOverride` (if present and non-empty)
2. `sourceRepo.defaultBranch` (if present and non-empty)
3. `main`

Algorithm:
1. If `targetBranchName` is non-null and `refs/heads/<targetBranchName>` exists:
   - use same-branch; `fallbackUsed=false`
2. Else:
   - set `fallbackUsed=true`
   - `fallbackReason = (targetBranchName is null ? DETACHED_HEAD : BRANCH_NOT_FOUND)`
   - let `b = effectiveDefaultBranch`
   - verify `refs/heads/<b>` exists; if missing, fail with `GV_REF_NOT_FOUND` (default branch missing is a repo/config error)
   - resolve SHA from `refs/heads/<b>`

### 5.4 Policy: `fixed-ref`

Inputs:
- `vendorEntry.ref` (required)

Algorithm:
1. Let `r = vendorEntry.ref`.
2. Resolve `r` to a commit SHA:
   - `git -C <mirrorPath> rev-parse --verify <r>^{commit}`
3. If resolution fails → `GV_FIXED_REF_INVALID`.
4. `fallbackUsed=false`.

### 5.5 Fallback visibility and failure

- If `--fail-on-fallback` is set (or strict policy requires it), any resolution that sets `fallbackUsed=true` MUST fail with `GV_REF_POLICY_VIOLATION`.
- Otherwise, fallback MUST be recorded in the Run Report.

---

## 6) Planning and grouping rules

### 6.1 Expand `paths` bundles

A `vendorEntries[].type: paths` is syntactic sugar.

Expansion rules:
- Resolve the Source Repo and policy once.
- Expand into per-path operations with:
  - same `entryId` plus a stable sub-identifier (e.g., `entryId#1` or `entryId:<fromPath>` normalized)
  - separate Target Paths for conflict checking and execution

The Run Report MUST allow attribution back to the parent Vendor Entry.

### 6.2 Duplicate and overlap detection

Before any extraction:
- compute normalized Target Paths for all operations across all manifests in the invocation
- fail on:
  - duplicate Target Paths
  - overlap/prefix conflicts

---

## 7) Extraction mechanisms (object-level; no checkout)

All extraction occurs from the **Mirror Repo**, using the `resolvedCommitSha`.

### 7.1 File extraction

For a `file` operation:

- Command:
  - `git -C <mirrorPath> show <sha>:<fromPath>`

The output bytes are the file content.

If the path is missing at that SHA:
- fail with `GV_SOURCE_PATH_NOT_FOUND`.

### 7.2 Directory extraction

For a `dir` operation:

- Command:
  - `git -C <mirrorPath> archive --format=zip <sha> <fromPath>`

Then extract the ZIP into a temp directory (implementation MUST not rely on external `unzip`; use a built-in ZIP library).

Constraints:
- `fromPath` MUST end with `/` in v1.

If the path is missing at that SHA:
- fail with `GV_SOURCE_PATH_NOT_FOUND`.

---

## 8) Atomic write strategy (temp → rename)

Atomicity is required to ensure a failed run does not leave partial outputs.

### 8.1 Common principles

- All writes MUST occur under the Target Repo root.
- Writes MUST be performed into a temp path first.
- Finalization MUST be a rename/move step.
- If the platform cannot rename over an existing destination atomically, gitvend MUST implement a safe replace sequence.

### 8.2 Temp layout

Recommended temp root:
- `<target-repo>/.gitvend/tmp/<runId>/...`

This directory MUST be ignored (auto-add to `.gitignore`).

### 8.3 Atomic write for files

Given final Target Path `T`:

1. Create temp file `T.tmp.<runId>` in the same directory as `T` (or under `.gitvend/tmp` plus a safe move).
2. Write full content.
3. fsync if available (best-effort).
4. Replace strategy:
   - If destination does not exist: rename temp → `T`.
   - If destination exists:
     - Prefer atomic replace when supported.
     - Otherwise:
       1) rename `T` → `T.old.<runId>`
       2) rename temp → `T`
       3) delete `T.old.<runId>`

If any step fails, attempt to roll back, and leave prior `T` intact whenever possible.

### 8.4 Atomic write for directories

Given final directory Target Path `D`:

1. Extract zip into temp directory `D.tmp.<runId>/`.
2. Validate extracted tree does not contain path traversal (ZIP slip prevention):
   - no absolute paths
   - no `..` segments
3. Replace strategy:
   - If `D` does not exist: rename/move temp dir → `D`.
   - If `D` exists:
     1) rename `D` → `D.old.<runId>`
     2) rename/move `D.tmp.<runId>` → `D`
     3) delete `D.old.<runId>`

If rename fails due to platform restrictions (e.g., Windows directory in-use), gitvend MUST fail with `GV_ATOMIC_RENAME_FAILED` and MUST leave the original `D` intact.

### 8.5 Crash recovery

On startup (after acquiring Target Repo Lock), gitvend SHOULD clean stale temp artifacts:
- `<target-repo>/.gitvend/tmp/*`
- `*.old.<runId>` older than a threshold

Cleanup MUST never delete the final Target Paths unless they are clearly gitvend temp artifacts.

---

## 9) `sync` vs `check` behavior

### 9.1 `sync` behavior

After successful planning, mirror update, and resolution:

For each operation:
1. Extract expected content from mirror at `resolvedCommitSha`.
2. Compare to existing Target Path content.
3. If unchanged: record `unchanged`.
4. If changed:
   - write using atomic strategy
   - record `synced`

After all operations:
- Write/update Vendor Lockfile.
- Write Run Report.
- Auto-commit managed changes if enabled.

### 9.2 `check` behavior

`check` MUST NOT modify vendored outputs.

For each operation:
1. Extract expected content into temp location (or in-memory for small files).
2. Compare to current Target Path.
3. If mismatch: record drift.

After all operations:
- Write Run Report.
- Exit with drift exit code if any drift detected.

### 9.3 Lockfile usage in `check`

If `--use-lockfile` is set:
- Prefer SHAs recorded in the Vendor Lockfile.
- If lockfile is missing/incomplete:
  - fall back to manifest resolution, and record this in the report.

`check` MUST NOT update the Vendor Lockfile.

---

## 10) CI mode semantics (`--ci`)

When `--ci` is enabled:

- Behavior MUST be non-interactive.
- Output formatting SHOULD be stable (e.g., JSON logs if enabled).
- Run Report MUST be written (default or explicit `--report`).

Recommended additional CI behaviors:
- ensure mirror fetch occurs (so refs are accurate)
- if `sync` is run in CI, Vendor Lockfile MUST be written/updated

---

## 11) Vendor Lockfile update rules

Vendor Lockfile is written next to the manifest.

On `sync`:
- The lockfile MUST record, at minimum, for each Source Repo used:
  - `repoUrl`
  - the effective ref name used per policy
  - the resolved full commit SHA

If multiple Vendor Entries use the same Source Repo but different resolved revisions (possible with `fixed-ref` per entry), the lockfile MUST capture each distinct resolved revision in a deterministic structure (e.g., keyed by `sourceId` plus effectiveRefName).

---

## 12) Auto-commit algorithm (managed-only)

When auto-commit is enabled:

1. Compute the set of **managed Target Paths** from the manifest(s).
2. Validate dirty policy:
   - If any managed Target Path has uncommitted changes before sync → fail.
   - If `dirtyPolicy=allow-unrelated`, uncommitted changes outside managed paths are allowed.
3. Stage only managed paths:
   - `git add -- <managed-paths>`
4. If staging produces no changes, do not commit.
5. Commit with audit-friendly message including:
   - manifest path(s)
   - Source Repo(s)
   - resolved SHAs

---

## 13) Open questions / decisions

1) **Resolution Override surface (flag/env):**
   - Do you want a v1 supported override such as `--branch <name>` and/or `GITVEND_BRANCH`?
   - If yes, we should add it to `doc/cli-spec.md` and the manifest spec only if needed.

2) **Parallel execution:**
   - v1 can remain sequential; confirm if any parallelism is desired (e.g., per Target Repo or per Source Repo) since it affects lock hold times.

3) **Directory replace on Windows in-use paths:**
   - Policy on failure: fail hard (recommended) vs retry/backoff.

4) **Vendor Lockfile key shape for multiple revisions per Source Repo:**
   - Confirm the canonical structure: by `sourceId -> refs[]` or `sourceId -> revisionsByRef`.

5) **Report content expectations for drift:**
   - Do you want per-entry byte diffs/hashes only, or also a “first differing path” summary for directories?

