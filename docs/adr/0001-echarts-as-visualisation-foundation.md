# ADR 0001 — Apache ECharts as the IBM / Carbon visualisation foundation

**Status:** Accepted
**Date:** July 2026 — amended October 2026 (ECharts 6 minimum)

---

## Context

IBM Software spans React, Angular, Vue, and vanilla JS codebases. A standard charting library needs to work across all of them without a framework-specific fork. The chosen solution must also satisfy IBM open-source governance requirements and perform well on real enterprise data volumes.

## Decision

Adopt Apache ECharts as the charting engine underpinning the Carbon design system's data visualisation standard. `@carbon/echarts-theme` provides a Carbon-aligned theme layer on top of it.

## Rationale

| Factor                      | Detail                                                                                                                          |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **Framework agnosticism**   | Works with React, Angular, Vue, Svelte, and vanilla JS via thin adapters — one package serves the entire IBM Software portfolio |
| **Apache governance**       | Top-level Apache project with a formal release process and long-term support expectations; aligns with IBM open-source posture  |
| **Performance**             | Canvas renderer handles 100k+ point datasets; SVG fallback available                                                            |
| **Chart coverage**          | 20+ built-in chart types covering all Carbon Charts parity targets plus extended types                                          |
| **TypeScript**              | Full type definitions shipped                                                                                                   |
| **Carbon Charts alignment** | `@carbon/charts` internally migrated to ECharts; this project builds on that foundation rather than diverging from it           |

Framework agnosticism is the decisive factor: a design-system-level standard must serve every IBM product team regardless of their JS framework.

## Consequences

- ECharts is a **peer dependency** — never bundled into `@carbon/echarts-theme`.
- Framework adapters (`echarts-for-react`, `ngx-echarts`, `vue-echarts`) are the responsibility of consuming applications.
- The theme output is a plain JS object, usable with any ECharts adapter or directly via `echarts.init()`.
- The package tracks **ECharts 6.x as the minimum supported version** (`peerDependencies: ">=6.0.0"`).
  - ECharts 6.0 (July 2025) is now the npm `latest` release. All major framework adapters
    (`echarts-for-react ≥3.0.6`, `vue-echarts ≥8.3.2`, `ngx-echarts ≥22.0.0`) support it.
  - Starting from v6 aligns the ZRender version with the `@carbon/echarts-motion` exploration
    (Mike Olasov's `temp/Carbon-Next-Motion-Mike/`), avoiding a retroactive ZRender 5→6 port
    when that package is built.
  - ECharts 5.x installations will receive a peer dependency warning from npm/pnpm; the floor
    is not enforced at runtime, but v6-only option keys used in presets will not work on 5.x.
