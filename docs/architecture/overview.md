# Architecture Overview

`@carbon/echarts-theme` is a pnpm workspaces monorepo containing five packages and a shared tooling layer.

---

## Repository structure

```
packages/
├── theme/          @carbon/echarts-theme        — core theme, published to npm
├── site/           (internal)                   — showcase site + dev harness
├── toolbar/        @carbon/echarts-toolbar       — chart toolbar + export, published to npm
├── motion/         @carbon/echarts-motion        — ECharts motion engine, published to npm (pending M2)
└── codemods/       @carbon/echarts-codemod       — migration transforms, published to npm (pending)
```

Root-level tooling: commitlint, ESLint, Prettier, Husky, Release Please, pnpm workspaces.

---

## `packages/theme` — `@carbon/echarts-theme`

The core publishable package. Zero runtime dependencies.

```
src/
├── tokens.ts         Carbon token → ECharts key mapping (derived from @carbon/themes at build time)
├── palettes.ts       IBM data-vis palettes: categorical, sequential, diverging, alert
├── skeleton.ts       showSkeleton / createSkeletonCSS loading-state helpers
├── themes/
│   ├── white.ts
│   ├── g10.ts
│   ├── g90.ts
│   └── g100.ts
├── presets/          createXxxOptions() helpers — one file per chart type
│   ├── _transform.ts shared data transforms (groupByGroup, pickColors, …)
│   ├── bar.ts
│   ├── line.ts
│   └── … (18 preset files total)
└── index.ts          Public API
```

### Public API

```ts
// Register all four themes on an echarts instance
export function registerCarbonThemes(echarts: EChartsType): void

// Raw theme objects — pass directly to adapters or echarts.registerTheme()
export { carbonWhite, carbonG10, carbonG90, carbonG100 }

// IBM data-vis color palettes
export { palettes }  // { categorical, sequential, diverging, alert }

// Flat token map for custom extensions
export { tokens }    // { white, g10, g90, g100 }

// Preset helpers (subpath: @carbon/echarts-theme/presets)
export { createBarOptions, createLineOptions, … }
```

Theme names for use with adapters: `"carbon-white"` | `"carbon-g10"` | `"carbon-g90"` | `"carbon-g100"`.

---

## `packages/toolbar` — `@carbon/echarts-toolbar`

A framework-agnostic toolbar for ECharts instances.

| Entry point                       | Contents                                                |
| --------------------------------- | ------------------------------------------------------- |
| `@carbon/echarts-toolbar`         | Core: data extraction, CSV/PNG export, fullscreen       |
| `@carbon/echarts-toolbar/vanilla` | `createChartToolbar` + `autoToolbar` imperative DOM API |
| `@carbon/echarts-toolbar/styles`  | Pre-compiled `dist/styles.css`                          |

The `core/extract/` directory contains per-chart-type data extractors used to power the "show as table" modal. `buildTableData` in `core/extract/index.ts` dispatches to these based on the ECharts `option` object.

---

## `packages/site`

Vite + React showcase site. Serves as both the **development harness** (adding a preset → adding a page, visual verification in the same environment) and the **public-facing showcase**.

See [`site.md`](./site.md) for routes, components, and deployment.

---

## `packages/codemods` — `@carbon/echarts-codemod`

jscodeshift-based CLI. Scaffolded; implementation pending.

```sh
npx @carbon/echarts-codemod carbon-charts-to-echarts ./src
npx @carbon/echarts-codemod echarts-theme-swap ./src
```

See ADR 0007 for the co-location rationale.

---

## Dependency graph

```
@carbon/themes (dev only)          @carbon/motion (dev only)
       ↓ build-time                        ↓ build-time (Tier A tokens)
packages/theme  ──────────────────────────────→  npm (@carbon/echarts-theme)
       ↓ workspace:*
packages/toolbar ─────────────────────────────→  npm (@carbon/echarts-toolbar)
packages/motion  ─────────────────────────────→  npm (@carbon/echarts-motion) [pending M2]
packages/site    (internal, not published)
packages/codemods ────────────────────────────→  npm (@carbon/echarts-codemod) [pending]
```

`echarts` is a **peer dependency** of `theme`, `toolbar`, and `motion` — never bundled.

`--cds-motion-*` CSS custom properties (Carbon v12 motion surface API) are read at runtime by
`@carbon/echarts-motion` when present — not bundled. See [ADR 0010](../adr/0010-motion-token-source-of-truth.md).

---

## Architecture rules

These rules govern where logic lives. Never deviate.

1. All structural chart logic belongs in `packages/theme/src/presets/` — never in site data files or page components.
2. All palette colors must come from `pickColors()` or named exports from `palettes.ts` — never hardcode hex in presets or data files.
3. Site data files (`packages/site/src/data/echarts/*.ts`) only call preset functions with data — no logic.
4. Site page files only import from data files and wire props — no option construction.
5. Every fix must be a general preset option, not a one-off bespoke override.
6. `packages/theme` is a plain JS object — zero runtime dependencies, no framework code, SSR-safe. Lifecycle-bound functionality (anything requiring a mounted chart) belongs in `packages/motion` or `packages/toolbar`. See [ADR 0002](../adr/0002-no-framework-adapters.md) and [ADR 0009](../adr/0009-motion-package-boundary.md).
7. Motion token values are never hardcoded in `packages/theme` or `packages/motion`. Existing `@carbon/motion` tokens are sourced at build time; new `--cds-motion-*` tokens are read from CSS custom properties at runtime. See [ADR 0010](../adr/0010-motion-token-source-of-truth.md).
