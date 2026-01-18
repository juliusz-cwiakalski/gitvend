# gitvend — CLI Specification (UX Contract)

This document specifies the v1 command-line interface for **gitvend**.

It is a UX contract for:
- human users (local dev)
- CI maintainers (GitHub Actions / GitLab CI)
- AI coding agents implementing the tool

It is aligned with:
- [doc/domain-model.md](doc/domain-model.md)
- [doc/requirements.md](doc/requirements.md)
- [doc/acceptance-criteria.md](doc/acceptance-criteria.md)

---

## 0) CLI design principles

1. **Deterministic by default**: outputs and lockfiles support reproducibility.
2. **Safe by default**: no writing outside Target Repo; atomic writes; clear errors.
3. **Fast on repeat**: mirror caching; avoid redundant fetches per invocation.
4. **CI-friendly**: predictable exit codes; machine-readable Run Report; non-interactive.
5. **Great DX**: convenience commands for manifest maintenance.

---

## 1) Global behavior

### 1.1 Executable name

- CLI command: `gitvend`

### 1.2 Global options

These options MAY apply to all commands:

- `--help` / `-h` — show help
- `--version` — print version
- `--log-level <error|warn|info|debug>` — default `info`
- `--json-logs` — emit logs in JSON (useful for CI log parsing)
- `--no-color` — disable ANSI colors

### 1.3 Environment variables

- `GITVEND_HOME` — override default `${HOME}/.gitvend`
- `GITVEND_LOG_LEVEL` — default log level (if `--log-level` not provided)

### 1.4 Home layout (informational)

- Mirrors: `${GITVEND_HOME:-${HOME}/.gitvend}/mirrors/<mirror>.git`
- Mirror Locks: `${GITVEND_HOME:-${HOME}/.gitvend}/mirrors/<mirror>.lock.json`

### 1.5 Repo-local layout (informational)

- Target Repo Lock: `<target-repo>/.gitvend/target-repo.lock.json`
- Run Report (default): `<target-repo>/.gitvend/gitvend.report.json`
  - If the file already exists, gitvend MUST move it aside before writing the new report:
    - `<target-repo>/.gitvend/gitvend.report-<iso-timestamp>.json`
- Vendor Lockfile: `<manifest-dir>/<manifest-base>.lock.<ext>`
  - Located next to the manifest file.
  - `<manifest-base>` is the manifest filename without extension (e.g., `gitvend` or `.gitvend`).
  - `<ext>` matches the manifest extension (`yml` vs `yaml`).

---

## 2) Manifest discovery

### 2.1 Default behavior

If multiple recognized manifests are present within the same directory, gitvend MUST fail with a manifest error (to avoid ambiguous configuration).

If `--manifest` is not provided:

1. Start at current working directory.
2. Traverse upward until reaching the Target Repo root (directory containing `.git`).
3. In each directory (starting from CWD), check for recognized manifest filenames in this precedence order:
   - `gitvend.yml`
   - `gitvend.yaml`
   - `.gitvend.yml`
   - `.gitvend.yaml`
4. Select the first manifest found by this algorithm.
5. If no manifest is found within the repo boundary, fail.

---

## 3) Command surface (v1)

### 3.1 `gitvend sync`

**Purpose:**
- Fetch/update required mirrors
- Resolve revisions
- Vendor files/directories into the Target Repo
- Write/update Vendor Lockfile (next to the manifest; see §1.5)
- Produce Run Report
- Auto-commit managed changes (default; see §5)

**Synopsis:**

- `gitvend sync [--manifest <path>] [--strict] [--lock-timeout <duration>] [--report <path>] [--ci] [--no-commit] [--commit] [--commit-message <template>] [--dirty-policy <fail|allow-unrelated>] [--dry-run] [--fail-on-fallback]`

**Options:**

- `--manifest <path>`
  - Path to manifest YAML.
  - May be repeated to sync multiple repos/manifests in one invocation.

- `--strict`
  - Enables stricter validation and stricter warning handling (see §6).

