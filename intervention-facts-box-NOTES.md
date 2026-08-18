# Intervention Facts Box — developer notes & handoff

_Last updated: 2026-08-18_

> **Keep this doc current.** Update it as part of _every_ meaningful change — a
> code change and its notes update ship together, not as an afterthought.

A single-file, dependency-free web app that Evidence Action uses to turn evidence
into on-brand figures. Each tool renders a figure on screen and exports it as a
PNG that gets pasted into Google Docs / decks.

**Published artifact:** https://claude.ai/code/artifact/dad090ea-7122-44ae-b664-7c1dbdd0c51a

---

## 1. What this project _is_ (read this first)

Two facts drive every design decision:

1. **It runs desktop-only in a browser.** There is no mobile use case.
2. **The only deliverable is a PNG** pasted into a document.

So this is a **figure factory**, not a responsive embeddable widget. That framing
is why the layout is fixed-width (no reflow) and why "what you see on screen is
exactly what exports" (WYSIWYG) is the central quality bar. If you're tempted to
add responsive breakpoints or make the on-screen view diverge from the export,
re-read those two facts first.

The app presents a tab bar grouped into three columns — **Evidence**,
**Scale**, **Cost-Effectiveness** — routed by URL hash (`#effect`, `#daly`,
etc.; the `NAV` map at the bottom of the script defines the layout). **All nine
tabs are built** — eight distinct tool factories, with the tornado tool
instantiated **twice** (CE Sensitivity + Scale Sensitivity). No
`createPlaceholderTool` stubs remain registered in `TOOLS[]`.

The table below follows the on-screen column order. "Slug/id" is the value
passed to `makeToolSection(...)` (also the URL hash); where the export filename
slug differs it's noted separately.

| Column | Tab | Factory | Slug/id | Figure |
|---|---|---|---|---|
| Evidence | Effect Size | `createEffectTool` | `effect` | metrics + sampling-distribution curve (+ optional icon array) |
| Evidence | Scorecard | `createScorecardTool` | `scorecard` | H/M/L heat matrix + rating-mix rail |
| Evidence | Evidence Chain | `createEvidenceChainTool` | `outcomes` | node→node evidence-link diagram (self-owned `paintEvidenceChain`) |
| Scale | Population Reached | `createPopulationTool` | `population` | reach cascade/waterfall (`paintPopulation`) |
| Scale | Scale Sensitivity | `createTornadoTool('scalesens')` | `scalesens` | tornado reframed for total DALYs averted (`paintTornado`) |
| Scale | Comparative Scale | `createComparativeScaleTool` | `compscale` (slug `comparative-scale`) | DALYs/year ranking bar chart (`paintScale`) |
| Cost-Effectiveness | DALY Source | `createDalyTool` | `daly` | side-by-side: dark rail (Total DALYs + legend) beside the stacked bar |
| Cost-Effectiveness | CE Sensitivity | `createTornadoTool` | `tornado` | data table + tornado plot (`paintTornado`) |
| Cost-Effectiveness | Switching Value | `createSwitchingTool` | `explainer` | switching-value tornado, clipped ±cutoff (`paintSwitching`) |

Three tabs (**Effect Size**, **DALY Source**, **Population Reached**) also embed
a live **"Example Report"** figure beneath the download button — see §5.

---

## 2. The file (single, self-contained)

There is now **one** HTML file:

- **`index.html`** — the whole app in one file, served at the GitHub Pages root
  URL. The two brand fonts (JetBrains Mono, Roboto Condensed) are **inlined as
  base64 `@font-face`**, so it is fully self-contained under the artifact sandbox
  CSP and renders correctly offline.

Edit this file directly — JS lives in its single `<script>` block; styles in its
`<style>` block; body markup below. There is no build step and no second copy to
keep in sync.

> **History:** a `…-template.html` twin used to exist (same page, but pulling the
> fonts from Google Fonts via a `<link>` instead of inlining them). Its `<script>`
> was kept byte-identical to the artifact's and changes were mirrored across both.
> That template was **deleted** — the base64-inlined artifact was the only file
> needed to run/publish, so the sync burden was dropped. If you ever want a
> lighter "source" copy again, regenerate one by swapping the inlined `@font-face`
> block for the Google Fonts `<link>` (`fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;500;700&family=Roboto+Condensed:wght@300;400;500;700&display=swap`).

---

## 3. Architecture

### Shell
`makeToolSection(id)` builds a `<section class="tool" data-tool="id">` containing
a `.card` (the figure) and a `.settings` block (controls + editor + a
"Download report (PNG)" button + an optional R-code `<details>`). The Shell
show/hides sections by hash. `TOOLS[]` lists the factory instances; `NAV`
defines the tab layout.

**Per-tab "Goal:" note.** Each section gets a `.goalbox` prepended above the
`.card` (`goalBoxHTML(id)`), a light-blue on-screen note with an uppercase
magenta "Goal:" label (an always-visible `<div>` — a `.goal-title` span + a
`.goal-text` span). Text comes from the `GOALS` map keyed by tool id — real
per-tab copy now (no longer lorem); a `"\n\n"` in an entry starts a new paragraph
(the "Example:" blocks), rendered via `.goalbox .goal-text{white-space:pre-line}`
(`esc()` escapes only `&<>`, so newlines/quotes/apostrophes pass through).
`goalHTML` bolds the **first sentence of the first paragraph** (`markFirstSentence`
splits it at the first `.!?`) plus any **`**phrase**`** explicitly marked in the
GOALS copy: each paragraph is tokenized on `**` and odd segments become `<b>`
(text parts still esc'd, so markers must be balanced). Later paragraphs (e.g.
"Example:") are not auto-bolded but still honor their own `**…**`. It is injected
in
`makeToolSection` and `createPlaceholderTool`; the Effect tool uses a static
section, so it inserts its own via `insertAdjacentHTML('afterbegin', …)`. The
goal box is **chrome only —
never in the PNG export** (the painter draws a fresh canvas from `cfg`, never a
DOM screenshot). On the scorecard (whose section breaks out to 1220px) a
`.tool[data-tool="scorecard"] .goalbox` rule pins it to the 720px column like
`.settings`.

