# ADR 0004 — pnpm workspaces monorepo

**Status:** Accepted  
**Date:** July 2026

---

## Context

The project encompasses multiple publishable packages (`@carbon/echarts-theme`, `@carbon/echarts-toolbar`, `@carbon/echarts-codemod`) and an internal showcase site. These need to share tooling, dev dependencies, and local source references during development.

## Decision

Use a pnpm workspaces monorepo. No additional monorepo orchestration layer (e.g. Nx, Turborepo) is used beyond pnpm's built-in workspace protocol.

## Rationale

- **pnpm's workspace protocol** (`workspace:*`) gives local cross-package references with zero config and no hoisting surprises.
- **Simplicity over orchestration.** The build graph is shallow — `theme` has no deps on other workspace packages; `site` and `toolbar` depend only on `theme`. A full Turborepo pipeline adds config complexity without meaningful benefit at this scale.
- **Single lockfile.** All packages share `pnpm-lock.yaml`, keeping dependency resolution consistent across CI and local development.
- **Filtering.** `pnpm --filter <package> <command>` provides targeted per-package operations without extra tooling.

## Consequences

- All workspace packages live under `packages/`.
- Root-level scripts (`pnpm build`, `pnpm test`, `pnpm lint`) run across all packages.
- If the dependency graph grows significantly more complex (parallel builds, remote caching needs), Turborepo can be layered on without changing the workspace structure.
