# ADR 0010 — Motion token source of truth: consume from Carbon v12's motion surface API

**Status:** Proposed  
**Date:** October 2026  
**Author:** Jeff Chew  
**Relates to:** ADR 0003 (token derivation at build time), ADR 0009 (motion package boundary)

---

## Context

Carbon v12 is building a **motion surface API** into the core libraries. This includes a
`--cds-motion-*` custom property layer with tokens that go significantly beyond what
`@carbon/motion` 11.x ships today. Mike Olasov's `motion-core` package (`temp/Carbon-Next-Motion-Mike/`)
is the prototype for this API and is expected to be the upstream source from which the v12
token layer is derived.

The current prototype token surface, relative to `@carbon/motion` 11.x, adds:

| New token category               | Examples                                                                                             | Status                                               |
| -------------------------------- | ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| **Additive durations**           | `fast-00` (40ms), `moderate-03` (320ms), `instant`, `frame`                                          | Proposed — fills gaps in the existing ladder         |
| **Spring tokens**                | `snap`, `settle`, `bouncy`, `gentle` — `{ visualDuration, bounce }`                                  | Proposed — open question in Carbon re: bounce ban    |
| **Stagger tokens**               | `tight`, `default`, `loose` — `{ each, window }`                                                     | Proposed                                             |
| **Distance tokens**              | `nudge`, `xs`, `sm`, `md`, `lg`, `xl`                                                                | Proposed                                             |
| **Scale tokens**                 | `subtle`, `enter`, `emphasis`, `press`                                                               | Proposed                                             |
| **Role tokens**                  | `enter`, `exit`, `move`, `emphasis`, `dismiss` — composite `{ duration, easing, distance?, scale? }` | Proposed                                             |
| **Chart reveal roles**           | `front`, `move`, `phase-gap`, `rule`, `projected-front`, `dissolve`                                  | `@proposal` — chart-specific, aliases existing rungs |
| **Graded reduced motion**        | `duration-scale`, `distance-scale`, `allow-spatial`, `duration-floor`                                | Proposed                                             |
| **Origin / direction / pointer** | `origin-x/y`, `pointer-x/y/dx/dy/angle/speed`                                                        | Proposed                                             |

The existing six durations and six easings are **byte-identical** to `@carbon/motion` 11.52.0
(verified by `assertCarbonParity()` in `motion-core`). They are not new — they are the existing
tokens, re-expressed as CSS custom properties.

### The problem with the current approach

ADR 0003 established that color tokens are sourced from `@carbon/themes` at build time and
inlined. That works well for colors because `@carbon/themes` is stable and versioned. We are
now doing the same thing for motion: [`packages/theme/src/themes/factory.ts`](../../packages/theme/src/themes/factory.ts)
has hardcoded motion values (`300`, `'cubicOut'`), and the Tier 1 work from ADR 0009 proposes
replacing them with values from `@carbon/motion` 11.x, also inlined at build time.

That approach is correct for the **existing** six durations and six easings, which are already
stable in `@carbon/motion`. But it becomes a liability for the **new** tokens (spring,
stagger, role, chart-reveal, reduced-motion grades) that are being designed as a runtime CSS
custom property layer — `--cds-motion-*` — precisely so they can be re-tuned without a rebuild.

If we inline the new token values at build time into the ECharts theme object, we:

1. Create a parallel, private copy of values that live in `--cds-motion-*` in the page's CSS
2. Guarantee drift: when v12's motion surface API updates a value, our inlined copy doesn't
3. Miss the point: the whole design of the runtime layer is that charts are re-tuned by the
   same token override that re-tunes everything else on the page

### What "consuming the runtime layer" means for ECharts

ECharts' `animationDuration` and `animationEasing` are option-level keys, not CSS properties.
They cannot read from CSS custom properties directly. Bridging them requires a small read at
chart initialisation time:

```ts
// read at init, not at module load — so the runtime layer is fully applied
const duration = readMotionVar(el, 'duration-moderate-02') // reads --cds-motion-duration-moderate-02
const easing = readMotionVar(el, 'easing-entrance-productive')
```

`readMotionVar` is already implemented in `motion-core/tokens.ts` (`getComputedStyle` +
`parseCssTime`). The pattern is: read once when the chart mounts, pass the values into
`setOption`. When a token override changes (theme switch, reduced-motion change), the chart
re-reads and re-calls `setOption` — which is a normal update, not a reinit.

For tokens that ECharts cannot consume natively (spring parameters, stagger `each`/`window`,
role distances) — these live entirely in `@carbon/echarts-motion`'s engine and are read there.

---

## Decision

**Token sourcing is split by token type:**

### Tier A — Existing `@carbon/motion` 11.x tokens (build-time inline, as today)

The six durations and six easings that exist in `@carbon/motion` today are sourced from
`@carbon/motion` at build time, following ADR 0003's model. Specifically:

- `@carbon/motion` remains a dev dependency of `@carbon/echarts-theme`
- `animationDuration` / `animationDurationUpdate` / `animationEasing` in `factory.ts` are set
  to values read from `@carbon/motion` at build time
- `assertCarbonParity()` (or an equivalent check) runs in CI to catch drift if `@carbon/motion`
  updates its values

This is the Tier 1 change from ADR 0009. It is safe to do now.

### Tier B — New motion surface API tokens (runtime CSS custom property read)

When Carbon v12 ships `--cds-motion-*` as part of its core CSS, `@carbon/echarts-motion`
will read those properties at chart mount time rather than inlining values at build time.

- `@carbon/echarts-motion` will **not** bundle its own copy of motion token values for the
  new token categories
