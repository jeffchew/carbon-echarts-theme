# ADR 0006 — Netlify for showcase site deployment

**Status:** Accepted  
**Date:** July 2026

---

## Context

The showcase site (`packages/site`) needs to be deployed on every merge to `main` and on every PR for preview. The original plan specified GitHub Pages.

## Decision

Deploy to Netlify. GitHub Pages was replaced during implementation.

## Rationale

- **PR preview deploys.** Netlify provides per-PR preview URLs out of the box with no additional workflow configuration. This is essential for visual review of chart changes before merging.
- **Simpler configuration.** `netlify.toml` in the repo root is the single deployment config. GitHub Pages requires a dedicated deploy workflow and either a `gh-pages` branch or an Actions artifact upload.
- **SPA routing.** Netlify handles client-side routing redirects with a single `[[redirects]]` rule. GitHub Pages requires a custom 404-page workaround.
- **Build caching.** Netlify's build cache reduces deploy times for the Vite site.

## Consequences

- `netlify.toml` is the authoritative deployment config.
- PR preview URLs are automatically posted as GitHub deployment statuses.
- Production deploys are triggered by non-RC tag pushes via the `deploy-site.yml` workflow.
- The original plan reference to `charts.carbondesignsystem.com/echarts` as the deployment target should be treated as aspirational; the current live URL is managed through the Netlify project settings.
