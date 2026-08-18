# Evidence Translation Tools

A set of browser-based tools for making evidence more interpretable, built for
Evidence Action. Everything lives in a single self-contained `index.html` — no
build step, no dependencies, fonts inlined — so it runs offline and deploys as a
static page.

## Live site

**https://tcolinrichardson.github.io/evidence-translation/**

## What's inside

Nine tools, grouped by theme:

- **Evidence** — Effect Size, Scorecard, Evidence Chain
- **Scale** — Population Reached, Scale Sensitivity, Comparative Scale
- **Cost-effectiveness** — DALY Source, CE Sensitivity, Switching Value

Each tool renders a branded figure you can customize with the input cells and
export as a PNG, and most also emit copy-pasteable R code for the same chart.

## Files

- `index.html` — the whole app in one file (served at the site root)
- `intervention-facts-box-NOTES.md` — design and implementation notes
- `.nojekyll` — tells GitHub Pages to serve the files as-is

## Deployment

The site is published with GitHub Pages using **Deploy from a branch**
(`main` / root). Every push to `main` redeploys automatically, usually within a
minute. There is no build step — the page is served exactly as committed.
