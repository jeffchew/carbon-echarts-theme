# Roadmap & Work Queue

Outstanding work in priority order. See [`current-state.md`](./current-state.md) for a full snapshot of what is already complete.

---

## High priority

| ID  | Area    | Task                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| --- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| W1  | Toolbar | Write tests in `packages/toolbar/src/__tests__/` — minimum one test per extractor in `core/extract/`. See `docs/legacy/12-code-review-tasks.md` T13 for suggested file structure and mock pattern.                                                                                                                                                                                                                                                     |
| W2  | Theme   | Write tests for `skeleton.ts` — `createSkeletonCSS` and `getSkeletonTokens` are pure and require no DOM. See T12 in the same doc.                                                                                                                                                                                                                                                                                                                      |
| M1  | Motion  | **Tier 1** — Replace hardcoded `animationDuration: 300` / `animationEasing: 'cubicOut'` in `factory.ts` with values from `@carbon/motion`. Add opt-in `carbonAnimationOption({ reduced: boolean })` export. Add `assertCarbonParity` CI check. See [ADR 0009](../adr/0009-motion-package-boundary.md) §Tier 1 and [ADR 0010](../adr/0010-motion-token-source-of-truth.md) §Tier A.                                                                     |
| M2  | Motion  | **Tier 2 — scaffold** — Create `packages/motion/` workspace: `@carbon/echarts-motion`. Peer dep: `echarts`. Zero framework deps. Publish config following `@carbon/echarts-toolbar` pattern. Add `motion` to `scope-enum` in [`commitlint.config.js`](../../commitlint.config.js) so Release Please tracks the new package. See [ADR 0009](../adr/0009-motion-package-boundary.md). ⚠️ _Spec needed: day-one public API surface (`index.ts` exports)._ |
| M3  | Motion  | **Tier 2 — engine primitives** — Extract and stabilise `createFrameDriver`, `getPlotRect`, `createFillLayer` + `CHART_ZLEVEL` from Mike's `motion-charts` into `@carbon/echarts-motion`. Framework-free, imperative API. Coordinate extraction order with Mike (ADR 0009 Q8). ⚠️ _Spec needed: exact post-extraction signatures, what changes vs. Mike's versions, test requirements._                                                                 |
| M4  | Motion  | **Tier 2 — easing bridge + reduced motion** — Add `createCubicEasingFunc` easing bridge and `chartMotionTokens` / `motionOption` reduced-motion policy to `@carbon/echarts-motion`. Implement `readMotionVar` for runtime CSS custom property reads (ADR 0010 Tier B). ⚠️ _Spec needed: `readMotionVar` API shape, fallback behaviour when `--cds-motion-*` is absent._                                                                                |
| M5  | Motion  | **Tier 2 — reveal engine** — Extract `createReveal` + `revealSchedule` into `@carbon/echarts-motion` after trimming C1-family-specific design props. Requires M3 and M4. ⚠️ _Spec needed: what gets stripped, minimum viable API, acceptance criteria (which chart type proves it works). Consider a POC against a real line chart before finalising the API._                                                                                         |

---

## Medium priority

| ID  | Area         | Task                                                                                                                                                                                                            |
| --- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| W4  | Code quality | Address code review findings from `docs/legacy/12-code-review-tasks.md`: `toArray` helper (T2), shared `resolveValue` helper (T7), floating-bar dedup (T8), `GRID` constant (T4), `tokens.ts` pick helper (T9). |
| W6  | Site         | Add missing GitHub community health files: `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`, `SUPPORT.md`.                                                                                                |
| M6  | Motion       | **Tier 2 — cursor pips** — Extract `createCursorPips` into `@carbon/echarts-motion` once Mike confirms the hover-snap design is settled. Requires M3.                                                           |

---

## Low priority / future

| ID  | Area     | Task                                                                                                                               |
| --- | -------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| W7  | Codemods | Implement `packages/codemods/` — see `docs/legacy/06-phase-6-migration-codemods.md` for the component→preset mapping and CLI spec. |
| W8  | Docs     | Write migration guides: `docs/migration-carbon-charts-to-echarts.md` and `docs/migration-echarts-to-carbon.md`.                    |
| W9  | CI       | Wire up Mend scan — blocked on IBM PSIRT product registration (see `.github/ISSUE_TEMPLATE/mend-scan-setup.md`).                   |
| W10 | CI       | Add AVT (IBM Equal Access) job to `ci.yml`.                                                                                        |

---

## Code quality work streams

These are incremental improvements with no functional changes. Recommended PR sequence:

**WS1 (quickfixes, single PR):** T1 activeSeries assignment → T2 toArray → T3 extract.ts comment → T4 GRID constant → T5 groupSparse export → T6 isLollipopSeries extraction

**WS2 (new helpers, before WS3):** T7 resolveValue → T8 buildFloatingSeries → T9 tokens pick helper

**WS3 (structural, after WS2):** T10 createBarOptions split → T11 buildTableData → build.ts

**WS4 (tests, last):** T12 skeleton tests → T13 toolbar extract tests

Full task specs with exact file locations and code sketches are in `docs/legacy/12-code-review-tasks.md`.

---

## Known ECharts limitations — do not attempt to fix

| Carbon Feature                         | Decision                                                          |
| -------------------------------------- | ----------------------------------------------------------------- |
| `tooltip.alwaysShowRulerTooltip`       | No ECharts equivalent — document in Overview tab                  |
| Skeleton / loading state               | Covered by `showSkeleton()` from `@carbon/echarts-theme/skeleton` |
| Bounded highlights (min/max bands)     | Approximated as stacked area — document the delta                 |
| Custom locale per chart                | ECharts uses `echarts.registerLocale()` globally — document       |
| Additional legend items (Bar slot 11+) | No ECharts equivalent                                             |
| Custom bin edges (Histogram)           | ECharts bin width only — document                                 |
| Circle pack                            | No native packed-circle layout — stub page                        |
| Choropleth                             | Requires consumer-provided GeoJSON + `echarts.registerMap()`      |
| Bullet chart                           | Layered bar approximation — not yet implemented                   |
