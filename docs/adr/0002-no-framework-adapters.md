# ADR 0002 — No framework adapters shipped with the theme package

**Status:** Accepted  
**Date:** July 2026

---

## Context

When designing `@carbon/echarts-theme`, one option was to ship thin React / Angular / Vue wrapper components that pre-apply the theme automatically. This is the model used by `@carbon/charts`, which ships separate `@carbon/charts-react`, `@carbon/charts-angular`, etc. packages.

## Decision

Ship no framework adapters. The package output is a plain JavaScript object. Users register it with their existing ECharts adapter.

## Rationale

- **ECharts already solves this.** Mature, well-maintained adapters exist for every major framework (`echarts-for-react`, `ngx-echarts`, `vue-echarts`). Wrapping them adds surface area with no user benefit.
- **One package, every framework.** A single `@carbon/echarts-theme` works everywhere without a separate release train per framework.
- **Release independence.** Framework adapter updates (e.g. React 19 compatibility) would otherwise block theme releases. Decoupling means a token update ships immediately.
- **SSR safety.** The `registerCarbonThemes(echarts)` function accepts the user's `echarts` instance as an argument — no `window` access at module load time. Raw theme objects are also exported for SSR adapters that take a `theme` prop directly.
- **Reduced maintenance burden.** No adapter code to test, type-check, or keep in sync with upstream adapter APIs.

## Consequences

- Consumers must install and configure their own ECharts framework adapter.
- The getting-started documentation must include per-framework wiring examples (React, Angular, Vue, vanilla).
- The package has zero runtime dependencies on any framework.
