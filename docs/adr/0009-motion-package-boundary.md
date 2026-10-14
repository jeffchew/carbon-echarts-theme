# ADR 0009 — Motion: keep the theme a plain object; engine goes in a separate package

**Status:** Accepted
**Date:** October 2026
**Author:** Jeff Chew
**Relates to:** ADR 0002 (no framework adapters), ADR 0008 (toolbar as separate package)

---

## Context

Carbon Next / Tiger Sprint 2 produced a substantial motion exploration by Mike Olasov
(`temp/Carbon-Next-Motion-Mike/`, git-ignored). The repo contains a full ECharts/ZRender motion
engine (`motion-charts`), a GPU fill surface package (`motion-gpu`), a `--cds-motion-*` token
layer (`motion-core`), and animated Carbon icons (`motion-icons`). All of this is expected to
be incorporated into Carbon v12 through the tiger team.

This ADR records the decision about **where motion belongs** relative to `@carbon/echarts-theme`
and defines the boundary between what the theme package can own and what must live elsewhere.

The question was framed as: _should we just fully incorporate the motion package into the theme
so motion is a default behavior?_

### What "the theme" actually is

[`packages/theme/src/themes/factory.ts`](../../packages/theme/src/themes/factory.ts) produces a
**plain JavaScript object**. The entirety of motion in the current package is four static keys:

```ts
animation: true,
animationDuration: 300,
animationEasing: 'cubicOut',
animationDurationUpdate: 300,
```

This is not a limitation — it is load-bearing. The plain-object contract (ADR 0002) is what
makes `@carbon/echarts-theme` work in every framework, SSR contexts, vanilla scripts, and any
future ECharts adapter without a single runtime dependency.

### What the motion engine requires

The Tier 2 engine primitives from `motion-charts` are all **imperative, lifecycle-bound
operations** that must run after a chart is mounted to a real DOM element:

| Primitive                          | Why it cannot live in the theme object                                          |
| ---------------------------------- | ------------------------------------------------------------------------------- |
| `createFrameDriver`                | Calls `zr.setSleepAfterStill(0)` and `zr.wakeUp()` on a live `ECharts` instance |
| `createFillLayer`                  | Calls `painter.insertLayer()` and creates a DOM node                            |
| `createReveal`                     | Writes `shape.points` into ZRender's scene graph per animation frame            |
| `prefers-reduced-motion` detection | Requires `window.matchMedia` — breaks SSR                                       |
| `ResizeObserver` (in `useECharts`) | Requires DOM                                                                    |

None of these can be expressed as a static JSON property. They are lifecycle operations that
require a mounted chart, not a registered theme.

### What CAN go in the theme (Tier 1)

ECharts' native animation system — bar growth, scatter entrance, pie sweep, gauge arc — is
entirely controlled by option-level keys. These keys are already in `factory.ts`; they just
carry the wrong values. Replacing hardcoded numbers with Carbon token values is a pure data
change inside the existing plain-object model:

| Key                       | Current               | After Tier 1 | Carbon token                 |
| ------------------------- | --------------------- | ------------ | ---------------------------- |
| `animationDuration`       | `300`                 | `240`        | `duration-moderate-02`       |
| `animationDurationUpdate` | `300`                 | `110`        | `duration-fast-02`           |
| `animationEasing`         | `'cubicOut'` (string) | easing fn    | `easing-entrance-productive` |

This gives every consumer Carbon-correct animation for every natively-animated chart type
(bars, scatter, pie, gauge, heatmap) for free, with no opt-in required.

---

## Decision

**Motion is split into two tiers with a hard boundary between them.**

### Tier 1 — Token-driven defaults (this package)

Replace the four hardcoded animation keys in `factory.ts` with values derived from
`@carbon/motion` tokens. Add a small opt-in export for reduced-motion variants.

- Use only tokens that exist in `@carbon/motion` 11.x today. Do not use Mike's proposed
  chart-reveal roles (`chartRevealTokens.*`) — they are Carbon Next proposals, not yet shipped.
- Expose reduced-motion as an opt-in function (e.g. `carbonAnimationOption({ reduced: true })`)
  rather than a side-effecting import, to preserve SSR safety.
- Making this the **default** is acceptable because the value change is small and directionally
  correct (240 ms vs 300 ms, a Carbon easing vs a generic named string). It will be called out
  as a minor behavior change in the changelog.

### Tier 2 — ECharts motion engine (separate package)

Everything that requires a mounted chart instance lives in a separate package:
**`@carbon/echarts-motion`** in `packages/motion/`, owned by this team, following the ADR 0008
precedent for `@carbon/echarts-toolbar`. The Carbon Next tiger team will contribute to this
package after v12 GA.

Candidate contents, in extraction order:

