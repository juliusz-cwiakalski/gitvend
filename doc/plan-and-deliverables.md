# gitvend — Full Documentation Plan (Specs → Tests → Manual → Agent Handoff)

This plan translates our conversation into a complete, implementation-ready documentation package for **gitvend** (the GitHub repo name and CLI command). The objective is to produce a set of documents that remove ambiguity for both humans and AI coding agents, covering PRD/specs, algorithms, storage/locking, test plan/fixtures, and user/maintainer manuals.

---

## Scope Summary (baseline for all documents)

**gitvend** provides:
- Local **bare Git mirrors** cached under `${HOME}/.gitvend/...` to save bandwidth and accelerate sync.
- Safe **locking** for mirror updates and workspace sync operations.
- **Selective vendoring** of files/folders from remote git repositories into a target repository based on a manifest.
- **Branch-aware ref resolution**: prefer a source branch matching the current workspace branch (or a configured change branch); configurable fallbacks.
- CI-friendly **determinism** via a lockfile (**`gitvend.lock`**) and drift detection via `gitvend check`.
- Clear provenance for vendored artifacts; policy to prevent edits of vendored files in the consumer repo.

Non-goals:
- Symlink-based linking.
- Git submodule management.
- Multi-repo atomic commits (gitvend supports deterministic assembly; release orchestration remains separate).

---

## Step-by-step plan (documents in creation order)

### Step 0 — Repo bootstrap (minimal but anchors everything)

**0.1 `README.md` (skeleton)**
- What gitvend is and what it is not
- Quickstart (1–2 commands)
- Link to `docs/INDEX.md`
- “Core concepts” 3–5 bullets

**0.2 `docs/INDEX.md` (canonical table of contents)**
- Reading order for implementers
- Reading order for users
- Cross-links to PRD, requirements, manifest, algorithm, test plan

**0.3 `docs/glossary.md` (optional but recommended early)**
- Mirror, Source, Manifest, Entry, Target, Ref Policy, Lockfile, Report

---

### Step 1 — Product definition (PRD)

**1.1 `docs/prd.md`**
- Problem statement and target outcomes
- Primary personas (solo founder, platform engineer, CI maintainer)
- Key use cases (central docs repo, contract vendoring, platform assembly)
- Explicit non-goals (symlinks, submodules, continuous syncing)
- Success criteria and measurable outcomes
- Constraints (offline-ish via mirrors, reproducibility, security)

---

### Step 2 — Requirements (FR/NFR) and acceptance criteria mapping

**2.1 `docs/requirements.md`**
- Functional requirements (FR-001…)
  - Mirror store structure, bare repos
  - Mirror update locking
  - Manifest-driven selective file/folder sync
  - Ref resolution policies (same-branch, fixed ref, fallback)
  - Deterministic SHA pinning for CI (lockfile)
  - `sync` vs `check` behaviors
  - Provenance metadata insertion
  - Workspace write atomicity
- Non-functional requirements (NFR-001…)
  - Determinism, performance, portability assumptions
  - Security (no secret material in manifests)
  - Reliability (crash-safety, lock timeout)
  - Observability (logs, JSON report)

**2.2 `docs/acceptance-criteria.md`**
- AC entries mapped to FR/NFR IDs
- “Definition of Done” for v1

---

### Step 3 — Domain model and error taxonomy

**3.1 `docs/domain-model.md`**
- Entities and relationships
- State transitions
- Error taxonomy with stable codes (e.g., `GV_REF_NOT_FOUND`, `GV_AUTH_FAILED`)

---

### Step 4 — CLI contract (UX spec)

**4.1 `docs/cli-spec.md`**
- Command surface (v1)
  - `gitvend sync`
  - `gitvend check`
  - `gitvend mirror update` (optional but useful)
  - `gitvend cache gc` (optional later)
- Flags and env vars
  - `--manifest`, `--strict`, `--lock-timeout`, `--report`, `--ci`
  - `GITVEND_HOME` override (optional)
- Exit codes and CI semantics
- Logging conventions

---

### Step 5 — Manifest specification (the centerpiece)

**5.1 `docs/manifest-spec.md`**
- File format (YAML recommended)
- Schema: required/optional fields, defaults
- Entry types:
  - single file
  - folder (recursive)
  - multiple paths
- Ref policies:
  - same-branch-else-fail
  - same-branch-else-default
  - fixed ref/tag
- Validation rules:
  - path traversal prevention
  - duplicate target detection
  - conflict detection

**5.2 `docs/manifest-examples/`**
- `central-docs.yml`
- `api-contracts.yml`
- `platform-assembly.yml`
- Each with brief “what this demonstrates” notes

