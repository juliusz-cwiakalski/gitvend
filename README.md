# gitvend

Vendoring selected files and folders from remote Git repositories—**deterministically**, with **local bare mirrors**, **locking**, and **CI drift checks**.

`gitvend` is a small infrastructure tool for multi-repo environments where some artifacts (API contracts, shared docs, schemas, reference implementations) must be consumed across repositories without symlinks, submodules, or manual copy/paste.

---

## Why gitvend

When a change spans multiple repositories, shared artifacts tend to drift:
- API contracts in one repo silently diverge from consumers.
- Central documentation becomes outdated.
- CI pipelines waste time and bandwidth repeatedly cloning the same dependencies.

`gitvend` addresses this by:
- Maintaining local **bare mirrors** of upstream repositories.
- Using a manifest to **selectively vendor** specific files/folders into your repo.
- Resolving refs in a **branch-aware** way (prefer same branch name as the current target repo; configurable fallback), while making fallback behavior explicit.
- Producing **lock/report** artifacts so CI can enforce **determinism** and prevent drift.

---

## Key features

- **Local bare mirror cache** under `${HOME}/.gitvend/mirrors/…` (configurable).
- **Repository update locking** to prevent concurrent mirror corruption.
- **Manifest-driven selective sync** of files and folders (recursive) from Git sources.
- **Branch-aware ref resolution** (prefer a branch matching your current workspace branch; configurable fallbacks).
- **Portable directory vendoring** via `git archive --format=zip` + unzip (no working tree required; avoids OS-specific `tar` quirks).
- **Deterministic CI** via a lockfile with resolved commit SHAs.
- **Drift detection** via `gitvend check`.
- **Provenance** for vendored outputs (source repo/ref/SHA/path).

---

## Non-goals

`gitvend` intentionally does **not**:
- Create or manage symlinks.
- Replace Git submodules.
- Provide atomic multi-repo commits/releases.
- Encourage editing vendored files in the consumer repo.

---

## Concepts (quick)

- **Mirror**: a local bare clone of a remote repository, updated via `git fetch`.
- **Manifest**: a config file declaring what to vendor (sources → targets).
- **Entry**: one vendoring rule (source repo + path + ref policy → target path).
- **Lockfile**: a file recording resolved SHAs for deterministic runs.
- **Report**: machine-readable summary of what was synced, including fallbacks/warnings.

---

## Installation

Choose one of the following approaches (depending on how you ship gitvend):

### Option A — Download a release (JAR + wrappers)
- Download the versioned `gitvend-<version>.jar` and the wrapper scripts (`gitvend` for macOS/Linux, `gitvend.cmd` for Windows).
- Ensure Java is available (`java` on `PATH` or `JAVA_HOME` set). Java 21+ is recommended.
- Put the wrapper on your `PATH`.

### Option B — Build from source
- Clone this repository.
- Build using the toolchain specified in `docs/implementation-plan.md`.

---

## Quickstart

1) Create a manifest file (example: `gitvend.yml`).

2) Sync vendored content:

```bash
gitvend sync --manifest gitvend.yml
```

3) Verify everything is up-to-date (CI-friendly):

```bash
gitvend check --manifest gitvend.yml
```

---

## Manifest example

The manifest is the single source of truth for what gets vendored.

```yaml
version: 1

settings:
  # Optional: override cache location
  # home: "${HOME}/.gitvend"

sources:
  - name: contracts
    repo: "git@github.com:acme/contracts.git"

entries:
  # Vendor a single file
  - id: openapi
    source: contracts
    type: file
    from: "api/openapi.yaml"
    to: "vendor/contracts/openapi.yaml"
    ref:
      policy: same-branch-else-main
      main_branch: main

  # Vendor a directory (recursive)
  - id: json-schemas
    source: contracts
    type: dir
    from: "schemas/json"
    to: "vendor/contracts/schemas/json"
    ref:
      policy: same-branch-else-fail
```