1. `createFrameDriver` — ZRender sleep fix (~160 lines, no React dependency)
2. `getPlotRect` — undocumented API with 3-stage fallback (~290 lines)
3. `createFillLayer` + `CHART_ZLEVEL` — canonical canvas layer stack
4. Easing bridge (`createCubicEasingFunc` from ZRender)
5. Reduced-motion policy (`chartMotionTokens`, `motionOption`)
6. `createReveal` + `revealSchedule` — per-vertex line/area reveal (needs trimming)
7. `createCursorPips` — hover snap (needs trimming; Mike's sign-off on design stability)

API shape: imperative and framework-free, e.g. `attachReveal(chart, opts)` returning a
disposer, consistent with ADR 0002. A thin React hook wrapper can follow if needed.

`useECharts` (Mike's hand-rolled React 19 ECharts hook) is **not** proposed for adoption
into either package — ADR 0002 prohibits framework code. However it is the right reference
implementation for consumer framework wrappers and should be documented as the recommended
replacement for `echarts-for-react` in our getting-started guide.

### Tier 3 — Deferred

- GPU / dither fill surfaces (`motion-gpu`) — visual encoding is still a Charts-track design
  decision; WebGL2 fallback is mandatory and large.
- Hatch / cross-hatch textures — depend on v12 texture semantics being settled (documented in
  `V12-GUIDE.md §6`).
- Motion icons (`@carbon-next/motion-icons`) — separate track, not chart-related.
- Layout transactions, choreography, page-change motion — page-level Carbon concerns.
- OKLCH / Carbon Next color token proposals — do not consume until they ship in Carbon.

---

## Rationale

### Ownership

`@carbon/echarts-motion` is owned by this team. ECharts is this team's responsibility in the
Carbon ecosystem. The Carbon Next tiger team (Mike Olasov et al.) will contribute as upstream
collaborators after v12 GA — the same relationship as the toolbar: we own the package and
release cadence, they contribute the engine primitives they developed during the sprint.

This resolves the previous open questions Q1 ("is a separate package acceptable?") and Q5
("where does the engine live?"). Both are answered by ownership: the engine lives here.

### Why not bundle everything into the theme?

| Argument for bundling                        | Why it doesn't hold                                                                                                                                                                                                             |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "Simpler install, motion just works"         | Motion that requires DOM lifecycle cannot be expressed as a plain object at all — the install count is irrelevant                                                                                                               |
| "Consumers expect animations out of the box" | They already get them: ECharts native animation (bars, scatter, pie) is already on. Tier 1 makes the values Carbon-correct. The advanced engine is for line/area reveals and fill textures, which are progressive enhancements. |
| "One package is easier to document"          | The toolbar (ADR 0008) established that lifecycle-bound functionality ships separately. The precedent is already set.                                                                                                           |

### Why not wait entirely for v12?

Tier 1 (token-correct defaults) has no dependency on v12 or on Mike's work. The Carbon motion
token values have been stable since `@carbon/motion` 11.x and are verified byte-identical to
what Mike's engine uses. There is no reason to ship the wrong durations (`300ms`) while waiting
for a larger decision.

### Why a separate package rather than waiting for a Carbon-level package?

Ownership is now settled — the engine lives in this repo. There is no Carbon-level package to
wait for. The tiger team's prototype work (`motion-charts`) is the upstream source; this team
extracts, stabilises, and publishes the framework-free primitives as `@carbon/echarts-motion`.

---

## Consequences

- `factory.ts` animation keys will change to Carbon token values in the next minor release.
  This is documented as a behavior change in the changelog.
- A `carbonAnimationOption({ reduced: boolean })` export (or equivalent) will be added to
  the theme package's public API.
- When Tier 2 starts: a new `packages/motion/` workspace entry and `@carbon/echarts-motion`
  npm package will be created, with `echarts` as a peer dependency and zero framework deps.
- `useECharts` from Mike's `motion-charts` will be linked from the getting-started
  documentation as the recommended framework wrapper, not adopted as a package dependency.
- The Carbon Next tiger team contributes to `@carbon/echarts-motion` as upstream collaborators
  after v12 GA. This team owns the package boundary, release cadence, and public API.

---

## Open questions (remaining — Tier 2 can start, these inform the API)

Q1 and Q5 from `docs/plan/motion-proposal.md` are **resolved** by ownership (see Rationale).
Remaining questions are API and design decisions, not blockers.

| #   | Question                                                                                                                          | Who answers          |
| --- | --------------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| Q6  | Will chart-reveal role tokens be added to `@carbon/motion` or stay local to `@carbon/echarts-motion`?                             | Mike / tiger team    |
| Q8  | Which parts of `motion-charts` does Mike consider stable enough to extract first?                                                 | Mike                 |
| Q11 | Is "fades only" for reduced motion final? Who performs the accessibility review?                                                  | Mike + accessibility |
| Q12 | `useECharts` is React 19 only — should docs recommend it with that caveat, or wait for broader React support?                     | Dev lead             |
| Q13 | v12 guide conflicts (projected color, forecast dot spacing, hatch, scatter grain) — any settled enough to affect Tier 2 priority? | Mike                 |