---

### Step 6 — Resolution and sync algorithm (determinism rules)

**6.1 `docs/sync-algorithm.md`**
- How workspace branch is detected
- How “branch exists in source” is checked
- Resolution precedence order:
  1) explicit override (flag/env)
  2) policy-driven same-branch
  3) fallback policy (to configured default branch per repo)
  4) default branch
- CI mode:
  - resolve all sources to SHAs
  - write/update lockfile
  - store report
- Extraction mechanisms:
  - file: `git show <sha>:<path>`
  - folder: `git archive --format=zip <sha> <path>` + unzip
- Atomic write strategy (temp → rename)

---

### Step 7 — Storage layout and locking

**7.1 `docs/storage-and-locking.md`**
- Mirror layout:
  - `${HOME}/.gitvend/mirrors/<id>.git` (bare)
  - `${HOME}/.gitvend/mirrors/<id>.meta.json`
  - `${HOME}/.gitvend/mirrors/<id>.lock` (per-mirror update lock)
- URL→ID scheme (hash + optional slug)
- Lock locations and formats
- Lock timeout and stale-lock recovery policy
- Lock metadata (acquired timestamp, PID when useful)
- Crash/retry behavior
- Cross-platform locking requirements (Windows/macOS/Linux)

---

### Step 8 — Output artifacts (lockfile + report + provenance)

**8.1 `docs/output-artifacts.md`**
- `gitvend.lock` format
  - repo, ref policy, resolved SHA, timestamp
- `gitvend.report.json` format
  - per-entry status, warnings, fallbacks, durations
- Provenance header conventions
- `check` mode rules for drift detection

---

### Step 9 — Test strategy, plan, and fixtures

**9.1 `docs/test-strategy.md`**
- Test levels (unit/integration/e2e)
- What must be mocked vs real git repos
- Coverage expectations and failure-mode testing

**9.2 `docs/test-plan.md`**
- Test cases mapped to FR/NFR/AC
- Concurrency and lock contention tests
- Crash-safety / partial write tests
- Security tests (path traversal, malicious manifest)

**9.3 `docs/test-fixtures-spec.md`**
- Definition of local fixture repos for tests
  - repo A with branch `main` and `feature-x`
  - repo B missing `feature-x`
  - tags
  - nested folders
  - binary file
- How to generate fixtures reproducibly

---

### Step 10 — User manual (how to use gitvend)

**10.1 `docs/user-manual.md`**
- Installation and prerequisites
- First sync, mirror warmup
- Typical workflows:
  - local dev: `gitvend sync`
  - change branch: same-branch policy usage
  - CI: `gitvend check` gates merges
- Troubleshooting and FAQ

---

### Step 11 — Maintainer/operator guide

**11.1 `docs/maintainer-guide.md`**
- Internal architecture overview
- Debugging (locks, auth, ref resolution)
- Release process
- Backward compatibility rules for manifest schema

---

### Step 12 — AI coding agent handoff package

**12.1 `docs/implementation-plan.md`**
- Recommended language/runtime (your choice)
- Module/package structure
- Key data structures and flows
- Stepwise implementation milestones

**12.2 `docs/agent-brief.md`**
- The exact instructions for an AI coding agent:
  - Must read: INDEX → PRD → requirements → manifest → algorithm → locking → tests
  - Hard constraints
  - Acceptance checklist

---

## Consolidated list of documents (final order)

1. `README.md`
2. `docs/INDEX.md`
3. `docs/glossary.md` (optional)
4. `docs/prd.md`
5. `docs/requirements.md`
6. `docs/acceptance-criteria.md`
7. `docs/domain-model.md`
8. `docs/cli-spec.md`
9. `docs/manifest-spec.md`
10. `docs/manifest-examples/central-docs.yml`
11. `docs/manifest-examples/api-contracts.yml`
12. `docs/manifest-examples/platform-assembly.yml`
13. `docs/sync-algorithm.md`
14. `docs/storage-and-locking.md`
15. `docs/output-artifacts.md`
16. `docs/test-strategy.md`
17. `docs/test-plan.md`
18. `docs/test-fixtures-spec.md`
19. `docs/user-manual.md`
20. `docs/maintainer-guide.md`
21. `docs/implementation-plan.md`
22. `docs/agent-brief.md`

---

## Optional extensions (defer until after v1)

- `docs/security.md` (threat model, token handling, CI permissions)
- `docs/performance.md` (benchmarks, caching strategy)
- `docs/design-decisions/ADR-0001.md` (why vendoring, why mirrors, why no symlinks)
- `docs/roadmap.md` (daemon mode, non-git sources, transforms)

