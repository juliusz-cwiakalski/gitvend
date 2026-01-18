# Manifest Examples

This folder contains example gitvend manifests based on:
- [doc/manifest-spec.md](../manifest-spec.md)
- [doc/domain-model.md](../domain-model.md)
- [doc/cli-spec.md](../cli-spec.md)

Each file starts with brief notes describing what it demonstrates.

## Examples

- `minimal.yml`
  - Smallest useful manifest: 1 source repo + 1 vendored file.
  - `toPath` omitted (defaults to `fromPath`).
  - `refPolicy` omitted (inherits `refPolicyDefault`).

- `central-docs.yml`
  - A single "central docs" Source Repo.
  - Mix of `dir` and `file` entries.
  - Branch-aware default policy (`same-branch-else-default`).

- `api-contracts.yml`
  - A contracts repo pinned to a specific tag via `fixed-ref`.
  - Demonstrates a `paths` bundle to keep a set of related artifacts on the same resolved revision.

- `platform-assembly.yml`
  - Multiple Source Repos assembled into one consumer repo.
  - Mix of policies and an entry-level `defaultBranchOverride`.

- `ci-strict.yml`
  - CI-oriented defaults (`settings.strict: true`).
  - Demonstrates pairing manifest defaults with CLI flags like `--ci` and `--fail-on-fallback`.

## How to use

- Run sync:

  ```sh
  gitvend sync --manifest doc/manifest-examples/central-docs.yml
  ```

- Check drift (CI gate):

  ```sh
  gitvend check --manifest doc/manifest-examples/central-docs.yml --ci
  ```

- Strict CI (fail on fallback decisions):

  ```sh
  gitvend check --manifest doc/manifest-examples/ci-strict.yml --ci --fail-on-fallback
  ```
