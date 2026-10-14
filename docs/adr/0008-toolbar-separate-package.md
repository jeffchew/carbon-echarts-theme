# ADR 0008 — Toolbar as a separate published package

**Status:** Accepted  
**Date:** July 2026

---

## Context

ECharts charts benefit from a toolbar offering CSV export, image export, fullscreen, and a "show as table" accessibility view — mirroring the toolbar experience in `@carbon/charts`. The question was whether this functionality should be bundled into `@carbon/echarts-theme` or shipped as a separate package.

## Decision

Ship the toolbar as a standalone package: `@carbon/echarts-toolbar` in `packages/toolbar/`, published independently to npm.

## Rationale

- **Not all consumers need a toolbar.** Teams using `@carbon/echarts-theme` for embedded charts, dashboards, or read-only contexts would take on dead code if the toolbar were bundled. A separate package means zero cost for non-users.
- **Different dependency profile.** The toolbar's vanilla DOM layer and SCSS styles are unrelated to the theme object. Bundling them into `@carbon/echarts-theme` would couple a CSS artifact and DOM API to a package that is currently a zero-dependency plain JS object.
- **Independent release cadence.** Toolbar bug fixes and new export features can ship without bumping the theme package version, avoiding unnecessary churn for consumers who only care about the theme.
- **Framework adapter path.** The toolbar is designed with a zero-dependency core (`@carbon/echarts-toolbar`) and a thin framework adapter layer on top. This architecture only makes sense as a separate package with its own subpath exports (`/vanilla`, `/styles`).
- **Precedent.** `@carbon/charts` separates concerns similarly — the core chart library is distinct from supplementary packages.

## Consequences

- `@carbon/echarts-toolbar` is versioned and released independently via Release Please (`toolbar-v*` tags).
- The toolbar has `echarts` as a peer dependency, matching the theme package.
- Framework adapter wrappers (React, Angular, Vue) are deferred — the vanilla DOM API (`createChartToolbar`, `autoToolbar`) is the v1 surface.
- Consumers who want the toolbar install it separately: `npm install @carbon/echarts-toolbar`.
- The showcase site (`packages/site`) uses the toolbar directly and serves as its integration test harness until dedicated tests are written (tracked in [`plan/roadmap.md`](../plan/roadmap.md) W1).
