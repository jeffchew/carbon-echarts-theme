# ADR 0007 — Codemods co-located in this repository

**Status:** Accepted  
**Date:** July 2026

---

## Context

The Carbon monorepo (`carbon-design-system/carbon`) has its own codemods package for automating migration between Carbon versions. The question was whether `@carbon/echarts-codemod` should live there or in this repository.

## Decision

`@carbon/echarts-codemod` lives in `packages/codemods/` in this repository and is published as a separate npm package from here.

## Rationale

- **Release independence.** Chart type additions and preset API changes can ship codemods without requiring a Carbon core review cycle.
- **Ownership.** The team maintaining `@carbon/echarts-theme` is best placed to own and update the transforms that migrate to it. Placing codemods in the Carbon monorepo creates a cross-team dependency for every chart type change.
- **Different concern.** Carbon core codemods handle Carbon internal API changes (token renames, component API changes). These codemods handle migrating _to_ the ECharts theme from `@carbon/charts` — a distinct migration direction with no overlap.
- **Co-location.** Keeping the component→preset mapping (`CHART_MAP`) adjacent to the preset source means it stays in sync automatically as presets are added or renamed.

## Consequences

- `@carbon/echarts-codemod` is versioned and released via Release Please alongside the other packages in this repo.
- The `packages/codemods/` directory was scaffolded; implementation is pending (tracked in [`docs/plan/roadmap.md`](../plan/roadmap.md)).
- If the Carbon Design System team wishes to include these transforms in the Carbon monorepo in future, this ADR should be revisited.
