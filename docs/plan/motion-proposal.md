# Motion for the ECharts theme — analysis and proposal

**Status:** Active — decisions recorded; implementation tracked in [`roadmap.md`](./roadmap.md) M1–M6
**Last updated:** October 2026

---

## 1. Purpose

Mike Olasov ("Motion Mike") produced a large set of motion explorations for data visualisation as the **Motion — Exploration** track of Carbon Next / Tiger Sprint 2. This work is being incorporated into Carbon v12, which is a **v12 GA deliverable for this team**.

This document records:

1. What is in his repository (a local copy was placed in `temp/Carbon-Next-Motion-Mike/`, which is git-ignored).
2. How it relates to this repo's current state.
3. The decisions made about package ownership and token sourcing (see ADRs 0009 and 0010).
4. The remaining open questions and the implementation work tracked in the roadmap.

> Scope of review: the plan, package contract, principles doc, token proposals and the source of `motion-charts`, `motion-core` and `motion-gpu` were read. **None of his code was built or run**, and the prototype app was not reviewed beyond its file layout. Claims about runtime behaviour below are Mike's own documented findings, not independently verified.

---

## 2. Current state of this repo

Motion in `@carbon/echarts-theme` is a single block of static defaults in `packages/theme/src/themes/factory.ts`:

| Key                       | Value        |
| ------------------------- | ------------ |
| `animation`               | `true`       |
| `animationDuration`       | `300`        |
| `animationEasing`         | `'cubicOut'` |
| `animationDurationUpdate` | `300`        |