- `--lock-timeout <duration>`
  - Max time to wait for Mirror Locks and Target Repo Lock.
  - Default: `60s`.
  - Duration format: `10s`, `2m`, `1h`.

- `--report <path>`
  - Path to write Run Report JSON.
  - If omitted, use the default report path (see §1.5).

- `--ci`
  - CI mode: implies non-interactive behavior and stable output formatting.
  - Recommended defaults:
    - disables color
    - ensures JSON report is written
    - avoids prompts

- `--no-commit`
  - Disables auto-commit even if enabled by defaults/config.

- `--commit`
  - Forces auto-commit of managed changes when sync modifies outputs (even if defaults would disable it).

- `--commit-message <template>`
  - Template for commit message.
  - Template variables are specified in §5.

- `--dirty-policy <fail|allow-unrelated>`
  - Controls when auto-commit is allowed.
  - `fail`:
    - If any uncommitted changes exist (managed or unmanaged), refuse to auto-commit and exit non-zero.
  - `allow-unrelated`:
    - If there are uncommitted changes outside gitvend-managed paths, auto-commit may proceed (but must only stage/commit managed paths).
    - If there are uncommitted changes inside gitvend-managed paths, gitvend MUST fail (to prevent overwriting uncommitted work).
  - Default: `allow-unrelated`.

- `--dry-run`
  - Plan and resolve, but do not write outputs, lockfiles, or commits.
  - Still may fetch mirrors (unless `--offline` is added in the future).

- `--fail-on-fallback`
  - Treat fallback (same-branch missing → default branch) as an error.
  - Useful in strict CI scenarios.

**Behavioral guarantees:**
- Idempotent: repeated runs without source changes produce no output changes.
- Golden source: managed paths are overwritten to match resolved source, except when they have uncommitted modifications (in which case sync must fail to prevent work loss).

---

### 3.2 `gitvend check`

**Purpose:**
- Verify that the Target Repo’s vendored outputs match what the manifest + resolved revisions dictate.
- Does not modify workspace outputs.

**Synopsis:**

- `gitvend check [--manifest <path>] [--strict] [--lock-timeout <duration>] [--report <path>] [--ci] [--use-lockfile] [--fail-on-fallback]`

**Options:**

- `--manifest <path>`
  - Same as `sync`.

- `--strict`
  - Stricter validation and warning handling.

- `--lock-timeout <duration>`
  - Max time to wait for locks.

- `--report <path>`
  - Path to write Run Report JSON.

- `--ci`
  - CI semantics (non-interactive, stable logs).

- `--use-lockfile`
  - If present, prefer resolved SHAs from the Vendor Lockfile (next to the manifest; see §1.5) when available.
  - If lockfile is missing or incomplete, fall back to manifest ref resolution.

- `--fail-on-fallback`
  - Treat fallback decisions as errors.

**Behavioral guarantees:**
- Must not modify vendored outputs.
- MUST acquire the Target Repo Lock (same lock as `sync`) to ensure a consistent view of the workspace.
- May fetch mirrors to evaluate refs, consistent with `sync` behavior, but must not write outputs.

---

### 3.3 `gitvend mirror update`

**Purpose:**
- Pre-warm or update one or more mirrors.
- Useful for CI caches or local prefetch.

**Synopsis:**

- `gitvend mirror update [--source <sourceId>]... [--url <repoUrl>]... [--manifest <path>] [--lock-timeout <duration>] [--ci]`

**Options:**

- `--source <sourceId>`
  - Update mirror for a named Source Repo from the manifest.

- `--url <repoUrl>`
  - Update mirror for an explicit repo URL without a manifest.

- `--manifest <path>`
  - Load `sourceRepos` from manifest. If omitted, requires `--url`.

- `--lock-timeout <duration>`
  - Wait time for Mirror Lock.

- `--ci`
  - CI semantics.

**Behavioral guarantees:**
- Updates each selected mirror at most once per invocation.

---

### 3.4 `gitvend cache gc` (optional later)

**Purpose:**
- Garbage-collect local mirrors and metadata.

**Status:**
- Not required for v1.

