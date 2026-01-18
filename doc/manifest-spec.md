# gitvend — Manifest Specification (v1)

This document defines the canonical **manifest file format** for gitvend.

The manifest is the centerpiece of gitvend: it declares **Source Repos** and **Vendor Entries** that copy content from a **Resolved Revision** to a **Target Path** inside a **Target Repo**.

---

## 0) Scope and non-goals

### In scope (v1)
- YAML manifest (recommended) with a stable schema and strict validation.
- Source Repo definitions (URL + defaults).
- Vendor Entry definitions for:
  - single file
  - folder (recursive)
  - multiple paths (batch / bundle)
- Ref policies:
  - `same-branch-else-fail`
  - `same-branch-else-default`
  - `fixed-ref`
- Safety validation:
  - path traversal prevention
  - duplicate target detection
  - conflict/overlap detection

### Not in scope (v1)
- Symlinks or filesystem linking.
- Git submodules.
- Continuous syncing/daemon mode.
- Transforms/templating (e.g., rewriting content while vendoring).
- Glob patterns for selecting files (may be added later).

---

## 1) File format

### 1.1 Supported formats

- **YAML** is the canonical format for v1.
- The schema is intentionally DTO-friendly so additional encodings (e.g., JSON) can be supported later.

### 1.2 Encoding and conventions

- UTF-8
- LF line endings
- Use spaces, not tabs
- Paths use forward slashes (`/`) regardless of OS

### 1.3 Default manifest filenames (discovery)

When `--manifest` is not provided, gitvend discovers a manifest by searching upward within the same Target Repo.

Recognized filenames (highest precedence first):
1. `gitvend.yml`
2. `gitvend.yaml`
3. `.gitvend.yml`
4. `.gitvend.yaml`

If more than one recognized filename exists in the same directory, gitvend fails (manifest ambiguity).

---

## 2) Top-level schema

Minimal skeleton:

```yaml
version: 1
sourceRepos: []
vendorEntries: []
```

### 2.1 Fields

| Field | Type | Required | Default | Notes |
|---|---:|:---:|---|---|
| `version` | int | yes | — | Must be `1` in v1. |
| `settings` | object | no | `{}` | Optional defaults (see §2.2). |
| `sourceRepos` | list[SourceRepo] | yes | `[]` | Declares Source Repos. |
| `vendorEntries` | list[VendorEntry] | yes | `[]` | Declares vendoring rules. |

### 2.2 `settings` (optional)

`settings` provides defaults used when a Vendor Entry does not override them.

Allowed keys (v1):

| Field | Type | Default | Notes |
|---|---:|---|---|
| `autoCommit` | bool | `true` | Auto-commit is enabled by default for `gitvend sync`. |
| `dirtyPolicy` | enum | `allow-unrelated` | Unmanaged changes allowed; managed-path dirtiness fails (see below). |
| `lockTimeout` | duration | `60s` | Used when CLI does not override. |
| `reportPath` | string | `<target-repo>/.gitvend/gitvend.report.json` | Default Run Report path (CLI may override). Existing report is rotated before writing a new one. |
| `strict` | bool | `false` | If true, behaves like `--strict` by default. |

Notes:
- CLI flags MUST override `settings`.
- `dirtyPolicy: allow-unrelated` means:
  - uncommitted changes in unmanaged paths are allowed
  - uncommitted changes in managed (vendored) paths are **not allowed** and MUST fail

---

## 3) Source Repo schema

A Source Repo declares a remote Git repository and defaults.

### 3.1 Source Repo fields

```yaml
sourceRepos:
  - sourceId: docs
    repoUrl: https://github.com/acme/platform-docs.git
    defaultBranch: main
    refPolicyDefault: same-branch-else-default
    authHint: ssh
```

| Field | Type | Required | Default | Notes |
|---|---:|:---:|---|---|
| `sourceId` | string | yes | — | Unique within manifest. Used by Vendor Entries. |
| `repoUrl` | string | yes | — | Git remote URL (HTTPS or SSH). |
| `defaultBranch` | string | no | `main` | Default branch in the source repo used by `same-branch-else-default` fallback. |
| `refPolicyDefault` | enum | no | `same-branch-else-default` | Default ref policy for Vendor Entries referencing this Source Repo. |
| `authHint` | string | no | — | Informational only; MUST NOT include secrets. |

### 3.2 Source Repo validation rules

- `sourceId` MUST be unique across `sourceRepos`.
- `repoUrl` MUST be non-empty and treated as an opaque Git URL string (gitvend normalizes it for mirror-id derivation).
- `defaultBranch` MUST be a simple branch name (no whitespace; no `..`; no NUL).