- Instead, the package reads `--cds-motion-*` from the DOM at chart init time using
  `getComputedStyle` on the chart's container element
- If the custom property is absent (consumer has not loaded Carbon v12's motion CSS), the
  package falls back to the Tier A build-time values for the tokens that exist in `@carbon/motion`,
  and to documented hardcoded fallbacks for the new categories
- The chart's motion re-synchronises whenever the option is rebuilt (theme switch, responsive
  breakpoint, reduced-motion change) — the caller rebuilds the option with the new read values

### Tier C — Chart-reveal role tokens (`--cds-motion-chart-reveal-*`)

These are `@proposal` in `motion-core` — aliases of existing Carbon rungs with chart-specific
names. They follow the same Tier B pattern: read from CSS custom properties if present, fall
back to the aliased Tier A values if not. No chart-specific tokens are inlined at build time.

The chart-reveal tokens are defined by `@carbon/echarts-motion`, not re-defined. If v12 ships
them in `@carbon/motion` or a related package, we read them from there. If they stay
chart-package-local (open question Q6 from ADR 0009), we define them in `@carbon/echarts-motion`
and they are still expressed as CSS custom properties, not inlined values.

---

## Rationale

### Why not inline everything at build time (extend ADR 0003 to all motion tokens)?

ADR 0003's model works for color because:

- Color token values are stable (a rebuild on token change is acceptable)
- ECharts cannot read CSS custom properties for color either — there is no alternative

For motion, there is an alternative: CSS custom properties exist precisely for runtime theming.
The v12 motion surface is being built as a runtime layer specifically so a dark-mode toggle,
a reduced-motion preference change, or a density theme switch retimes everything without a
JS rebuild. Inlining defeats that design.

### Why not read ALL motion values from CSS at runtime (abandon build-time inline entirely)?

For the existing six durations and six easings, the build-time approach is already correct
and well-tested. There is no value in switching them to a runtime read while Carbon v12's
motion CSS is still `@proposal`. The fallback story would be: "if `--cds-motion-duration-moderate-02`
is not set, use 240ms" — which is exactly what inlining at build time already gives us, more
reliably.

The runtime read becomes the right model when the CSS layer is stable and shipped. Until then,
the build-time values are the fallback and the runtime read is the progressive enhancement.

### Why not copy `motion-core`'s token values directly into `@carbon/echarts-motion`?

`motion-core/tokens.ts` comments its values as deliberately byte-identical to `@carbon/motion`
11.52.0 and explicitly does not import `@carbon/motion` as a runtime dependency. It runs
`assertCarbonParity()` to catch drift. That is the right approach for a prototype.

For a published package, the right model is to source from `@carbon/motion` directly (for Tier A)
and from the runtime CSS layer (for Tier B) — not to maintain a third copy. Copying
`motion-core`'s values would create a fourth copy and make `assertCarbonParity()` the only
thing standing between them.

### Relationship to ADR 0003

ADR 0003 is not superseded — its principle ("single source of truth") is preserved. The
refinement is that there are now two sources of truth depending on token tier:

| Token tier                                    | Source of truth                                       | When read             |
| --------------------------------------------- | ----------------------------------------------------- | --------------------- |
| Existing `@carbon/motion` durations + easings | `@carbon/motion` (npm package)                        | Build time            |
| New `--cds-motion-*` runtime tokens           | Carbon v12's motion CSS layer                         | Runtime (chart mount) |
| Chart-reveal roles                            | `@carbon/echarts-motion` (pending v12 decision on Q6) | Runtime (chart mount) |

---

## Consequences

- **No change to ADR 0003 for color tokens.** The color token sourcing from `@carbon/themes`
  is unaffected.
- **Tier A (Tier 1 of ADR 0009) can proceed now.** Replace `factory.ts`'s hardcoded `300` / `'cubicOut'`
  with values from `@carbon/motion`. This does not require any runtime CSS read.
- **`@carbon/echarts-motion` must not bundle its own `--cds-motion-*` values** for the new
  token categories. It reads from the DOM. Fallbacks are documented and tested.
- **A `readMotionVar` utility** (equivalent to `motion-core`'s) will be part of
  `@carbon/echarts-motion`'s internal API. It reads a `--cds-motion-*` property from a given
  element and returns a typed value (number for duration, cubic-bezier tuple for easing).
- **Reduced-motion is also runtime.** `prefers-reduced-motion` is a browser event, not a build
  flag. `@carbon/echarts-motion` reads the CSS reduced-motion scale properties at chart mount
  and on any preference change, then updates the option. This replaces the current plan of
  an opt-in `carbonAnimationOption({ reduced: boolean })` — the chart reads the state itself.
- **Gated on v12 motion CSS shipping.** Tier B work does not start until `--cds-motion-*`
  is available in a stable Carbon release. In the interim, the package ships with Tier A
  build-time values for all six existing tokens, and no read for the new categories.

---

## Open questions

| #   | Question                                                                                                                                       | Who answers          |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| Q-A | Which package will ship `--cds-motion-*` CSS in Carbon v12 — `@carbon/styles`, `@carbon/motion`, or a new package?                             | Mike / v12 team      |
| Q-B | Will `--cds-motion-chart-reveal-*` tokens be in the core motion CSS or in `@carbon/echarts-motion`? (Open Q6 from ADR 0009)                    | Mike / tiger team    |
| Q-C | When does Carbon v12's motion CSS ship a stable (non-`@proposal`) version?                                                                     | v12 timeline         |
| Q-D | Are spring tokens going into v12 core? Carbon has a published bounce ban; `bouncy` is explicitly flagged as an open argument in `motion-core`. | Mike / Carbon design |
