# ADR 0003 — Design tokens derived from `@carbon/themes` at build time

**Status:** Accepted  
**Date:** July 2026

---

## Context

The theme package needs to map Carbon design tokens (colors, typography, spacing) to ECharts theme keys. These values could be hardcoded as hex strings, or derived programmatically from the canonical `@carbon/themes` package.

## Decision

All token values are imported from `@carbon/themes` at build time and inlined into the distributed bundle. There is zero runtime dependency on Carbon.

## Rationale

- **Single source of truth.** Token values live in `@carbon/themes`. A Carbon token update requires only a version bump and rebuild — no manual diff-and-patch across the theme files.
- **Zero runtime overhead.** The bundle contains only the resolved hex values; consuming apps do not need to install `@carbon/themes` or any Carbon package.
- **Auditability.** `tokens.ts` expresses the mapping relationship explicitly (`$background` → `backgroundColor`, etc.), making it straightforward to verify correctness against the Carbon spec.
- **Four-theme coverage.** White, G10, G90, and G100 are all derived from the same mapping, ensuring consistency across modes.

## Consequences

- `@carbon/themes` is a **dev dependency**, not a runtime dependency.
- When Carbon ships a token rename or value change, the theme package must be rebuilt and re-released.
- The token map (`tokens.ts`) is the authoritative record of which Carbon tokens are used and how they map to ECharts keys. New ECharts theme keys should be added there first.
- Hardcoding hex values anywhere in preset files or theme objects is prohibited; all colors must trace back to `tokens.ts` or `palettes.ts`.
