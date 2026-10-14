# ADR 0005 — Release Please for versioning and changelog generation

**Status:** Accepted  
**Date:** July 2026

---

## Context

The monorepo contains multiple independently versioned packages. An automated release process is needed that generates changelogs, bumps versions, and publishes to npm without manual intervention.

The spirit of this choice goes beyond tooling convenience: the release process should feel frictionless enough that community contributors can land a fix or feature and see it ship without needing to understand internal release mechanics. Versioning and releasing should be a near-invisible byproduct of good commit hygiene, not a separate workflow step that gatekeeps contribution.

## Decision

Use [Release Please](https://github.com/googleapis/release-please) (`googleapis/release-please-action`) driven by Conventional Commits. Changesets was evaluated and rejected.

## Rationale

- **Community-first contribution model.** A contributor only needs to write a well-formed commit message. Release Please handles everything downstream — changelog entry, version bump, Release PR — with no additional action required from the contributor. This keeps the bar to contribution as low as possible.
- **Dead-simple versioning.** The version a change produces is determined by the commit type (`feat` → minor, `fix` → patch, `feat!` → major). There is no separate decision point, no file to author, no PR review step owned by the contributor. The rules are mechanical and documented once.
- **Conventional Commits are already enforced** via commitlint + Husky. Release Please consumes commit history directly — no separate changeset authoring step required per PR.
- **Automated PR workflow.** Release Please opens and maintains a Release PR on every push to `main`, accumulating changes until a maintainer merges. This gives a clear preview of the next release without blocking development.
- **Scope-based package targeting.** Commit scope (`theme`, `toolbar`, `codemods`, `site`) maps directly to which package Release Please bumps, keeping releases independent.
- **Changesets comparison.** Changesets requires contributors to author a changeset file per PR. For a project aiming for broad community contribution, this is redundant ceremony — it moves release decisions into the contribution flow rather than out of it.

## Consequences

- All commits to `main` must follow the Conventional Commits format (`type(scope): subject`).
- Commitlint enforces this at commit time via Husky.
- Release PRs are opened automatically; maintainers review and merge to trigger a release.
- Tags follow the pattern `theme-v<version>` and `toolbar-v<version>` for independent package versioning.
- npm publish is triggered by tag push, not by merging the Release PR directly.