### The figure model — one painter, two consumers (the WYSIWYG core)
Every tool has a **canvas painter** that draws the entire figure — header bar,
metric tiles, chart, legend, narrative — at a given scale and returns a
`<canvas>`. That single painter feeds **both**:

- **Export:** paint at **scale 3**, `canvas.toBlob` → download the PNG.
- **On-screen preview:** paint at **scale 2** (debounced ~50ms), then set an
  `<img>`'s `src` to `canvas.toDataURL()`. The card is literally an `<img>` of
  the exported canvas.

Because the preview and the export come from the same paint call, they are
**pixel-identical by construction** — they cannot drift. This replaced the old
design where each tool had a separate on-screen HTML/SVG render and a separate
export painter that silently diverged (e.g. the DALY export used to drop the
legend; the Effect export dropped the icons).

```
paint(cfg/​state, scale) ──► <canvas>
        ├── scale 3 ──► toBlob ──► download  (PNG export)
        └── scale 2 ──► toDataURL ──► <img class="figimg">  (on-screen preview)
```

### Why canvas, not SVG, for the whole figure
Canvas `fillText` renders with the document's **loaded brand fonts**. An SVG
rasterized via `Image`/data-URL is an isolated document that does **not** get the
page's `@font-face` fonts, so its text falls back to generic stacks. The figure
chrome (title, big metric numbers) must be in the brand fonts, so it is
hand-drawn on canvas. (The chart *interior* is still an SVG that gets rasterized
into the canvas — its text now renders in the brand fonts too, because
`svgFontStyle()` embeds the page's `@font-face` rules inside each chart SVG; see
§9's resolved TODO.)

### Key shared functions (names are stable; line numbers drift)
- `paintReport(cfg, scale)` — shared painter for **Tornado** (the vertical
  header → metric grid → chart → optional legend → narrative layout). Returns
  `{canvas, W, H}`. `W` is 720.
- `paintDaly(cfg, scale)` — **DALY's own** painter (same `{canvas,W,H}` contract):
  the bespoke **side-by-side** layout — Ink header, a dark left rail (Total-DALYs
  panel above a legend panel) beside the stacked-bar chart, then the narrative.
  DALY uses this instead of `paintReport`; the chart is the tallest element and
  sets the figure height, and the legend panel stretches to the chart's bottom.
  Rail/chart split: `railW=236` (the Total-DALYs + legend panels) leaves the chart
  box the remaining width — the rail was widened from 190 to reclaim space freed by
  tightening the bars (below).
- **Bar tightening** (in `createDalyTool` → `chartSVG`): bars fill `band*0.84`
  (cap 280) with side margins `left:72 / right:14`, so the three horizontal gaps —
  y-axis→bar, bar→bar, bar→edge — are ~half their earlier width; the reclaimed
  width goes to the rail panels.
- `renderReportPNG(cfg)` / `previewReport(imgEl, cfg)` — thin export / debounced-
  preview wrappers. **Both dispatch on `cfg.kind`** through one ternary (checked in
  this order): `'switching'` → `paintSwitching`, `'population'` → `paintPopulation`,
  `'daly'` → `paintDaly`, `'scale'` → `paintScale`, `'tornado'` → `paintTornado`,
  otherwise → `paintReport`. So DALY, both tornados, Population, Switching Value,
  Comparative Scale, and any plain report share the same two wrappers but paint with
  different painters.
- **Self-owned tools bypass that dispatch.** `paintEffect`/`paintScorecard` and
  `paintEvidenceChain` each have their **own** `preview*`/`download*` wrappers and
  are **not** routed through `renderReportPNG`/`previewReport`. Effect, Scorecard,
  and Evidence Chain therefore never set a `cfg.kind`.
- `paintEffect(m, scale)` / `paintScorecard(scale)` — the Effect and Scorecard
  painters (bespoke layouts, same `{canvas,W,H}` contract). Each has its own
  `download*` and `preview*` wrappers.
- `narrativeLines(html, mc, sizePx, sans, maxW)` — parses a narrative **HTML**
  string, keeping `<b>`/`<strong>` runs, into wrapped lines of
  `{text,bold,space}` tokens. `drawNarrative(ctx, lines, x, y, sizePx, sans, ink)`
  draws them, rendering bold tokens in the 700-weight face. Both painters
  (`paintReport`, `paintEffect`) use this so the light-blue band keeps its bold
  emphasis instead of flattening to plain text. **`<b>`-only** — see §10.
- `makeToolSection(id)`, `wireToolChrome(root, getCfg, getR)`, `dlBlob`,
  `BRAND`, `niceTicks`.

### The `cfg` contract for `paintReport` (Tornado; DALY uses `paintDaly` — see §5)
```js
{
  title,                 // header text
  columns,               // [{label, accent?, vals:[big, sub1, sub2...]}]
  chartSVG,              // inner SVG markup (no <svg> wrapper) or '' 
  chartLabel,            // small uppercase label above the chart
  narrative,             // HTML string (may contain <b> runs) — parsed by
                         // narrativeLines/drawNarrative; NOT flattened to plain text
  slug,                  // filename slug
  chartAspect,           // chart height/720 (DALY uses CHART_H/720 = 560/720)
  legend,                // optional [{color, label, value}] → legend band
  legendLabel,           // optional heading above the legend band
}
```
Each tool builds this in a `reportCfg()` function used by both `redraw()`
(preview) and `wireToolChrome` (export), so there is one source of truth.

---

## 4. Dimensions & the standardized figure spec

- **Chart tools (Effect, Tornado, DALY):** authored at **720px** wide (fits a
  Letter page with ~0.5" margins). Export at **3×** → **2160px** wide PNG.
- **Scorecard:** fixed **1220px** wide (its layout needs the width). Export at
  **3×** → **3660px** wide PNG.
- Preview is painted at **2×** and displayed at logical width (720 or 1220) via
  an inline `img.style.width`, so it's crisp on screen.
- Heights vary by content; the painter computes `H` from the sections present.

### Fixed desktop layout (no reflow)
- `.wrap` is a fixed **720px** column (`.wrap{width:720px;margin:0 auto;}`).
- The scorecard breaks out of that column to a fixed **1220px**, centered via
  `left:50%; transform:translateX(-50%)`.
- The old `@media(max-width:760px)` block and `min(1180px,96vw)` sizing were
  **removed**. At a narrow window the layout does not reflow (it just overflows —
  intended for a desktop tool).
- `.figcard{border:0;background:none;padding:0;}` and
  `.figimg{display:block;width:720px;max-width:100%;height:auto;}` style the
  figure image; the painter draws the card border itself, so the container has
  none.

### Notable spec details inside the painter
- `fitPx(text, weight, base, maxW, floor)` — **shrink-to-fit** for long metric
  values so a value like "Mortality (deaths averted)" no longer overflows into
  the next column (shrinks from 30px down to a 13px floor).
- Legend band: color swatch + label (Roboto Condensed) + right-aligned value
  (JetBrains Mono), wraps within the content width; height is measured up front
  so the canvas is tall enough.

---

## 5. Per-tool notes

### Effect (`createEffectTool`)
- Bespoke metric layout: the "Absolute reduction" column **auto-sizes**, and the
  big value can split into number + smaller unit (`11.5 pp`). Kept intentionally.
- Painter `paintEffect(m, scale)` derives its narrative and chart label directly
  (does **not** read them back out of the DOM). The narrative HTML is passed
  through `narrativeLines`/`drawNarrative`, so its `<b>` runs survive into the PNG.
- **Metric tiles (`reportCols`) carry emphasis the canvas must reproduce:** the
  accent column (RRR/MD) draws magenta; the ARR value splits into a big number +
  smaller `pp` unit; and the **NNT tile** has two special behaviors — `tightLabel`
  shrinks "Number needed to treat" to a single line, and `flagSub` draws a
  **magenta-bold** discontinuous-interval subline (`NNTB x – ∞ – NNTH y`) when the
  ARR CI crosses zero, matching the hint text below the figure.
- **Icon-array legend** draws the count in the **bold** face and the label plain
  (`700 14px` + `14px`), mirroring the old `<b>count</b> label` HTML. Value and
  label are stored separately in `icData.legs` (`{cls, v, label}`) so wrap-math and
  drawing use matching fonts.
- **Icon array (waffle of 100):** drawn **inside the figure** and exported **only
  when the "Icon array" checkbox is on** (condition:
  `state.showIcons && isRatio() && m.pSQ!=null`). Colors:
  `is-prevented`=`#e600a0`, `is-develop`=`#20253a`, `is-well`=`#d9dbe4`. Data
  comes from `iconGroups(m)` + `alloc100(fill)` (exact 100-square largest-
  remainder allocation).
- On-screen render fn is `draw()`; it calls `previewEffect(m)`.
- **Metric row balance:** the metric tiles are **center-justified**, and the row stays
  balanced when only two metrics are shown (so unchecking RRR/NNT no longer leaves the top
  line lopsided). The big metric numbers are **vertically centered** in their cells, and the
  whitespace above the sampling-distribution curve was tightened.
- **The "Confidence intervals (95%)" toggle was removed** — CIs always render.
- **Example Report figure.** Below the download button sits a live-rendered
  **"Example Report: Maternal Mortality — High Coverage Participatory Women's Groups"**
  figure, painted from a hard-coded example `m` through the same `paintEffect` path as the
  real figure (so it can't drift from an export) — see the shared example-figure note at the
  end of this section.

### DALY (`createDalyTool`)
- Stacked bars, one bar per subpopulation, segments per source. Source colors
  come from a dark→light green ramp (`srcColor(si)`), so any number of sources
  stays on-brand. `CHART_H = 560`; the chart SVG is authored at `720×560`.
- **Bespoke side-by-side layout via `paintDaly`** (not the shared `paintReport`).
  Its `reportCfg()` returns `{kind:'daly', title, total, totalLabel, chartSVG,
  narrative, legend, legendLabel}` — note the shape differs from the `paintReport`
  cfg (no `columns`/`chartLabel`/`chartAspect`); `kind:'daly'` routes it to
  `paintDaly` in the shared wrappers. Layout: dark left rail (Total-DALYs panel +
  legend panel) beside the stacked bar, narrative below. The "Legend box" toggle
  (`state.showLegend`) drops the legend panel, leaving just the Total-DALYs panel.
- This layout was **restored** (the redesign had briefly replaced it with the
  standardized `paintReport` metric-tiles layout). Restoring it on canvas keeps
  WYSIWYG — preview == export — unlike the pre-redesign version, whose nice
  side-by-side view was on-screen HTML only while its PNG export used a different
  layout. The two extra tiles the redesign added ("Largest source",
  "Subpopulations") were intentionally dropped; `metricsCols`/`chartLabel` were
  removed with them.
- Editor supports add/remove/rename sources and subpopulations; rows are ordered
  top→bottom to match the bar stack + legend.
- **Example Report figures.** Below the download button the DALY tab embeds **two**
  live-rendered example reports — **"Example Report: MMS"** and **"Example Report: IV Iron"**
  (x-axis labeled "DALYS From MMS") — captioned in black at an enlarged size. Both are painted
  from hard-coded example cfgs through the same `paintDaly` path as the real figure (shared
  example-figure note at the end of this section).

### Tornado (`createTornadoTool`) — bespoke `paintTornado`
- Rebuilt to the design handoff (`c76853b5-README.md`): a **one-way sensitivity
  diagram** = compact data table (Parameter / Low / Base / High) row-aligned with a
  **tornado plot**, on the app house ink header (`drawBrandMark` + magenta eyebrow +
  white two-weight title rendered via `narrativeLines`/`drawNarrative`).
- **Bespoke canvas painter `paintTornado(cfg, scale)`** (like `paintDaly`), returning
  `{canvas,W,H}`, **W = 720**. Routed via `cfg.kind:'tornado'` — both `previewReport`
  and `renderReportPNG` dispatch `daly→paintDaly, tornado→paintTornado, else paintReport`.
- **Data contract:** `params:[{label, low, base, high, lowOut, highOut}]` where
  low/base/high are **display strings** (units baked in) shown in the table, and
  `lowOut/highOut` are the **model outputs** ($/DALY) that drive the bars; plus
  `base` (spine), `ax` (axis max), `unit`, `eyebrow`, `title`. Rows **sorted by span
  descending** (the tornado shape).
- **Color-by-input (important):** blue `#5e93fb` = outcome at the **low** input,
  magenta `#e600a0` = outcome at the **high** input — segments colored by their source
  input, split at the base spine, so magenta/blue land on either side depending on
  direction. Zebra band `#eceef2` on odd rows spans table+plot; LOW chip blue / BASE
  ghost / HIGH chip magenta; sharp corners, no shadows.
- Editor is ratings-of-inputs style: global fields (title/eyebrow/base/axis-max/unit) +
  a param table (label + low/base/high strings + `$@Low`/`$@High` numbers) + add/remove.
  Show/hide controls dropped (design is fixed). `rCode()` emits a ggplot tornado from the
  new `lowOut/highOut` data.
- **Two instances — CE Sensitivity + Scale Sensitivity.** `createTornadoTool(id)` builds
  the tab **twice**: the default `id:'tornado'` is the **CE Sensitivity** tab, and
  `createTornadoTool('scalesens')` is the **Scale Sensitivity** tab. A single flag
  `scale=(id==='scalesens')` drives every variant difference: Scale reframes the tornado
  around **total DALYs averted** (title "Drivers of Total DALYs Averted", `unit:'DALYs'`,
  `base 1319`, `ax 3500`, and the value-column headers switch from `$ @Low/@High` to
  `DALY @Low/@High` via `outWord`), and seeds the **cataract** dataset instead of the CE
  default set. Both instances share the same `paintTornado` painter and `reportCfg` shape;
  the tab labels ("CE Sensitivity" / "Scale Sensitivity") live in `NAV`.
- **Axis gridlines.** Light vertical gridlines are drawn at the `niceTicks(0,AX,7)` tick
  positions, painted **before** the spine and bars so they run beneath them.
- **Cumulative "all parameters" row (own auto-fit axis).** Below the main chart, both
  instances show a single **manual, editable** `Cumulative (all parameters)` row — the
  combined effect of setting every parameter to its low vs. its high. Because that swings
  far more than any one parameter, it gets its **own axis** below the main one, auto-fitted
  with `niceTicks(0, max(cumHi,BASE,1), 7)` (`CUMAX` extended if the high output exceeds the
  top tick) and its own spine/gridlines/ticks via a second scale `xOf2(v)=plotX0+v/CUMAX*plotW`.
  The values are **not computed** — they're a seeded, editable row on `state.cumulative`
  (`{label, base, lowOut, highOut}`; Scale defaults 183 → 17582, CE **80 → 840**), edited in a
  "Cumulative row (own axis)" editor block (just **Label + @Low + @High**) bound via `data-cum`
  inputs. Unlike the parameter rows, the cumulative row is drawn as a **single free-flowing
  phrase** (not bound to the Low/Base/High table columns): `"<label>: <low> – <high> [unit]"`,
  vertically centered on the bar, shrink-to-fit down to a 9px floor. The **cumulative plot begins
  right after the phrase** (`cumPlotX0 = tableX + phraseWidth + 18`; its own `xOf2` / spine /
  gridlines / axis span `cumPlotX0…plotR`), so the bar fills the space rather than starting at the
  main chart's `plotX0` — a short phrase (CE) widens the bar, a long one (Scale) lands near the
  main plot. `cumCell(v)` prefixes `$` when the
  `unit` contains `$`, and the unit noun is appended as a suffix only for non-`$` units — so CE
  reads `Cumulative (all parameters): $80 – $840` and Scale reads
  `Cumulative (all parameters): 183 – 17,582 DALYs`. The range always tracks the `@Low`/`@High`
  inputs. `reportCfg()` passes `cumulative` only when both outputs are finite — clear either and
  the section hides (`hasCum` false), leaving just the main chart. The R-code export stays
  params-only.
- **Caption inline with the header row.** The `unit · base case = BASE` caption sits on the
  **table header row** (vertically centered with the PARAMETER label + Low/Base/High chips),
  centered over the plot area — not beneath the main axis, so the cumulative section can own the
  bottom.

### Population Reached (`createPopulationTool`) — bespoke `paintPopulation`
- Reach **cascade / waterfall** from the design handoff (`0be26c6d-README.md`). Re-authored in
  the app house style (ink header + `drawBrandMark` + magenta eyebrow + two-weight title) at
  **W = 720**, not the README's fixed 900×600. Default seed is the reading-glasses model: a
  1,000-person pool ("Total presbyopic") shrinks across **9 steps** to "Using glasses (Yr 2)"
  = 170 (**17%**). The number of steps is not fixed — everything derives from the array length.
- **Bespoke painter `paintPopulation(cfg, scale)`** → `{canvas,W,H}`. Routed via
  `cfg.kind:'population'` — the branch is **first** in both `previewReport`/`renderReportPNG`
  ternaries (`population→paintPopulation, daly→paintDaly, tornado→paintTornado, else paintReport`).
  Canvas height is computed from the step count (`plotTop + n*rowPitch + bottomPad`), so it grows
  with more steps.
- **Data contract (cfg):** `startPool` (number), `steps:[{label, value, note}]`, plus `eyebrow`
  and `title`. `note` holds **only the multiplier number** (e.g. `'60%'`); the fixed `'×'` is
  supplied by the painter and editor, not stored — so only the number is user-editable. The first
  (starting-pool) row is not a multiplier, so its `note` is a plain label (`'Starting pool'`) with
  no `×`. Derived per step: `prev = i===0?startPool:steps[i-1].value`, `loss = prev-value`,
  `detail = i===0?note:'× '+note+' (-'+loss+')'`, final step painted **ink** not magenta, callout
  `pct = last.value/startPool*100`.
- **Fixed vs editable branding (in `createPopulationTool`):** the eyebrow and the title up to the
  colon are **hard-coded, non-editable** constants — `EYEBROW='Scale Cascade'` and
  `TITLE_HEAD='Population reached out of <b>1,000 people</b>:'`. Only the portion after the colon
  is user-editable: `state.titleTail` (default `'reading glasses'`). `reportCfg()` returns
  `eyebrow:EYEBROW` and `title:TITLE_HEAD+' '+esc(state.titleTail)` (tail escaped so it can't inject
  markup into the bold-run parser).
- **Layout:** left label column (label + muted detail line) | plot with 5 gridlines
  + ticks `0/250/500/750/1,000`, per row a grey **loss track** (width = prev) and a magenta/ink
  **value bar** (white right-aligned value), plus a **loss label** only when `loss ≥ 100`
  (no dashed connectors — removed). The reach **callout** (big magenta % + "of the Target
  Population Reached", semi-opaque white ground, 4px magenta left rule) is parked in the **lower-
  right** corner (`numBase ≈ plotTop+8.2*rowPitch`, `boxX = plotX0+plotW*0.50`); the magenta rule
  is sized to wrap the text block exactly so the number+subtitle read centered on it.
- **People is computed, not entered.** `recompute()` sets `steps[0].value = startPool` and each
  later `steps[i].value = round(steps[i-1].value × pctFrac(note))` (`pctFrac('60%')=0.60`). It runs
  at mount and on every edit to the **starting pool** or any **note %**, plus after
  add/remove/reorder. The People cells are `readonly` inputs (`data-vidx`); note/pool edits update
  them in place via `refreshValues()` (no `renderRows()`, so the field being typed keeps focus),
  while structural changes go through `recompute();renderRows();redraw()`.
- Editor: **fixed** eyebrow (none — permanent) and a locked title prefix shown as static text
  beside a single editable **title-tail** input; a **starting pool** field (the base of the
  cascade); and a step table (name / people[computed] / note) + add/remove, with
  **drag-to-reorder** (grip handle per row → HTML5 DnD reorders `state.steps` then
  `recompute();renderRows();redraw()`; the row is made `draggable` only on handle mousedown so the
  inputs stay editable). `rCode()` emits a ggplot2 waterfall (grey `geom_col` prev track under a
  magenta/ink value `geom_col` + white value labels); its title tracks `titleTail`.
- **Example Report figures.** Below the download button the Population tab embeds **two**
  live-rendered example cascades — captioned **"Example Report: Conservative"** and
  **"Example Report: Generous"** (a reading-glasses / cataract-style cascade) — painted through
  the same `paintPopulation(cfg,2)` → `toDataURL` path as the live figure (shared example-figure
  note at the end of this section).

### Switching Value (`createSwitchingTool`) — bespoke `paintSwitching`
- **Threshold analysis, Graph 1** from `a6c16abe-thresholdanalysisspec.md`: a switching-value
  tornado of each parameter's signed % change to its switching value. **Display tool** — the % are
  typed, not computed (no NMB root-finding). App house style, W=720. Internal id is **`explainer`**
  (the tab was renamed "Switching Value" earlier; id kept so NAV + the GOALS slot match).
- **Bespoke painter `paintSwitching(cfg, scale)`** → `{canvas,W,H}`. Routed via `cfg.kind:'switching'`
  — the branch is **first** in both `previewReport`/`renderReportPNG` ternaries. Height derives from
  the row count.
- **Data contract:** `{eyebrow, title, cutoff, params:[{label, pct, never}]}`. `pct` is a signed
  percent; `never:true` → robust ("never flips", no bar). Rows sort by `abs(pct)` ascending (most
  fragile on top); `never` rows sink to the bottom.
- **Fixed branding:** eyebrow and the title prefix are hard-coded, non-editable constants —
  `EYEBROW='How far off would this parameter alone need to be to change the decision?'` and
  `TITLE_HEAD='Switching Values:'`. Only the portion after the colon is editable
  (`state.titleTail`, default `'Program Name'`); `reportCfg()` returns `eyebrow:EYEBROW` and
  `title:TITLE_HEAD+' '+esc(titleTail)`. The editor shows the fixed prefix as static text beside the
  tail input (same pattern as the Population tool).
- **Encoding:** symmetric x-axis `−cutoff…+cutoff` (default ±150, editable), 5 **bold dark (ink)**
  ticks, emphasized zero line; axis label "% change required to become non-CE". One diverging bar per
  row using the **CE Sensitivity scheme** — **blue `#5e93fb` = flips down** (`pct<0`), **magenta
  `#e600a0` (`BRAND.mag`) = flips up** (`pct>0`); bars beyond `±cutoff` are clamped to the edge,
  capped with an **arrowhead**, and the true % printed just past the axis in the direction color;
  in-range bars print the % just past the bar end. No legend and no bottom caption (removed). Per the
  user, **no** plausible-range/in-range distinction (the spec's dashed encoding is omitted).
- Editor: global Title / Eyebrow / **Cutoff (±%)** + a `.etable` of rows (Parameter / % change /
  Never-flips checkbox / remove) + add. Toggling **Never** disables that row's % input. `rCode()`
  emits a ggplot2 switching-value tornado (`geom_col` for unclipped, `geom_segment`+closed `arrow`
  for clipped, coral/blue scales, `coord_cartesian(clip="off")`).

### Scorecard (`createScorecardTool`)
- 13 criteria in 3 fixed tiers (tier names locked: `PRIMARY` /
  `SECONDARY · FEASIBILITY` / `TERTIARY`), each rated **H/M/L**, with a
  rating-mix rail summarizing the tally. Colors: **H `#63be7b`, M `#ffeb84`,
  L `#f8696b`**.
- Painter `paintScorecard(scale)` hand-draws everything (title with skewed
  magenta slash, tier blocks, tiles, rail). No R-code for this tool (the
  `.rcode` block is removed in its `mount`).
- Counts/percent derive from the live total N, not hard-coded to 13.

### Evidence Chain (`createEvidenceChainTool`) — self-owned `paintEvidenceChain`
- **Renamed from the old "Outcomes Table" tab** — the section id / URL hash is still
  **`outcomes`** (kept so `NAV` and the `GOALS` slot match), but the tab now shows a
  **fully dynamic node→node evidence-link diagram**: a vertical chain of outcome **nodes**,
  each link annotated by one or more **evidence cards** on a rail to the right.
- **Self-owned tool** (like Scorecard): it has its **own** `paintEvidenceChain(scale)` painter
  plus its own preview/download wiring and does **not** route through the shared
  `previewReport`/`renderReportPNG` dispatch (no `cfg.kind`). The `.rcode` block is removed in
  its `mount` (no R-code for this tool).
- **Data model on `state`:** `intervention` (label), `nodes:[{label}]`, and
  `cards:[{from,to,source,estimate,meta,confidence}]` where `from`/`to` are **node indices**
  and `confidence` is `0–3`. Everything is editable — add/remove/rename nodes and cards.
- **Confidence drives color** (`accentOf`): **High (3) = magenta**, **Medium (2) = ink**,
  **Low (1) / None (0) = grey**; the card header text flips to white on the magenta/ink accents
  (`headTextOf`). Each card carries a **3-segment confidence meter** and an **auto caption**
  (`confCaption` → "High/Medium/Low/No confidence"). Card background is a light cyan (`#d5f4f7`).
- **Overlap handling:** when two arrows would originate and terminate on the **same node**, they
  are separated so they don't overlap (commit `9974bf0`).

### Comparative Scale (`createComparativeScaleTool`) — shared `paintScale`
- Ranks a program's **DALYs averted per year** against Evidence Action's existing
  program-geographies. Section id **`compscale`**; the export slug is **`comparative-scale`**.
  Fixed, non-editable title: **"DALYs Per Year Compared to Existing EvAc Program Geographies"**.
- **SVG chart, not a bespoke canvas painter.** `chartSVG()` builds the ranked bar chart as SVG
  (authored **872×660**) and `reportCfg()` returns `{kind:'scale', title, chartSVG, slug}`, which
  the shared wrappers rasterize via **`paintScale`** (new `cfg.kind:'scale'` branch).
- **Data:** a fixed array of **21** reference DALY/year values (`FIXED`) plus **one editable
  magenta user bar** — `state.intervention` (default "Example Program") and `state.dalys`
  (default 15,000). `compute()` appends the user value, sorts ascending (on a tie the user bar
  sorts **after** the equal fixed value so it ranks higher), and derives its **rank**
  (`arr.length - userIdx`). Two reference bars are labeled with curved leader lines:
  **SFS Cameroon (~92,000)** and **SFS Liberia (~5,000)**.
- **Callout:** a two-line user-bar callout naming the program and its DALYs/year, positioned in
  the lower area (nudged down so it clears the bars); the intro caption is a two-sentence block
  with a forced line break. Bars: user = magenta, references = ink; ascending rank labels
  (`01` is the tallest, on the right).

### Example Report figures (shared pattern — Effect, DALY, Population)
Three tabs embed one or more **"Example Report: …"** figures directly below the
"Download report (PNG)" button. Each is a plain `<img>` whose `src` is set from a
**hard-coded example `cfg`/`m`** passed through the **same `paint*` function** the tool uses for
its real preview/export (`paintEffect` / `paintDaly` / `paintPopulation` at scale 2 →
`canvas.toDataURL()`). They are **rendered live**, not baked-in PNG assets — so an example can
never silently drift from what the tool actually exports. Captions are static text under each
image. These example figures are **on-screen chrome only**; they are not part of any export.

---

## 6. Verify / test workflow

Everything is offline and dependency-free. Global `playwright@1.56.1` is
available; Chromium lives at `/opt/pw-browsers/chromium-1194/chrome-linux/chrome`
(do **not** run `playwright install`).

1. **Syntax check** — extract the largest `<script>` block and `new Function(it)`
   (or `node --check` on the extracted JS):
   ```js
   const big=s=>{const m=s.match(/<script>([\s\S]*?)<\/script>/g);let b='',l=0;
     for(const x of m){const y=x.replace(/^<script>/,'').replace(/<\/script>$/,'');
       if(y.length>l){l=y.length;b=y;}}return b;};
   ```
2. **Headless render/export test** — load the **artifact** file (it has inlined
   fonts) via `file://`, for each tool: confirm the `<img class="figimg">`
   preview has a `data:image/png` src, capture the exported PNG via a
   `download` event, and read its dimensions from the PNG IHDR
   (`buf.readUInt32BE(16)` = width, `readUInt32BE(20)` = height). Expected:
   charts 2160×H, scorecard 3660×H.
3. **WYSIWYG check** — preview logical height (bitmap/2) should equal export
   logical height (bitmap/3) for the same tool/state.
4. **No-reflow check** — shrink the viewport to ~480px; `.wrap` width should stay
   720.
5. Scratch test scripts from this session live under the session scratchpad
   (`smoke.mjs`, `wysiwyg.mjs`, `full.mjs`, `icons.mjs`) — reusable patterns if
   they're still around; otherwise the recipe above reconstructs them.

Note the Effect tool's PNG button is `#btnPng` (an element id), while the other
tools use `[data-el="btnPng"]`.

---

## 7. Publishing the artifact

- The artifact file is **body-content only** — it starts with `<style>` and has
  **no** `<!doctype>/<html>/<head>/<body>` wrapper and no `<title>`. The Artifact
  tool wraps it in a skeleton at publish time, so do **not** add those tags.
- **Republish to the same URL:** pass `url:` =
  `https://claude.ai/code/artifact/dad090ea-7122-44ae-b664-7c1dbdd0c51a`. A
  conversation that didn't originally publish it will otherwise mint a new URL.
- If you get a version-guard error ("hasn't viewed the latest version"),
  `WebFetch` the artifact URL first (to view current state), then republish;
  `force:true` overwrites but should only be used when you intend to discard
  whatever is currently published.
- Artifacts are **private** until shared from the page's share menu.

---

## 8. Git conventions

- Work branch: **`claude/intervention-facts-box-v0kgou`**.
- Commit + push there; use `git push -u origin <branch>` with retry/backoff on
  network errors. Don't open a PR unless asked.
- Do not put the model identifier in commits/PRs.

### History of this effort (most recent last)
```
89f2856 Make the Effect tool preview the exported figure (WYSIWYG)
35ed288 Make the Scorecard preview the exported figure (WYSIWYG)
735d785 Include the Effect icon array in the figure when it is toggled on
7222fce Add developer notes / handoff document
79f3d89 Restore rich narrative and NNT emphasis lost in the canvas rewrite
        (narrativeLines/drawNarrative for <b> runs; NNT discontinuous-CI flag)
6344bf7 Bold the icon-array legend count + refresh these notes
e8ad1da Render the canvas figure chrome in JetBrains Mono (mono stack + await
        document.fonts.ready in paintEffect/paintReport)
77782b1 Document chart-SVG brand-font embedding as a future TODO
(next)  Restore the DALY side-by-side layout as a bespoke canvas painter
        (paintDaly; cfg.kind dispatch; drop the two added metric tiles)
```
(Earlier commits built the DALY tool and the Scorecard tool.)

### Changes since the last notes update (2026-07-31 → 2026-08-17, most recent last)
The doc had fallen behind by 37 commits; this refresh covers them (range
`7f270d5..da1b850`). Grouped by feature, chronological within the group:

```
# Switching Value finish
8e2cc64 Rename Effect size -> Treatment Effect, drop the VHT-time row
a4e51e6 Enter base + switching value, compute the % change

# DALY Source example reports
9ff51f8 Embed an example exported report on the DALY Source tab
6465e86 Relabel the DALY example caption to "Example Report: MMS"
343395b Sync template with artifact: Scale Sensitivity tab + DALY example
3c33b02 Label the DALY example x-axis "DALYS From MMS"
f47b5e5 Add a second DALY example report ("Example Report: IV Iron")
a0fc7c1 Enlarge the DALY example captions and set them black

# Effect Size polish
bb96c12 Add a Maternal Mortality example figure below the download button
04d38c8 Render the effect + DALY example figures live (not baked PNGs)
b6cb3c0 Prefix the effect example caption with "Example Report:"
26b43e8 Vertically center the metric numbers in their cells
11a3e55 Reduce whitespace above the sampling-distribution curve
0b1b740 Balance the metric row when only two metrics show
6c67942 Remove the Confidence intervals (95%) toggle
001a3ee Center-justify the metric row

# Comparative Scale tab (new)
bacb8de Build the Comparative Scale tab (DALYs/year ranking)
66e48db Drop the example arrow, fix the SFS arrowheads
6550f72 Retitle the figure
c1bbd36 Reword the intro caption + force a line break
21416eb Two-line user-bar callout
b55cacd Default name "Example Program" + rule fits callout text
55adb6a Nudge the user-bar callout down +30

# Population Reached examples
f2d9d86 Add a live default-cascade example below the button
5ef14e3 Add a second reading-glasses example
aa355d9 Rename example captions to Conservative / Generous

# Evidence Chain tab (new; renamed from Outcomes Table)
7b6ce1f Rename the "Outcomes Table" tab to "Evidence Chain"
5132a70 Implement the Evidence Chain tab (fully dynamic)
9974bf0 Separate arrows that share a node

# Scale Sensitivity + tornado enhancements
28fa223 Duplicate the CE Sensitivity tool onto a Scale Sensitivity tab
80fbebd Reframe the Scale tornado for total DALYs averted
a3276d1 Default Scale Sensitivity to the cataract dataset
73c406f Sharpen the tornado taper (narrow the bottom two rows)
ff81246 Add light vertical gridlines at the axis ticks
3eace12 Add the cumulative "all parameters" bar (own auto-fit axis)
da1b850 Set the Scale cumulative defaults to 183 / 17582
```
(`5cedff4` was the merge of PR #17 and is omitted above.)

---

## 9. Known dead code & cleanup opportunities

The WYSIWYG refactor left some now-unused code in place (intentionally, to reduce
risk). Safe to remove later, with a full re-test:

- On-screen HTML builders no longer called (audited dead — defined, never
  invoked): `buildMetrics`, `tableCols`, `drawDist`, `drawIcons` (Effect);
  `metricsHTML` (Tornado); `matrixHTML`, `railHTML` (Scorecard).
- **What is _live_ (don't confuse with the above):** metric tiles come from each
  tool's `reportCols`/`metricsCols` (the single source), and the light-blue
  narrative comes from each tool's `narrative()`/`narrativeHTML()` **passed as raw
  HTML** through `narrativeLines`/`drawNarrative`. These are _not_ flattened via
  `textContent` — that flattening was the regression fixed in commit 79f3d89.
- CSS for the old HTML figures is now unused: `.metrics`, `.chart`, `.narrative`,
  `.card__header`/`.card__title`, the `.daly-*` layout rules, the `.sc-body/
  .sc-matrix/.sc-rail/.sc-tier/...` matrix rules, `#benefitTable`, `#rrDist`, and
  the `.iconarray--aside` rule.
- The `.iconarray` on-screen classes are still used only as reference colors in
  the painter's `ICONC` map (the swatches are drawn on canvas now).

(The old template/artifact split and its manual `<script>`-copy sync are gone —
there is a single self-contained file now; see §2.)

### Resolved — brand fonts inside the chart SVGs
The canvas figure chrome (title, metric numbers, labels, legend) renders in the
brand fonts on all tabs, and **the chart interior now does too.** Previously the
Effect distribution axis ticks and `RR` labels, and the Tornado/DALY in-plot
value/axis labels, fell back to a generic monospace because the chart SVG is
rasterized via a data-URL `Image` — an isolated document that can't see the
page's `@font-face`. Fix: `svgFontStyle()` (a memoized module-level helper) reads
the page's own `@font-face` rules out of `document.styleSheets` (the base64 woff2
already inlined in `<head>`) and returns a `<style>…</style>` string; each of the
three chart-SVG wrappers (Effect `svgStr`, Tornado/DALY `paintReport`/`paintDaly`)
injects it right after the opening `<svg>` tag, so the isolated SVG carries its
own fonts. Verified by rasterising the same text with and without the injected
style: ~3,500 of ~6,100 glyph pixels differ, confirming JetBrains Mono is applied.
No file-size cost — the rules are lifted at runtime from the fonts already shipped.

---

## 10. Gotchas

- **Canvas can't carry inline HTML.** The figure is a `<canvas>`, so any inline
  formatting the on-screen HTML used to show (bold, color, superscript) must be
  drawn explicitly by the painter — putting `<b>` in a string alone does nothing.
  Route narrative/legend emphasis through `narrativeLines`/`drawNarrative` (or a
  `flagSub`/bold-face branch like the NNT tile and icon legend). This is the class
  of regression that hit the narrative bold, the NNT flag, and the icon-legend
  count during the WYSIWYG rewrite — assume any _new_ formatting has the same trap.
- **`narrativeLines` is `<b>`-only.** It recognizes top-level `<b>`/`<strong>`
  and otherwise takes `textContent`. `<sup>`/`<sub>` render as baseline text,
  `<br>` silently vanishes, and colored/nested `<span>`s lose their styling. No
  current narrative uses those, but extend the helper first if you add one.
  (`&entities;` are fine — `innerHTML` decodes them before `textContent`.)
- **Preview is async + debounced.** A `redraw()` schedules a paint ~50ms later;
  don't assume the `<img>` src is updated synchronously in tests — wait.
- **Canvas brand fonts need two things, and the painters must each do both.**
  Canvas `fillText` only renders a brand font if (a) that font is **named in the
  font string** and (b) it is **already loaded**. So every painter must:
  1. Build its `mono` stack as `"'JetBrains Mono',ui-monospace,SFMono-Regular,
     Menlo,Consolas,monospace"` and `sans` as `"'Roboto Condensed',system-ui,…"`.
     If `'JetBrains Mono'` is missing from `mono`, the title/numbers/labels
     silently fall back to the OS monospace (SF Mono/Menlo on macOS, DejaVu on
     Linux) — this was a real bug in `paintEffect`/`paintReport` (Scorecard's
     `HEAD` had it right), fixed after the WYSIWYG rewrite.
  2. `await document.fonts.ready` before it measures or draws, so the glyphs are
     decoded first (the page loads both fonts via `--font-heading`/`sans` on
     on-screen chrome, so `fonts.ready` resolves with them present). All three
     painters (`paintEffect`, `paintReport`, `paintScorecard`) now do this.

  Symptom to watch for: exported title/metric numbers in a system mono while the
  narrative (which uses `sans`) looks correct → `'JetBrains Mono'` dropped from
  `mono`, or the paint ran before `fonts.ready`.
- **SVG-in-canvas text** is an isolated document that can't see the page's
  `@font-face`. It now renders in the brand fonts because every chart-SVG wrapper
  injects `svgFontStyle()` (the page's own `@font-face` rules) right after the
  `<svg>` tag. If you add a new rasterized chart SVG, inject `svgFontStyle()` the
  same way or its text will fall back to a generic stack.
- **Scorecard width** is 1220 and it deliberately overflows the 720 column,
  centered. That's intended, not a layout bug.