**Planned synopsis:**

- `gitvend cache gc [--max-age <duration>] [--max-size <bytes>] [--dry-run]`

---

### 3.5 Manifest convenience commands (v1 DX)

These commands are recommended for v1 to support great developer experience.

#### 3.5.1 `gitvend manifest init`

**Purpose:** Create a new manifest with minimal boilerplate.

- `gitvend manifest init [--path <path>] [--format yaml]`

#### 3.5.2 `gitvend manifest add-source`

**Purpose:** Add a Source Repo entry to a manifest.

- `gitvend manifest add-source --manifest <path> --id <sourceId> --url <repoUrl> [--default-branch <name>]`

#### 3.5.3 `gitvend manifest add-entry`

**Purpose:** Add a Vendor Entry to a manifest.

- `gitvend manifest add-entry --manifest <path> --id <entryId> --source <sourceId> --type <file|dir> --from <path> [--to <path>] [--policy <same-branch-else-default|same-branch-else-fail|fixed-ref>] [--ref <ref>]`

**Notes:**
- For `fixed-ref`, `--ref` is required.
- Inputs must be validated; `--to` defaults to `--from` when omitted.

---

## 4) Exit codes and CI semantics

### 4.1 Exit codes (stable)

- `0` — success
- `1` — generic failure (unexpected)
- `2` — usage error (invalid arguments, help)
- `3` — manifest error (parse/validation)
- `4` — lock timeout / concurrency error
- `5` — git/remote/auth error
- `6` — ref resolution error
- `7` — extraction or filesystem write error
- `8` — drift detected (`check`)

### 4.2 CI semantics

When `--ci` is set:
- No interactive prompts.
- Disable color unless explicitly enabled.
- Ensure Run Report is produced (either `--report` or default).
- Prefer stable, concise log formatting.

---

## 5) Auto-commit UX contract

### 5.1 Enablement

Auto-commit is enabled by default for `sync`.

It may be enabled/forced by:
- `--commit`

It may be disabled by:
- `--no-commit`

### 5.2 Dirty workspace policy

- Default: `--dirty-policy allow-unrelated`.
- If `allow-unrelated`, gitvend MUST stage and commit only managed vendored paths.
- If any gitvend-managed path has uncommitted changes, gitvend MUST fail (to avoid overwriting local work).

### 5.3 Commit message template

Default commit message (conceptual):

- `chore(gitvend): sync vendored artifacts` + summary line(s)

Template variables (proposed):
- `${manifest}` — manifest path
- `${sourceRepos}` — list of `sourceId`s used
- `${revisions}` — list of `sourceId@sha`
- `${entriesChanged}` — count
- `${pathsChanged}` — list or truncated list

Exact formatting is finalized in `doc/output-artifacts.md`.

---

## 6) Strict mode (`--strict`)

Strict mode tightens behavior:

- Treat selected warnings as errors (e.g., LFS pointer-only, fallback used).
- Fail on manifest ambiguities.

Exact strict-mode matrix (which warning becomes error) is finalized in `doc/output-artifacts.md`.

---

## 7) Logging conventions

### 7.1 Log levels

- `error` — failures, non-recoverable events
- `warn` — recoverable issues, fallbacks, non-fatal anomalies
- `info` — high-level progress and decisions
- `debug` — detailed internals (git commands, timings, paths)

### 7.2 Required log events (minimum)

gitvend SHOULD log at `info`:
- manifest selected + target repo root
- number of source repos and vendor entries
- for each source: mirror id and whether fetched
- for each entry: resolved revision (ref + sha) and whether fallback was used
- summary: counts (synced/unchanged/failed), duration

At `warn`:
- fallback used (same-branch missing)
- stale lock recovered (if applicable)
- mirror healed by reinit

At `error`:
- stable error code + message
- context fields (entryId/sourceId/repoUrl)

### 7.3 JSON logging

When `--json-logs` is set, each log line SHOULD be a JSON object including:
- `ts` (timestamp)
- `level`
- `msg`
- `code` (if error)
- `context` (structured)

---