Notes:
- By default, gitvend attempts to resolve sources using the **current branch name** of the target repo. If the same branch exists in the source repo, gitvend uses it.
- Fallback behavior is controlled by `ref.policy` (e.g., fallback to `main`, or fail if the branch is required).
- `policy: same-branch-else-fail` is recommended when the change *must* include a matching branch in the source.
- `policy: same-branch-else-main` is acceptable for optional dependencies but should still be visible in reports.

The exact schema is defined in `docs/manifest-spec.md`.

---

## Mirror cache

By default, gitvend stores bare mirrors under:

- `${HOME}/.gitvend/mirrors/<id>.git`

Where `<id>` is derived from the repository URL (typically via a stable hash). Additional metadata may be stored alongside the mirror.

You can override the base directory via:
- `--home <path>` (if supported), or
- environment variable: `GITVEND_HOME` (recommended for CI).

See `docs/storage-and-locking.md`.

---

## Locking and concurrency

gitvend uses locking to ensure safety:

- **Per-mirror lock** during mirror updates (`fetch`) to prevent corruption.
- **Workspace lock** during sync to prevent concurrent writes to the same target paths.

Lock timeout and stale-lock recovery behavior are specified in `docs/storage-and-locking.md`.

---

## Commands

Common commands (v1 target):

- `gitvend sync` — update mirrors as needed, resolve refs, vendor files/folders.
- `gitvend check` — verify vendored outputs match sources (fails on drift).

Optional/extended commands (may be added later):

- `gitvend mirror update` — update mirrors without syncing files.
- `gitvend cache gc` — cleanup old mirrors / unused objects.

Full CLI contract: `docs/cli-spec.md`.

---

## Outputs: lockfile, report, and provenance

A typical sync run produces:

- **Lockfile** (example name): `gitvend.lock` or `sync.lock`
  - Records resolved commit SHAs per source/ref.
  - Enables deterministic CI re-runs.

- **Report** (example name): `gitvend.report.json`
  - Per-entry statuses, warnings, fallbacks used, timings.

- **Provenance markers** in vendored files (optional but recommended)
  - e.g., `synced from <repo>@<sha>:<path>`.

Exact formats: `docs/output-artifacts.md`.

---

## CI usage pattern

Recommended pattern:

1) In developer workflow:
```bash
gitvend sync --manifest gitvend.yml
```

2) In CI:
```bash
gitvend check --manifest gitvend.yml
```

If `check` fails, the PR must run `sync` and commit the updated vendored content (and lockfile, if used).

---

## Security notes

- Do **not** store credentials in manifests.
- Prefer SSH with agent locally.
- In CI, use read-only tokens with least privilege.

(Threat model and token handling can be expanded in `docs/security.md` if/when added.)

---

## Troubleshooting

### `check` fails with drift
- Run `gitvend sync --manifest …` and commit the changed vendored files.
- Inspect `gitvend.report.json` to see which source/ref changed.

### Branch not found
- If policy is `same-branch-else-fail`, create the same-named branch in the source repo.
- If fallback is allowed, confirm it is acceptable and recorded.

### Lock contention
- Another gitvend process may be running.
- If configured, gitvend can detect stale locks after a timeout.

---

## Documentation

All project documentation lives in `docs/`:

- `docs/INDEX.md` — reading order and table of contents
- `docs/prd.md` — product definition
- `docs/requirements.md` — FR/NFR list
- `docs/manifest-spec.md` — config schema
- `docs/sync-algorithm.md` — resolution + vendoring algorithm
- `docs/storage-and-locking.md` — mirror layout + locking
- `docs/test-plan.md` — test cases
- `docs/user-manual.md` — usage guide

---

## Contributing

- Propose changes via PR.
- Keep behavior deterministic and compatible with the manifest schema.
- Add tests for any behavior change (see `docs/test-plan.md`).

---

## License

MIT, see [LICENSE](./LICENSE).