These are documented in [`architecture/token-mapping.md`](../architecture/token-mapping.md#animation-defaults) as matching the Carbon Charts defaults.

What is **not** there today:

- No link to Carbon motion tokens (durations, easings).
- No `prefers-reduced-motion` handling.
- No entrance choreography, stagger, hover behaviour or update-versus-entrance distinction beyond the above.

Existing architecture constraints that bear on this (see ADRs):

- [ADR 0002](../adr/0002-no-framework-adapters.md): the theme package is a plain object with no runtime and no framework code.
- [ADR 0008](../adr/0008-toolbar-separate-package.md): precedent for shipping non-declarative functionality as a **separate package** with `echarts` as a peer dependency.
- [ADR 0009](../adr/0009-motion-package-boundary.md): **decided** — `@carbon/echarts-motion` is a separate package owned by this team; the Carbon Next tiger team contributes after v12 GA.
- [ADR 0010](../adr/0010-motion-token-source-of-truth.md): **proposed** — existing `@carbon/motion` tokens sourced at build time (Tier A); new `--cds-motion-*` runtime tokens read from CSS custom properties when available (Tier B).
- The theme package peer range is `echarts >=5.0.0` and it targets ECharts 6.1 in dev.

---

## 3. What is in Mike's repository

A pnpm/turbo monorepo (`Carbon-Next`, IBM GHE). Chart-relevant parts:

| Package / app       | Size (approx.) | What it is                                                                                                                                           |
| ------------------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `motion-core`       | 500K           | `--cds-motion-*` tokens (DTCG JSON, CSS, TS), React primitives, layout transactions, choreography, `useCursorDirection`                              |
| `motion-charts`     | 350K           | ECharts/ZRender substrate: `useECharts`, `createFrameDriver`, `getPlotRect`, `createFillLayer`, `createReveal`, `cursorPips`, easing, reduced motion |
| `motion-gpu`        | 500K           | WebGL2 / canvas / DOM "fill surfaces" for the measured / modeled / projected dither texture, with a mandatory fallback                               |
| `motion-icons`      | ~15M           | Animated Carbon icons for React — hover, toggles, status, actions, press. `v0.1.0`, not yet published. Not chart-related                             |
| `motion-icons-core` | —              | Framework-free spec + engine shared by React and WC variants. Bundled into `motion-icons` dist                                                       |
| `motion-icons-wc`   | —              | Web Component version of the same icons (separate README)                                                                                            |
| `apps/prototypes`   | large          | The C1–C10 chart family, dashboard and layout labs. A reference app, explicitly not a component library                                              |
| `apps/storybook`    | —              | Storybook for `motion-icons`. Published to gh-pages alongside the prototypes                                                                         |
| `apps/wc-harness`   | —              | Playwright harness for WC + React icon real-browser tests (Chrome + WebKit)                                                                          |

Key documents in his repo: `MOTION-PROTOTYPES-PLAN.md`, `docs/PACKAGE-CONTRACT.md`, `docs/motion-principles.md`, `docs/chart-family/TOKEN-PROPOSALS.md`.

### 3.1 Principles relevant to charts

From `docs/motion-principles.md` (several are marked _Draft_ / _Provisional_):

- **2 — Changes move from where they are.** An update never replays an entrance.
- **3 — Data animates once.** Re-animate only when something logically changed.
- **4 — Reduced motion removes movement, not meaning.** Fades only, no growing or sweeping shapes.
- **5 — Interaction answers the pointer.** Hover is a "snap": marks appear at half size, surfaces at about 85%.
- **13 — Charts reveal data in the order it happened.** Measured first, projections after.
- **Timing** uses Carbon's duration ladder (`fast-02` 110 ms, `moderate-02` 240 ms, `slow-02` 700 ms), with easings byte-identical to `@carbon/motion` 11.x.

### 3.2 Technical findings worth carrying forward

These are documented in his plan (§1.3, §5.5, §5.6, §5.6b) and are useful independent of whether the code is adopted:

| Finding                                                                                                                             | Why it matters for us                                                          |
| ----------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| ECharts' line/area entrance is a **clip-rect wipe**; the path cannot animate per vertex natively                                    | A "grow from baseline" line reveal needs custom code (his `createReveal`)      |
| Bars and scatter **do** stagger natively via `animationDelay`                                                                       | No engine needed for those; a helper is enough                                 |
| ZRender stops its frame loop after 10 idle frames, so a `'frame'` handler on a settled chart never fires                            | Any frame-driven motion needs a wake/restore wrapper (his `createFrameDriver`) |
| `showSymbol: false` still draws a hover symbol using ECharts' **default palette** if `option.color` is unset                        | A latent theme-correctness issue: our themes should be checked for this        |
| ECharts and ZRender do not honour `prefers-reduced-motion`. `animation: false` is the wrong fix because it removes object constancy | We need an explicit reduced-motion strategy                                    |
| ECharts 6's advertised `bar range` / `line range` custom charts are **not shipped** in the package                                  | Relevant to any error-bar / range work, independent of motion                  |

### 3.3 What is new since the prior analysis

Four categories of new work landed since this document was first written:

#### New `motion-charts` primitives

| File            | What it adds                                                                                                                                                                                                                                                                                                    |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `fillLayer.ts`  | `createFillLayer(chart)` — inserts a ZRender custom layer as the documented fill slot. Exports `CHART_ZLEVEL` constants (`base: 0`, `fill: 1`, `rule: 1.5`, `data: 2`, `overlay: 3`). Also handles `getDataURL()` compositing and guards against SVG / single-canvas renderers.                                 |
| `useECharts.ts` | Hand-rolled React 19 ECharts hook. Init once, StrictMode-safe (guard against double-invoke), `ResizeObserver`-based resize that skips a repaint when size unchanged (prevents cancelling native bar entrance), `setTheme()` without dispose, reference-comparison option updates. Replaces `echarts-for-react`. |
| `types.ts`      | Shared type aliases: `ECharts = EChartsType`, `ZRender`, `EasingFn`, `CubicBezier`, `PlotRect`.                                                                                                                                                                                                                 |

All three are now publicly exported from the package's `index.ts`.

#### `motion-icons` — animated Carbon icons (new package)

A complete animated-icon library for React (`@carbon-next/motion-icons`), `v0.1.0`, not yet published, license pending IBM legal. Key facts for our decision-making:

- **Drops into Carbon icon slots.** Same 32×32 viewBox, `currentColor`, same size prop. At rest it is byte-identical to `@carbon/icons-react` (verified by a parity test).
- **Behaviours:** hover (mouse/pen/keyboard, not touch), toggles with `asButton`/`aria-pressed`, status marks with `count`/`tone`, one-shot and looping actions, press scale (1px inset at every size, on the drawing group so it composes with everything else).
- **Accessibility:** `prefers-reduced-motion: reduce` → still by default; `animated` override exists. `forced-colors` sheet moves toned paints to `currentColor`. Keyboard focus triggers hover; touch does not.
- **Bundle:** 10.5 kB gzip for a hover icon, 404 kB for the whole barrel. `sideEffects: false`. ESM only.
- **React 18+.** Tested on React 19; a React 18 compatibility suite exists but has not run against a real React 18 install.
- **Engine is custom.** Does not use `motion.dev`; `motion-icons-core` has its own tween (`ease.ts`, `engine/`) so the package has zero non-Carbon deps.
- This is **out of scope for the echarts-theme repo** — confirmed non-chart-related. It is noted here because it clarifies that Mike's repo is now a complete motion framework, not just a prototype.

#### v12 Explorer's Guide analysis (`V12-GUIDE.md`)

Mike read the Figma v12 Explorer's guide (frame `2281:82264`) and compared it against the C1–C10 prototypes. The result is a 20-row match/conflict/missing table. Most conflicts are internal to the prototype family and not yet decisions for the echarts-theme repo. Key points that bear on us:

- The guide establishes the **texture semantics** the theme will need to support eventually: solid = observed, dot = projected, hatch = qualified/estimate, cross-hatch = compounded. ECharts supports solid and dot natively; hatch and cross-hatch require the GPU or CPU fill surface.
- **No value draws to zero unless zero is the true value** — applies to live/real-time data. A stale feed holds its last value. ECharts has no native "stale holds" behavior; this must be enforced by the option builder.
- **Freshness/receipt states** (Live, Stale, Paused, Failed) are defined icons and key cards. The motion-icons package has a `Live` icon with a `status` prop; the Paused state requires a play/pause toggle.
- Open questions from the comparison are in `V12-GUIDE.md §6` — those are for Mike to answer, not us.

#### Chart color token proposals (`TOKEN-PROPOSALS.md`)

A second tranche of token proposals was added alongside the existing motion tokens:

- **OKLCH-based primitives** — five hue ramps (`chart-<hue>-<L>` at fixed OKLCH lightness L=27/37/47/57/67/77/87) plus two tinted gray ladders (`chart-gray-cool/warm-NN`). These are not Carbon v11 colors; they are Carbon **Next** proposals.
- **Roles** — `chart-cool/warm/categorical-cool/warm-NN` (1–10 series), `chart-estimate` (the orange qualified/estimate color), `chart-projected-fill-NN` (the OKLCH computed solid tone for the dotted-outline projected fill look).
- **Vivid default palette** — `chart-vivid-NN` (1–10) built entirely from Carbon v11 color steps, with separate light/dark values and large/small-mark tiers. `chart-vivid-01 = interactive` in both themes, so the first series' blue is the family's projection blue.

**Nothing in this section changes our position:** these proposals are plainly marked as Carbon Next proposals in the source. The `chart-vivid-NN` roles are the most adoption-ready because they use existing v11 color steps. Everything else is experimental and should not be consumed until they ship in Carbon.

### 3.4 Coupling and portability of his packages

| Package         | Portability notes                                                                                                                                                                                                                                |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `motion-charts` | `private: true`, exports raw `.ts` from `src`, depends on `workspace:*` and exact `echarts`/`zrender` 6.1.0. Only `useECharts.ts` and `reducedMotion.ts` import React. `createFrameDriver`, `reveal`, `easing`, `getPlotRect` are framework-free |
| `motion-core`   | React 19.3 peer, depends on `motion` (motion.dev). Token files (`motion.tokens.json`, `tokens.css`, `tokens.ts`) are separable                                                                                                                   |
| `motion-gpu`    | React components over raw WebGL2 plus a large CPU fallback (about 2.5k lines in `band-fill-cpu.ts` alone)                                                                                                                                        |

Size indicators in `motion-charts`: `reveal.ts` is about 2.4k lines and `cursorPips.ts` about 1.9k lines. Both carry prototype-specific design properties from the C1 family and would need trimming before being a general-purpose API.

Undocumented ECharts/ZRender internals are relied on: `zr._sleepAfterStill` and `grid.coordinateSystem.getRect()`. Deep imports of `zrender/lib/...` are used. These can break on minor releases.

`useECharts` is now a first-class export. It is React 19 only (uses `useLayoutEffect` with an empty dep array and `getInstanceByDom` StrictMode guard). It is valuable as a reference implementation but is not proposed for adoption into the theme package, which is framework-free by ADR 0002.

`createFillLayer` is framework-free and the most portable of the new primitives. It requires the canvas (not SVG) renderer and a multi-canvas painter — returns `null` for SVG and single-canvas targets.

---

## 4. Proposal

### 4.1 Guiding principle

Because this work is expected to land in Carbon through the v12 tiger team, **this repo should consume and align with it, not fork it.** Specifically:

- Do not copy Mike's numbers into the theme as our own constants. They are still moving and will drift.
- Do not present his **proposed** tokens (for example the chart-reveal roles, the 150 ms phase gap, or the placeholder `moderate-03`) as Carbon tokens. They are labelled proposals in `TOKEN-PROPOSALS.md`.
- Keep the declarative theme declarative.

### 4.2 Package placement — decided

**Option B: `@carbon/echarts-motion` as a separate package in this repo.** Decided October 2026 — ECharts is this team's responsibility in the Carbon ecosystem. The tiger team contributes as upstream collaborators after v12 GA. See ADR 0009 for the full rationale.

<details>
<summary>Original options considered</summary>

| Option                                              | Pros                                                                 | Cons                                                                                                             |
| --------------------------------------------------- | -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **A. Put motion engine in `@carbon/echarts-theme`** | One install                                                          | Breaks ADR 0002's "plain object, no runtime" stance; forces ZRender imports and lifecycle code on every consumer |
| **B. New package `@carbon/echarts-motion`** ✓       | Matches ADR 0008 precedent; independent release cadence; opt-in cost | Another package to version and document                                                                          |
| **C. Leave in Mike's Carbon Next monorepo**         | Single owner for engine                                              | Depends on where v12 puts it; timing uncertain                                                                   |

</details>

### 4.3 Tiered approach

```
Tier 1  Theme animation keys driven by Carbon motion tokens         (this repo, low risk)
Tier 2  Framework-free ECharts motion engine                        (@carbon/echarts-motion, this repo)
Tier 3  GPU fills, choreography, icons, page-level motion           (out of scope for now)
```

#### Tier 1 — token-driven animation defaults in the theme

**Tracked as M1 in [`roadmap.md`](./roadmap.md).** Replace the hardcoded `300` / `'cubicOut'` in `factory.ts` with Carbon motion tokens and add a reduced-motion variant.

- Source from `@carbon/motion` 11.x at build time, per ADR 0010 Tier A. Do not inline values manually — add `@carbon/motion` as a dev dependency and read from it directly, so a version bump + rebuild stays in sync automatically.
- Add `assertCarbonParity` as a CI check to catch drift if `@carbon/motion` updates its values.
- Changing default durations is a visible behaviour change (300 → 240 ms). Call it out in the changelog. Making it opt-in first is acceptable if preferred.
- Reduced motion: `carbonAnimationOption({ reduced: boolean })` opt-in export. Fades only, durations on the short end of the ladder, floor of 1 ms so lifecycle callbacks still fire.
- Update the "Animation defaults" section of `token-mapping.md`.

#### Tier 2 — ECharts motion engine (`@carbon/echarts-motion`)

**Ownership settled (ADR 0009). Tracked as M2–M5 in [`roadmap.md`](./roadmap.md).** Package lives in `packages/motion/`, follows the `@carbon/echarts-toolbar` pattern.

Extraction order (M2 → M3 → M4 → M5):

1. **M2 — Scaffold**: `packages/motion/` workspace, publish config, peer dep `echarts`, zero framework deps.
2. **M3 — Engine primitives**: `createFrameDriver`, `getPlotRect`, `createFillLayer` + `CHART_ZLEVEL`. These are stable and framework-free today.
3. **M4 — Easing bridge + reduced motion**: `createCubicEasingFunc` easing bridge, `chartMotionTokens` / `motionOption` policy, `readMotionVar` for runtime CSS custom property reads (ADR 0010 Tier B).
4. **M5 — Reveal engine**: `createReveal` + `revealSchedule` after trimming C1-family-specific design props.

API shape: imperative and framework-free, e.g. `attachReveal(chart, opts)` returning a disposer, consistent with ADR 0002.

`useECharts` is not adopted into either package (ADR 0002: no framework code). It is the right reference implementation for consumer framework wrappers — better than `echarts-for-react` in every measurable way. Document it in the getting-started guide as the recommended React wiring.

`createCursorPips` (cursor hover snap) is tracked separately as **M6** at medium priority, pending Mike's confirmation that the design is settled.

#### Tier 3 — deferred

- **GPU / dither fill surfaces.** The visual encoding (measured = solid, modeled = dithered, offline = dashed outline) is a Charts-track design decision, the engine is large, and WebGL2 may be unavailable (Chrome 139+, VDI), so a fallback is mandatory.
- **Layout transactions and choreography, page-change motion.** These are page-level Carbon concerns, not chart-theme concerns.
- **Motion icons.** Separate track.

---

## 5. Risks

| #      | Risk                                                                                                | Mitigation                                                                                  |
| ------ | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| 1      | Moving target: many principles are Draft/Provisional and timings have been revised repeatedly       | Source from `@carbon/motion` at build time; `assertCarbonParity` CI check catches drift     |
| 2      | Token drift between this repo, Mike's `--cds-motion-*` layer and what v12 ships                     | ADR 0010: Tier A from `@carbon/motion`, Tier B from runtime CSS — no private copy of values |
| 3      | Proposed tokens mistaken for Carbon tokens                                                          | Only `@carbon/motion` 11.x tokens in Tier 1; new categories are runtime reads, not inlined  |
| 4      | Reliance on ECharts/ZRender internals (`_sleepAfterStill`, `getRect()`, deep `zrender/lib` imports) | Version-matrix tests in CI; fallbacks as in `getPlotRect`; see §8.6 stability table         |
| 5      | ECharts version range: Mike's work is pinned to 6.1.0, our peer range is `>=5.0.0`                  | Decide minimum for `@carbon/echarts-motion`; test on the supported range (open Q9)          |
| 6      | Bundle size: `reveal.ts` (~2.4k lines) and `cursorPips.ts` (~1.9k lines) are large                  | Trim C1-family-specific design props before adoption; set a size budget                     |
| 7      | Reduced-motion: "fades only" policy changed once already; final accessibility review pending        | Treat as provisional until Q11 is answered; floor of 1 ms ensures lifecycle callbacks fire  |
| ~~8~~  | ~~Two owners for the same ECharts-specific code~~                                                   | **Resolved** — single owner (this team); tiger team contributes after v12 GA. See ADR 0009. |
| 9      | Changing default animation values alters existing consumers' charts                                 | Call out as behaviour change in changelog; opt-in export as a transitional path             |
| ~~10~~ | ~~Ownership / contribution path~~                                                                   | **Resolved** — ECharts is this team's responsibility in the Carbon ecosystem. See ADR 0009. |

---

## 6. Open questions

For the **dev lead**:

1. ~~Is a separate motion package acceptable?~~ **Resolved (October 2026):** `@carbon/echarts-motion` is owned by this team. ECharts is this team's responsibility in the Carbon ecosystem. See ADR 0009.
2. Is an opt-in Tier 1 export acceptable, or should the theme's default animation keys change directly?
3. What is the minimum supported ECharts version for anything that uses ZRender internals?
4. What size or dependency budget applies to a motion package?

For **Mike / the tiger team**:

5. ~~Where will the ECharts motion engine live?~~ **Resolved (October 2026):** `@carbon/echarts-motion` lives in this repo. The tiger team contributes as upstream collaborators after v12 GA. See ADR 0009.
6. Will chart-specific roles (reveal front, phase gap, projected front, dissolve) be added to `@carbon/motion` or stay chart-package-local?
7. Is `--cds-motion-*` the settled namespace for v12?
8. Which parts of `motion-charts` does he consider stable enough to extract now (suggest: `createFrameDriver`, `getPlotRect`, `createFillLayer`, easing bridge, reduced-motion policy) and which are still exploratory (reveal, pips, snap)?
9. Which ECharts versions will v12 support?
10. Does he want help hardening the undocumented-internals wrappers with version tests?
11. Reduced motion: is "fades only" final, and who performs the accessibility review?
12. `useECharts` is a much better replacement for `echarts-for-react`. Should we point our docs and examples at it? (It's React 19; React 18 support is uncertain.)
13. The v12 guide has 8 open questions for Mike recorded in `V12-GUIDE.md §6` (projected color, forecast dot spacing, hatch, scatter grain, etc.). Are any of these settled enough to affect which chart types we prioritize for the motion package?
14. `chart-vivid-NN` is built from Carbon v11 steps — is it close enough to stable that it should inform `@carbon/echarts-theme`'s default palette when v12 lands?

---

## 7. Implementation status

| Item                                                                               | Status         | Roadmap       |
| ---------------------------------------------------------------------------------- | -------------- | ------------- |
| Ownership settled — `@carbon/echarts-motion` owned by this team                    | ✅ Done        | ADR 0009      |
| Token source of truth decision recorded                                            | ✅ Done        | ADR 0010      |
| Motion as high-priority v12 GA deliverable                                         | ✅ Done        | roadmap M1–M6 |
| Tier 1 — Carbon motion token defaults in `factory.ts`                              | 🔲 Not started | M1            |
| Tier 2 — scaffold `packages/motion/`                                               | 🔲 Not started | M2            |
| Tier 2 — engine primitives (`createFrameDriver`, `getPlotRect`, `createFillLayer`) | 🔲 Not started | M3            |
| Tier 2 — easing bridge + reduced motion + `readMotionVar`                          | 🔲 Not started | M4            |
| Tier 2 — reveal engine (`createReveal` + `revealSchedule`)                         | 🔲 Not started | M5            |
| Tier 2 — cursor pips (pending Mike design sign-off)                                | 🔲 Not started | M6            |
| Audit `showSymbol: false` default-palette leak                                     | 🔲 Not started | —             |
| Tier 3 (GPU fills, icons, layout motion)                                           | ⏸ Deferred     | —             |

---

## 8. How Mike's motion engine works on top of ECharts

This section is a technical explainer for anyone trying to understand the approach before reading the source code.

### 8.1 The two-layer model

ECharts and ZRender are two separate things. ECharts is the chart library — it owns the data model, the series rendering, the axes and the declarative option system. ZRender is the 2D rendering engine underneath it: it owns the canvas (or SVG), the scene graph of elements, and the requestAnimationFrame loop. Mike plugs in at both levels, using each for what it can actually do.

```
┌─────────────────────────────────────────────────────────────┐
│  ECharts declarative layer                                  │
│  animationDuration / animationEasing / animationDelay       │  ← Layer 1: use for bars,
│  (option keys)                                              │    scatter, native transitions
├─────────────────────────────────────────────────────────────┤
│  ZRender frame driver  (zr.animation 'frame' event)         │  ← Layer 2: write shape.points
│  writes shape.points / shape.stackedOnPoints after          │    for line/area reveals and
│  ECharts animators, before paint — deterministic override   │    overlay cursor motion
├─────────────────────────────────────────────────────────────┤
│  ZRender custom layer  (painter.insertLayer)                │  ← Layer 3: fill textures
│  extra <canvas> inserted between grid (zlevel 0) and        │    (dither, dissolve) sit
│  data series (zlevel 2)                                     │    above grid, below data
└─────────────────────────────────────────────────────────────┘
```

### 8.2 Layer 1 — ECharts' option-level animation (native, no custom code)

ECharts exposes seven animation keys on the option object. Mike uses three of them:

```ts
{
  animationDuration: 400,         // Carbon moderate-02 (enter)
  animationDurationUpdate: 200,   // Carbon fast-02 (update)
  animationEasing: fn,            // a (t: number) => number function
}
```

The critical detail: `animationEasing` accepts a raw easing **function**, not just a named string like `'cubicOut'`. There is no `registerEasing` API — the named easing table in ZRender is a fixed set of 31 and cannot be extended. Passing a function directly is the only supported bridge between Carbon's `cubic-bezier(...)` tokens and ECharts' animation system.

The easing bridge turns a Carbon token's control-point tuple `[0.2, 0, 0.38, 0.9]` into that function using ZRender's own `createCubicEasingFunc` (a deep import from `zrender/lib/animation/cubicEasing`), so the ECharts curve is mathematically identical to the CSS transition using the same token.

For **bars and scatter**, this is the whole answer. ECharts already grows bars from the value-axis baseline and can stagger them per index via `animationDelay: (dataIndex) => ms`. No engine needed.

### 8.3 Layer 2 — the ZRender frame driver and why it exists

#### The line/area problem

ECharts' line and area series entrance cannot do "each vertex grows from the baseline". What it actually does:

1. Draw the line at its **final shape** from frame 0.
2. Place a `Rect` clip with `width: 0` over it.
3. Animate the clip rect from `width: 0` → full width.

The silhouette never moves. It is progressively uncovered left to right. `animationDelay` on a line series only applies to the hover symbols, not to the path — the path is one `ECPolyline` element with one clip animator, so per-vertex timing is simply not expressible. This was verified by reading the compiled `LineView.js` and measuring vertex positions at 120 ms into a 1200 ms entry: byte-identical to final values throughout.

#### The sleep problem

ZRender runs one rAF loop per chart and stops it after `_sleepAfterStill` (default 10) consecutive frames where nothing repainted. A `'frame'` handler registered on a chart that has already finished its entrance **never fires**. Measured: 0 ticks in 1 second on a settled chart.

#### The execution-order guarantee

When ZRender's loop does tick, the order within each tick is fixed:

```
1. Every ECharts animator steps (writes its values)
2. 'frame' event fires  ← your handler runs here
3. paint
```

Step 2 runs after ECharts' own writes and before paint. So anything written from a `'frame'` handler wins deterministically every frame. It is an override, not a race. The cost is that ECharts' own animator keeps spending CPU on a value nobody paints — you should call `el.stopAnimation()` on any property you intend to own.

#### What createFrameDriver does

`createFrameDriver(chart, onFrame)` solves both problems:

- **Sleep:** on `driver.start()`, it reads ZRender's `_sleepAfterStill` field, sets it to `0` (never sleep), and calls `wakeUp()` to restart the rAF loop. On `driver.stop()`, it restores the original value. A `WeakMap` reference-counts this across multiple drivers on the same chart so they don't fight.
- **Handler:** registers `onFrame(t)` on `zr.animation.on('frame', ...)`, where `t` is milliseconds elapsed since `start()`, accumulated from ZRender's per-tick delta — correct at 60 Hz and 240 Hz alike.
- **Cleanup:** guards against calling `animation.off` on a disposed chart (which would throw), a real crash that was measured mid-reveal on component unmount.

#### What createReveal does with the frame driver

When called, `createReveal(chart, opts)`:

1. Walks the ZRender scene graph to find the `ECPolyline` (line) and `ECPolygon` (area) elements for each targeted series.
2. Stops ECharts' clip-rect animation on those elements (but keeps the clip itself, so the line doesn't draw outside the grid).
3. Computes a **reveal schedule** — for each vertex: when it starts moving, how long it takes, how far it travels.
4. Starts the frame driver. On every tick, calculates where each vertex should be at elapsed time `t` and writes directly into `shape.points` (line) and `shape.stackedOnPoints` (area band's lower edge).

### 8.4 The reveal schedule

The timing is kept in a separate, pure module (`revealSchedule.ts` — no DOM, no clock) so it can be unit-tested and composed without geometry.

The model is a single front that sweeps left to right across the plot:

```
front progress:   p(t) = frontEase(t / frontDuration)

vertex at u starts:  startAt(u) = frontDuration · frontEase⁻¹(u)
    where u = vertex's position across the plot, 0 = left edge, 1 = right edge

vertex openness:  openAt(u, t) = moveEase( clamp( (t − startAt(u)) / moveDuration ) )

reveal ends:      frontDuration + moveDuration
```

Typical values from `chartMotionTokens` (Carbon token names):

- `frontDuration` = 700 ms (`slow-02`) — time for the sweep to cross the plot
- `moveDuration` = 240 ms (`moderate-02`) — each vertex's own travel time
- `frontEase` = Carbon standard-productive `[0.2, 0, 0.38, 0.9]`

**Stagger distribution.** The stagger delays are spaced using the _inverse_ of the easing curve, which produces a long-gap / short-gap / long-gap pattern. Using the curve directly produces the opposite. The inverse is computed via a 256-sample lookup table plus 24-step bisection, built once per easing function and memoized. This runs at construction time, never per frame.

**Confidence bands (EmergeSpec).** A band's vertex starts only after its driving line vertex at the same x has landed (`lineDelay(x) + moveDuration`). The band reads the line's _settled_ y at that x — a static value, since the line is at rest by then. With `lead > 0` the band starts earlier and must track the line's live y per frame, which couples them at the frame level. The tradeoff is documented explicitly in the code.

**Annotation leaders (LeaderSpec).** `markLine` elements live in a separate ZRender component tree that scene-graph traversal misses. Mike finds them by position instead: a leader whose near endpoint is within 1 pixel of a data vertex belongs to it. The leader then grows from zero length and fades in, starting when its vertex lands.

**Reduced motion.** `frontDuration` is set to `0`. Every vertex departs at the same moment (`startAt(u) = 0` for all `u`) and travels together over a short, linear `moveDuration` (140 ms). The stagger is gone; the travel — which preserves object constancy — stays.

### 8.5 The fill layer

For the modeled/projected texture fill, ECharts' normal `areaStyle` cannot carry a fill that changes per pixel per frame. Mike inserts a ZRender **custom layer** between the grid and the data series.

`createFillLayer(chart)` calls `painter.insertLayer(zlevel, layer)` — the same mechanism ECharts GL uses for its WebGL canvas. ZRender then manages the layer's size, positions and clips it alongside its own canvases, resizes it with the chart, and notifies it to repaint after each chart paint (so the fill is never one frame behind the lines above it).

The result is a `<canvas>` element sitting between the grid canvas (zlevel 0) and the data series canvas (zlevel 2). Any renderer — WebGL2, CPU canvas, or plain DOM — can draw into it.

### 8.6 Stability of the undocumented internals

Two ECharts/ZRender internals are used. Both have been **verified against published TypeScript declarations across every major version since ECharts 5.0.0 / ZRender 5.0.0** (released 2020–2021):

| Internal                                                           | Nature                                                                                              | Present since                                                                                                                                                                                                 | Risk level                                                                                                                                                                                                                                                                                                          |
| ------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `zr.setSleepAfterStill(n)` and `zr.wakeUp()`                       | **Public API** — declared in `zrender.d.ts` as `setSleepAfterStill(stillFramesCount: number): void` | ZRender 5.0.0 (earliest checked) through 6.1.0                                                                                                                                                                | Low. These are in the public TypeScript types and have not moved in five years.                                                                                                                                                                                                                                     |
| `zr._sleepAfterStill` (field read)                                 | **Private field** — declared `private _sleepAfterStill` in `zrender.d.ts`                           | ZRender 5.0.0 through 6.1.0                                                                                                                                                                                   | Low-medium. Private but structurally stable. Only read, never written. Has a hardcoded fallback of `10` (ZRender's own default) if the field is absent or non-numeric.                                                                                                                                              |
| `chart.getModel().getComponent('grid').coordinateSystem.getRect()` | **Private method** on ECharts instance + undocumented `getRect()` on coordinate system              | ECharts 5.0.0 through 6.1.0: `getModel` is `private` on the `ECharts` class in all versions; `getRect()` is on the coordinate system's interface as optional (`getRect?: () => RectLike`) since ECharts 5.0.0 | Medium. The instance `getModel` is typed private (requires a cast to call). The `getRect` on the coordinate system has existed since 5.0.0. `getPlotRect` has a three-stage fallback: coordinate system → axis extremes via `convertToPixel` → full canvas bounds. A dev-mode warning fires if the fast path fails. |

In short: `setSleepAfterStill` and `wakeUp` are public and have been stable for five-plus years. `_sleepAfterStill` is a private field read only as a restore value and has an explicit fallback. `getPlotRect` is the most fragile, wraps an undocumented chain, but its fallback chain means a breakage degrades gracefully rather than crashing.

---

## 9. References

All paths below are in Mike's repository (local copy under `temp/Carbon-Next-Motion-Mike/`):

- `MOTION-PROTOTYPES-PLAN.md` — overview, ownership boundary (§5), reduced motion (§5.3), ECharts capability inventory (§5.5), reveal mechanism (§5.6), ECharts traps (§5.6b)
- `docs/PACKAGE-CONTRACT.md` — package signatures and corrections (including choreography contract added 2026-10-09)
- `docs/SESSION-STATE.md` — startup context: decisions, status, open questions
- `docs/motion-principles.md` — the principles and specifications
- `docs/chart-family/TOKEN-PROPOSALS.md` — proposed (not yet Carbon) chart motion and color tokens; includes vivid/OKLCH palette as of 2026-10-08
- `docs/chart-family/V12-GUIDE.md` — Figma v12 Explorer's guide read + conflict/match/missing comparison against prototypes (2026-10-05)
- `docs/chart-family/PLAN.md` — decisions log (rounds 1–13+), constraints, open questions
- `docs/chart-family/DESIGN-PROPS.md` — `DEFAULT_CHART_DESIGN` fields: fill, boundary, color roles, motion token names, layout
- `docs/chart-family/PHASE1-BRIEF.md` — brief for chart implementation agents (shared code entry points, ownership rules, verify steps)
- `docs/chart-family/FRAMEWORK-INVENTORY.md` — what a new chart can reuse
- `packages/motion-charts/src/` — `createFrameDriver.ts`, `getPlotRect.ts`, `fillLayer.ts`, `useECharts.ts`, `types.ts`, `reveal.ts`, `reducedMotion.ts`, `easing.ts`, `cursorPips.ts`
- `packages/motion-icons/README.md` — full API for the animated icon library
- `packages/motion-icons-core/src/` — engine specs and tokens shared between React and WC variants

---

## 10. What changed since the prior analysis

Summary of the delta for anyone who has already read §1–§9 and wants only what is new.

**New `motion-charts` exports (all from latest `index.ts`):**

- `createFillLayer` / `CHART_ZLEVEL` / `FillLayer` / `FillLayerFrame` — the ZRender custom-layer slot with `onPaint` callback, `fitCanvas` helper and `getDataURL` compositing.
- `useECharts` / `UseEChartsOptions` / `UseEChartsResult` — React 19 hook (init once, StrictMode-safe, `ResizeObserver`, `setTheme` without dispose, reference-identity option updates).
- `ECharts` / `ZRender` / `EasingFn` / `CubicBezier` / `PlotRect` — type aliases from `types.ts`.

**`motion-charts` `PACKAGE-CONTRACT.md` additions:**

- `useECharts` contract frozen alongside the original three.
- Choreography contract added 2026-10-09: `beginLayout`, `holdLayout`, `followLayout`, `afterLayout`, `beginPageChange` — page-level layout transactions. Charts are "follow" role.
- `useCursorDirection` hook added — cursor velocity, entrance/exit edge, aspect-normalized position, zero re-renders.

**`motion-icons` package added:**

- `@carbon-next/motion-icons` `v0.1.0` — 400 kB barrel, ESM only, React 18/19 peer, custom tween engine, `prefers-reduced-motion`, forced-colors, press, actions. Not chart-related but confirms the repo is a complete Carbon Next motion framework.

**Chart color token proposals (`TOKEN-PROPOSALS.md`) added since prior reading:**

- OKLCH color primitives (`chart-<hue>-<L>`, `chart-gray-<family>-NN`, `chart-orange-<L>`).
- Role tables: `chart-cool/warm/categorical-cool/warm-NN`, `chart-estimate`, `chart-projected-fill-NN`.
- Vivid default palette: `chart-vivid-NN` built from Carbon v11 steps (most adoption-ready; `chart-vivid-01 = interactive`).
- Projected fill computed color: `projectedFillColor` is an OKLCH step from card toward ink, not alpha transparency.

**v12 Explorer's Guide analyzed (`V12-GUIDE.md`, 2026-10-05):**

- Key for us: texture semantics (solid/dot/hatch/cross-hatch ↔ observed/projected/qualified/compounded) now have Figma documentation. The chart-theme will eventually need to express these.
- 8 open questions recorded for Mike in that doc (`§6`). None are blocking for the echarts-theme repo right now.
