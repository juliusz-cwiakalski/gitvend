# gitvend Documentation Index

This index is the canonical entry point for gitvend documentation. It defines the recommended reading order and links to all major documents.

---

## Reading order for implementers (AI coding agents and maintainers)

Implementers should read documents in the order below to minimize ambiguity and ensure consistent behavior.

1. **Product definition** — `docs/prd.md`
   - Problem statement, scope, personas, success criteria.

2. **Requirements** — `docs/requirements.md`
   - Functional requirements (FR) and non-functional requirements (NFR).

3. **Acceptance criteria** — `docs/acceptance-criteria.md`
   - “Definition of Done” and mapping to FR/NFR.

4. **Domain model** — `docs/domain-model.md`
   - Terminology, entities, state transitions, error taxonomy.

5. **CLI contract** — `docs/cli-spec.md`
   - Commands, flags, exit codes, output locations, CI semantics.

6. **Manifest specification** — `docs/manifest-spec.md`
   - Config schema, validation rules, examples, defaults.

7. **Sync & resolution algorithm** — `docs/sync-algorithm.md`
   - Branch-aware ref resolution (same-branch-first), fallback policies, SHA pinning.

8. **Storage & locking** — `docs/storage-and-locking.md`
   - Mirror cache layout, lock strategy, timeouts, crash safety.

9. **Output artifacts** — `docs/output-artifacts.md`
   - Lockfile format, JSON report format, provenance markers.

10. **Test strategy** — `docs/test-strategy.md`
    - Test pyramid and what is mocked vs real Git repos.

11. **Test plan** — `docs/test-plan.md`
    - Test cases mapped to FR/NFR/AC, concurrency and security tests.

12. **Test fixtures** — `docs/test-fixtures-spec.md`
    - How to build reproducible local Git repos for integration tests.

13. **Implementation plan** — `docs/implementation-plan.md`
    - Module structure, sequencing, milestones.

14. **Agent brief** — `docs/agent-brief.md`
    - High-precision instructions for coding agents.

---

## Reading order for users

Users should read in this order:

1. **README** — `README.md`
   - High-level purpose, quickstart, manifest example.

2. **User manual** — `docs/user-manual.md`
   - Installation (JAR + wrappers), everyday workflows, troubleshooting.

3. **Manifest specification** — `docs/manifest-spec.md`
   - When creating or modifying manifests.

4. **CLI contract** — `docs/cli-spec.md`
   - Detailed command usage, flags, exit codes.

5. **Troubleshooting (reference)**
   - `docs/user-manual.md` (Troubleshooting section)
   - `docs/output-artifacts.md` (report/lock inspection)

---

## Cross-links to core specs

- Product requirements document (PRD): `docs/prd.md`
- Requirements (FR/NFR): `docs/requirements.md`
- Manifest schema: `docs/manifest-spec.md`
- Sync / ref resolution algorithm: `docs/sync-algorithm.md`
- Test plan: `docs/test-plan.md`

---

## Optional / auxiliary documents

- Glossary: `docs/glossary.md`
- Maintainer guide: `docs/maintainer-guide.md`
- Security (if added): `docs/security.md`
- Performance (if added): `docs/performance.md`
- ADRs (if added): `docs/design-decisions/`

