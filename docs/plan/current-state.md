# Current State

> Last updated: October 2026

---

## Package status

| Package                   | Status             | Version                |
| ------------------------- | ------------------ | ---------------------- |
| `@carbon/echarts-theme`   | ✅ Published       | tracks Carbon v11      |
| `@carbon/echarts-toolbar` | ✅ Published       | independent versioning |
| `@carbon/echarts-motion`  | 🔲 Not started     | scaffolding pending M2 |
| `@carbon/echarts-codemod` | ❌ Not started     | scaffolded only        |
| Showcase site             | ✅ Live on Netlify | all 30 pages routed    |

---

## What is complete

### Theme core

- Four ECharts theme objects (White, G10, G90, G100) — all tokens derived from `@carbon/themes`
- IBM data-vis palettes: categorical, sequential, diverging, alert — light and dark variants
- `registerCarbonThemes(echarts)` public API — SSR-safe
- Skeleton / loading-state helpers (`skeleton.ts`)

### Presets (`@carbon/echarts-theme/presets`)

All 18 preset files shipping; 192 tests passing.

| Preset          | Exports                                                                                                                            |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `bar.ts`        | `createBarOptions`, `createGroupedBarOptions`, `createStackedBarOptions`, `createHorizontalBarOptions`, `createFloatingBarOptions` |
| `line.ts`       | `createLineOptions`, `createStepLineOptions`, `createTimeSeriesLineOptions`                                                        |
| `area.ts`       | `createAreaOptions`, `createStackedAreaOptions`, `createBoundedAreaOptions`                                                        |
| `donut.ts`      | `createDonutOptions`, `createPieOptions`                                                                                           |
| `scatter.ts`    | `createScatterOptions`, `createBubbleOptions`                                                                                      |
| `heatmap.ts`    | `createHeatmapOptions`                                                                                                             |
| `choropleth.ts` | `createChoroplethOptions`                                                                                                          |
| `gauge.ts`      | `createGaugeOptions`, `createMeterOptions`                                                                                         |
| `histogram.ts`  | `createHistogramOptions`                                                                                                           |
| `treemap.ts`    | `createTreemapOptions`, `createTreemapOptionsFromHierarchy`, `createRadarOptions`                                                  |
| `sunburst.ts`   | `createSunburstOptions`                                                                                                            |
| `boxplot.ts`    | `createBoxplotOptions`                                                                                                             |
| `combo.ts`      | `createComboOptions`                                                                                                               |
| `lollipop.ts`   | `createLollipopOptions`, `createSparklineOptions`                                                                                  |
| `alluvial.ts`   | `createAlluvialOptions`, `createAlluvialOptionsFromTabular`                                                                        |
| `tree.ts`       | `createTreeOptions`, `createTreeOptionsFromTabular`                                                                                |
| `network.ts`    | `createNetworkOptions`                                                                                                             |
| `wordcloud.ts`  | `createWordCloudOptions`                                                                                                           |

### Toolbar (`@carbon/echarts-toolbar`)

- `createChartToolbar` / `autoToolbar` vanilla DOM API
- Show-as-table modal (Carbon `cds--modal` classes)
- CSV export, PNG/JPG image export
- Fullscreen with Safari webkit fallback
- `extract/` refactored into 14 per-chart-type files under `core/extract/`

### Architecture decisions (ADRs)

Ten ADRs recorded and indexed in [`docs/README.md`](../README.md):

| ADR  | Decision                                                                   | Status   |
| ---- | -------------------------------------------------------------------------- | -------- |
| 0001 | Apache ECharts as the IBM / Carbon visualisation foundation                | Accepted |
| 0002 | No framework adapters shipped with the theme package                       | Accepted |
| 0003 | Design tokens derived from `@carbon/themes` at build time                  | Accepted |
| 0004 | pnpm workspaces monorepo                                                   | Accepted |
| 0005 | Release Please for versioning and changelog generation                     | Accepted |
| 0006 | Netlify for showcase site deployment                                       | Accepted |
| 0007 | Codemods co-located in this repository                                     | Accepted |
| 0008 | Toolbar as a separate published package                                    | Accepted |
| 0009 | Motion: keep the theme a plain object; engine in `@carbon/echarts-motion`  | Accepted |
| 0010 | Motion token source of truth: consume from Carbon v12's motion surface API | Proposed |

### Site

- All 30 chart pages exist and are routed (23 parity + 7 extended)
- Alphabetical nav matching `charts.carbondesignsystem.com`
- Side-by-side comparison panels with global theme switcher
- IBM footer integrated
- Deployed to Netlify

### CI / CD

- `ci.yml` — dedupe, format, lint, build + test, typecheck
- `release-please.yml` — automated Release PRs on push to main
- `publish.yml` — npm publish on tag push
- `deploy-site.yml` — Netlify deploy on non-RC tag push
- `dco.yml` — Developer Certificate of Origin check

---

## What is incomplete

| Area                                                                                           | Status                                              | Reference                                   |
| ---------------------------------------------------------------------------------------------- | --------------------------------------------------- | ------------------------------------------- |
| **Motion: Tier 1** — Carbon token defaults in `factory.ts` + reduced-motion export             | 🔲 Not started                                      | `roadmap.md` M1                             |
| **Motion: Tier 2 scaffold** — `packages/motion/` + `@carbon/echarts-motion` package            | 🔲 Not started                                      | `roadmap.md` M2                             |
| **Motion: Tier 2 engine primitives** — `createFrameDriver`, `getPlotRect`, `createFillLayer`   | 🔲 Not started                                      | `roadmap.md` M3                             |
| **Motion: Tier 2 easing + reduced motion** — easing bridge, `readMotionVar`, runtime CSS reads | 🔲 Not started                                      | `roadmap.md` M4                             |
| **Motion: Tier 2 reveal engine** — `createReveal` + `revealSchedule`                           | 🔲 Not started                                      | `roadmap.md` M5                             |
| **Motion: Tier 2 cursor pips** — pending Mike design sign-off                                  | 🔲 Not started                                      | `roadmap.md` M6                             |
| Toolbar tests                                                                                  | ❌ `__tests__/` is empty                            | `roadmap.md` W1                             |
| `skeleton.ts` tests                                                                            | ❌ No test file                                     | `roadmap.md` W2                             |
| Community health files                                                                         | ❌ CONTRIBUTING, SECURITY, CODE_OF_CONDUCT, SUPPORT | `roadmap.md` W6                             |
| Codemods implementation                                                                        | ❌ Not started                                      | `roadmap.md` W7                             |
| Migration guides                                                                               | ❌ Not written                                      | `roadmap.md` W8                             |
| Mend / PSIRT scan                                                                              | ❌ Blocked on IBM PSIRT registration                | `.github/ISSUE_TEMPLATE/mend-scan-setup.md` |
| AVT (IBM Equal Access) in CI                                                                   | ❌ Not wired                                        | `roadmap.md` W10                            |
