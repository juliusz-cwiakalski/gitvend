# gitvend Glossary

This glossary defines the core terms used throughout gitvend documentation and specifications.

---

## Mirror

A **Mirror** is a local, **bare** Git repository that acts as a cache for a remote repository.

- Stored under `${HOME}/.gitvend/mirrors/<id>.git` by default.
- Updated via `git fetch` (typically `--prune` and optionally `--tags`).
- Mirrors are protected by a **per-mirror lock** during updates to prevent corruption.

Purpose:
- Reduce bandwidth usage and speed up repeated operations.
- Provide a local object database for deterministic extraction (`git show`, `git archive`).

---

## Source

A **Source** is a logical reference to a Git repository declared in a manifest.

Typically includes:
- `name` (stable identifier within the manifest)
- `repo` (remote URL)
- optional defaults (e.g., default/main branch name)

Purpose:
- Avoid repeating repository URL and defaults across multiple entries.

---

## Manifest

A **Manifest** is a configuration file that declares what gitvend should vendor into the current repository.

A manifest typically defines:
- Sources (repositories)
- Entries (vendoring rules)
- Settings (cache location overrides, behavior defaults)

Purpose:
- Serve as the single source of truth for vendored artifacts.
- Enable deterministic and reviewable assembly of shared artifacts.

---

## Entry

An **Entry** is a single vendoring rule in a manifest.

An entry typically includes:
- Source (which repository)
- Type (`file` or `dir`)
- Source path (`from`)
- Target path (`to`)
- Ref Policy (how to resolve what commit to extract from)

Purpose:
- Define one unit of content to be copied (vendored) into the target repository.

---

## Target

A **Target** is the output location (path) inside the repository where the vendored content is written.

- For `file` entries: a single file path.
- For `dir` entries: a directory path (content is extracted recursively).

Purpose:
- Provide a stable, version-controlled destination for vendored artifacts.

---

## Ref Policy

A **Ref Policy** defines how gitvend selects the source revision (commit) to vendor from.

Common policies:
- **same-branch-else-fail**: Use a source branch with the same name as the current target repo branch; fail if not found.
- **same-branch-else-main**: Use same-name branch if it exists; otherwise fallback to a configured main/default branch.
- **fixed-ref**: Use an explicitly configured branch, tag, or commit.

Key attributes:
- Policy name
- Main/default branch name (if fallback is allowed)
- Optional overrides (e.g., explicit branch name)

Purpose:
- Support multi-repo feature development where the same feature branch exists across repos.
- Ensure deterministic behavior in CI by resolving to a commit SHA and recording it.

---

## Lockfile

A **Lockfile** is an output artifact that records the exact resolved commit SHAs used during a sync.

The lockfile typically includes (per source/ref):
- Repository URL
- Resolved ref (branch/tag)
- Resolved commit SHA
- Timestamp (optional)

Purpose:
- Ensure deterministic CI behavior (repeatable runs).
- Make provenance and debugging easier (“what did we sync from exactly?”).

---

## Report

A **Report** is a machine-readable summary of a gitvend run, typically produced as JSON.

The report typically includes:
- Per-entry status (synced/unchanged/failed)
- Ref resolution outcomes (which ref used, whether fallback occurred)
- Warnings and errors
- Timings (optional)

Purpose:
- Provide observability for CI and local runs.
- Make fallback usage explicit.
- Support troubleshooting and auditing.

