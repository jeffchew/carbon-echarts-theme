# Documentation

---

## Architecture

Reference material for how the project is structured and why.

| Doc                                                                | Contents                                                                                         |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------ |
| [`architecture/overview.md`](./architecture/overview.md)           | Monorepo structure, packages, public API, dependency graph, architecture rules                   |
| [`architecture/site.md`](./architecture/site.md)                   | Site stack, routes, components, IBM platform requirements, deployment                            |
| [`architecture/token-mapping.md`](./architecture/token-mapping.md) | Carbon token → ECharts key mapping, IBM data-vis palettes, animation defaults, known limitations |

---

## ADRs

Architectural Decision Records — why key choices were made.

| ADR                                                       | Decision                                                                                |
| --------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| [0001](./adr/0001-echarts-as-visualisation-foundation.md) | Apache ECharts as the IBM / Carbon visualisation foundation                             |
| [0002](./adr/0002-no-framework-adapters.md)               | No framework adapters shipped with the theme package                                    |
| [0003](./adr/0003-token-derivation-at-build-time.md)      | Design tokens derived from `@carbon/themes` at build time                               |
| [0004](./adr/0004-pnpm-workspaces-monorepo.md)            | pnpm workspaces monorepo                                                                |
| [0005](./adr/0005-release-please-versioning.md)           | Release Please for versioning and changelog generation                                  |
| [0006](./adr/0006-netlify-deployment.md)                  | Netlify for showcase site deployment                                                    |
| [0007](./adr/0007-codemods-co-located.md)                 | Codemods co-located in this repository                                                  |
| [0008](./adr/0008-toolbar-separate-package.md)            | Toolbar as a separate published package                                                 |
| [0009](./adr/0009-motion-package-boundary.md)             | Motion: keep the theme a plain object; engine in a separate package                     |
| [0010](./adr/0010-motion-token-source-of-truth.md)        | Motion token source of truth: consume from Carbon v12's motion surface API _(Proposed)_ |

---

## Plan

Current status and outstanding work.

| Doc                                                    | Contents                                                                           |
| ------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| [`plan/current-state.md`](./plan/current-state.md)     | What is complete, what is incomplete — point-in-time snapshot                      |
| [`plan/roadmap.md`](./plan/roadmap.md)                 | Prioritised work queue, code quality work streams, known limitations               |
| [`plan/motion-proposal.md`](./plan/motion-proposal.md) | Draft: analysis of Carbon Next motion explorations and proposal for ECharts motion |

---

## Legacy

`docs/legacy/` contains the original numbered phase documents from initial planning. These are retained for historical reference; the architecture and plan docs above supersede them.