---

## 4) Vendor Entry schema

A Vendor Entry declares what to vendor and where to write it.

### 4.1 Vendor Entry fields (common)

```yaml
vendorEntries:
  - entryId: north-star
    sourceId: docs
    type: file
    fromPath: doc/north-star.md
    toPath: doc/north-star.md
    # optional: empty/unset inherits sourceRepo.refPolicyDefault
    refPolicy: same-branch-else-default
    # optional: empty/unset inherits sourceRepo.defaultBranch
    defaultBranchOverride: release/2026-01
```

| Field | Type | Required | Default | Notes |
|---|---:|:---:|---|---|
| `entryId` | string | yes | — | Unique within manifest. |
| `sourceId` | string | yes | — | References `sourceRepos[].sourceId`. |
| `type` | enum | yes | — | `file` \| `dir` \| `paths`. |
| `refPolicy` | enum | no | empty (inherit) | If empty/unset: use `sourceRepo.refPolicyDefault`. If that is also unset: default to `same-branch-else-default`. |
| `defaultBranchOverride` | string | no | empty (inherit) | Optional entry-level fallback override used only by `same-branch-else-default`: effective order is `vendorEntry.defaultBranchOverride` → `sourceRepo.defaultBranch` → `main`. |

Additional fields depend on `type`.

### 4.2 Entry type: `file` (single file)

```yaml
- entryId: api-contract
  sourceId: contracts
  type: file
  fromPath: api/openapi.yaml
  toPath: doc/contracts/openapi.yaml
  refPolicy: fixed-ref
  ref: v1.2.0
```

Additional fields:

| Field | Type | Required | Default | Notes |
|---|---:|:---:|---|---|
| `fromPath` | string | yes | — | Path inside Source Repo at the Resolved Revision. |
| `toPath` | string | no | same as `fromPath` | Target Path inside Target Repo. |
| `ref` | string | required for `fixed-ref` | — | Tag/branch/commit-ish for `fixed-ref`. |

### 4.3 Entry type: `dir` (folder, recursive)

```yaml
- entryId: docs-folder
  sourceId: docs
  type: dir
  fromPath: doc/
  toPath: doc/vendor/docs/
  refPolicy: same-branch-else-default
```

Additional fields:

| Field | Type | Required | Default | Notes |
|---|---:|:---:|---|---|
| `fromPath` | string | yes | — | Directory path in Source Repo. Trailing slash is required (v1). |
| `toPath` | string | no | same as `fromPath` | Target directory in Target Repo. |
| `ref` | string | required for `fixed-ref` | — | Tag/branch/commit-ish for `fixed-ref`. |

Semantics:
- The directory is copied recursively.
- The output directory contents become the golden source (gitvend-managed).

### 4.4 Entry type: `paths` (multiple paths / bundle)

Use this type when you want to declare multiple file/dir items under one entry ID.

Important: `paths` is **syntactic sugar**. gitvend processes each path item as a separate vendoring operation (each with its own Target Path), but they share the same Source Repo and ref resolution.

```yaml
- entryId: platform-docs-bundle
  sourceId: docs
  type: paths
  refPolicy: same-branch-else-default
  paths:
    - fromPath: doc/north-star.md
      toPath: doc/north-star.md
    - fromPath: doc/adr/
      toPath: doc/adr/
    - fromPath: api/openapi.yaml
      toPath: doc/contracts/openapi.yaml
```

Additional fields:

| Field | Type | Required | Default | Notes |
|---|---:|:---:|---|---|
| `paths` | list[PathItem] | yes | — | Each item is a file or directory; directory items MUST use trailing slash in `fromPath` (and SHOULD in `toPath`). |
| `ref` | string | required for `fixed-ref` | — | Applied to all path items. |

`PathItem` fields:

| Field | Type | Required | Default | Notes |
|---|---:|:---:|---|---|
| `fromPath` | string | yes | — | Path in Source Repo. Trailing slash indicates directory. |
| `toPath` | string | no | same as `fromPath` | Target Path in Target Repo. |

Notes:
- `paths` supports mixing files and directories.
- Each `PathItem.to` is a Target Path and MUST pass all path safety validation.

---

## 5) Path semantics

### 5.1 `fromPath` (Source path)

- `fromPath` is interpreted as a path **relative to the Source Repo root**.
- Must not be absolute.
- Must not contain `..` segments.
- Must not contain backslashes (`\\`).
- Must not contain NUL bytes.

Directory rule (v1):
- directories MUST be expressed with a trailing slash (e.g., `doc/adr/`).

### 5.2 `toPath` (Target Path)

- `toPath` is interpreted as a path **relative to the Target Repo root**.
- Must not be absolute.
- Must not escape the Target Repo root.
- Must not contain `..` segments.
- Must not contain backslashes (`\\`).
- Must not contain NUL bytes.

Directory rule (v1):
- for directory items, `to` SHOULD end with a trailing slash.

### 5.3 Normalization

For validation and conflict detection, gitvend MUST normalize paths by:
- collapsing redundant separators
- removing `./` segments
- rejecting any `..` segment

---

## 6) Ref policies and Resolved Revision

Each Vendor Entry (or `paths` bundle) is resolved to a **Resolved Revision** (effective ref + full commit SHA).

### 6.1 `same-branch-else-fail`

Algorithm:
1. Detect current branch name in the Target Repo.
2. Check if the same branch exists in the Source Repo.
3. If yes: resolve to that branch.
4. If no: fail the Vendor Entry (and the run).

Constraints:
- If Target Repo is in detached HEAD, behavior is the same as “branch not available” and MUST fail.

### 6.2 `same-branch-else-default`

Algorithm:
1. Detect current branch name in the Target Repo.
2. If branch exists in Source Repo: use it.
3. Otherwise fall back to the effective default branch.

Effective default branch resolution order:
1. `vendorEntry.defaultBranchOverride` (if present)
2. `sourceRepo.defaultBranch` (if present)
3. `main`

### 6.3 `fixed-ref`

- Uses `entry.ref` as the ref/tag/commit-ish.
- gitvend resolves it to a commit SHA (Resolved Revision).

Accepted `ref` examples:
- tag: `v1.2.3`
- branch: `release/2026-01`
- SHA (full or short): `a1b2c3d...`

Validation:
- `ref` MUST be present and non-empty.

---

## 7) Validation rules (v1)

### 7.1 Structural validation

- `version` must be `1`.
- `sourceRepos` and `vendorEntries` must be present (may be empty).
- `sourceRepos[].sourceId` unique.
- `vendorEntries[].entryId` unique.
- Every `vendorEntries[].sourceId` must reference an existing source ID.
- `type` must be one of `file`, `dir`, `paths`.

### 7.2 Path safety

For every Target Path:
- Reject absolute paths (POSIX `/x` and Windows `C:\\x`).
- Reject any path containing `..`.
- Reject any path containing NUL.
- Reject any path containing backslashes.

For every Source path:
- Reject absolute paths.
- Reject `..`.
- Reject NUL.
- Reject backslashes.

### 7.3 Duplicate target detection

gitvend MUST fail if any two outputs would write to the same normalized Target Path.

This applies across:
- multiple Vendor Entries
- multiple PathItems inside a `paths` Vendor Entry
- and across all combined manifests in a multi-manifest invocation

### 7.4 Conflict / overlap detection

gitvend MUST fail if any two outputs overlap such that one Target Path is a strict prefix of another, e.g.:

- Entry A targets `doc/vendor/` and Entry B targets `doc/vendor/readme.md`

Rationale:
- Overlap implies non-determinism and/or hidden overwrites depending on execution order.

### 7.5 Policy validation

- `fixed-ref` requires `ref`.
- `same-branch-else-*` must not specify `ref`.
- `defaultBranchOverride` is only meaningful for `same-branch-else-default`; if set for other policies, gitvend SHOULD warn (or fail in strict).

---

## 8) Vendor Lockfile coupling

Vendor Lockfile behavior is part of the gitvend contract.

- **Location:** next to the manifest file.
- **Name:** derived from the manifest filename (base name + `.lock.` + same extension):
  - Manifest `gitvend.yml` → lockfile `gitvend.lock.yml`
  - Manifest `gitvend.yaml` → lockfile `gitvend.lock.yaml`
  - Manifest `.gitvend.yml` → lockfile `.gitvend.lock.yml`
  - Manifest `.gitvend.yaml` → lockfile `.gitvend.lock.yaml`

The Vendor Lockfile records the Resolved Revisions (commit SHAs) used during `sync` to support deterministic CI and improved diagnostics.

---

## 9) Unknown keys and forward-compatibility

- Unknown top-level keys SHOULD fail in `--strict` and SHOULD warn otherwise.
- Unknown Vendor Entry keys SHOULD fail in `--strict` and SHOULD warn otherwise.
- `version` bump is required for schema-breaking changes.

---

## 10) Examples (pointer)

See `doc/manifest-examples/` for complete working examples.

