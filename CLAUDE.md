# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Woodworking build plans and sheet-goods cut plans — not an application. There is no
build system, no package manager, no test framework, and no dependencies to install.
Every deliverable is a self-contained HTML or Markdown file. The only external
dependency anywhere is three.js, loaded from a CDN via import map in
`lumber_cart/lumber_cart_3d.html`.

One directory per furniture project: `lumber_cart`, `media_console`, `mitre_station`,
`workbench`.

## Commands

Open any plan directly in a browser:

```bash
open mitre_station/cut_plans.html
```

Some tooling (Playwright, for one) blocks the `file:` protocol. Serve over HTTP when
you need to drive a page programmatically:

```bash
cd mitre_station && python3 -m http.server 8731    # then http://127.0.0.1:8731/cut_plans.html
```

**Validating a layout change without a browser.** `mitre_station/cut_plans.html` keeps
its data model and geometry in one contiguous script region, so it can be sliced out
and run under node. This is the fastest way to check a layout edit:

```bash
sed -n '/^const SHEET_W/,/^function render/p' mitre_station/cut_plans.html | sed '$d' > /tmp/lay.mjs
cat >> /tmp/lay.mjs <<'EOF'
for (const t of ['t34','t12','t14']) {
  const L = LAYOUTS[t], errs = validate(L);
  console.log(t, L.sheets.length + ' sheet(s)', errs.length ? errs : 'OK');
  L.sheets.forEach((s,i) => console.log(`  sheet ${i+1}: ${fr(sheetH(s))}"/96", ` +
    `${fr(Math.max(...s.map(rowW)))}"/48", ${(sheetArea(s)/4608*100).toFixed(1)}% yield`));
}
EOF
node /tmp/lay.mjs
```

The same `validate()` also runs in the browser on page load and prints a green or red
banner above the drawings, so a broken layout is visible without running node.

## Architecture

### Spec lives in Markdown, layout lives in HTML

Each project's `README.md` (or `lumber_cart_plan.md`) is the human source of truth:
finished dimensions, material thicknesses, build spec prose, and part tables with
quantities. The HTML files consume those numbers but are **not generated from them** —
there is no pipeline. Part sizes and quantities are duplicated by hand into each HTML
file, so **any dimension change must be applied in both places**.

### Two generations of cut-plan file

- **Hand-authored static** (`media_console/*.html`, `mitre_station/cut_plan_all.html`):
  a single inline SVG with hardcoded pixel coordinates and no `<script>` at all.
  Changing a dimension means recomputing every downstream `x`/`y`/`width` by hand.
  Treat these as finished artifacts; prefer not to extend them.
- **Data-driven** (`mitre_station/cut_plans.html`): a declarative model rendered by JS.
  New cut plans should follow this shape.

### The data-driven model (`mitre_station/cut_plans.html`)

Two data structures drive everything; the rest of the file is rendering and validation.

`PARTS` is a flat catalog keyed by short id, spanning all three material thicknesses:

```js
c1s: { n:'C1 Side', wid:4.5, len:20, qty:16, c:'#3498db' }
```

- `wid` is the dimension **across the sheet's 48" width**; `len` runs **along the 96"
  length**. This is the natural orientation, and it also sets grain direction — sheet
  face grain runs along the 96" axis, so a part's `len` dimension is its grain
  direction unless a column rotates it.
- `qty` is what validation checks placements against, so it must match the README.

`LAYOUTS` nests **sheet → rows → columns → pieces**, built with the `row`/`col`/`rcol`/`rep`
helpers. Rows stack down the 96" length; columns sit across the 48" width within a row;
pieces stack down a column. Every boundary in that nesting is a **guillotine cut** — a
cut that crosses the full width of the stock it is dividing — which is what makes a
layout actually cuttable on a table saw. `rcol` rotates every piece in a column
(`len` across, `wid` along).

Row heights, column widths, and sheet lengths are all **derived** from the pieces via
`rowH`/`colW`/`sheetH`; you never write a coordinate. `place()` walks the nesting and
emits absolute rects. Only add a piece to a column and let the geometry fall out.

`validate()` checks placed counts against `PARTS[].qty`, every row against 48", and
every sheet against 96".

## Domain conventions

- Sheet stock is 48" × 96"; kerf is 1/8" (`KERF = 0.125`) and is applied between every
  adjacent piece, column, and row.
- Dimensions are decimal inches in code, rendered to sixteenths by `fr()` (`19.4375` →
  `19 7/16`) for display and cut lists.
- `SCALE = 5.5` px per inch. Drawings are always portrait: 48" across, 96" down.
- Cut-guide colors in the SVG: solid red = crosscut a row off, dashed red = rip between
  columns, thin grey = crosscut within a strip.
- Parts of identical size across cabinets may be merged into one `PARTS` entry even
  though the README lists them separately — e.g. `c1btm` qty 10 is the README's 8
  shallow drawer bottoms plus 2 bottom-drawer bottoms, which share a size.

## Planning guidance

Cabinets are built **one at a time**, so minimizing total sheet count is necessary but
not sufficient: concentrate a single cabinet's parts onto as few sheets as possible,
even at a few points of yield, so that cabinet can be finished without breaking down
stock earmarked for the others. When a cabinet's parts genuinely cannot fit on one
sheet, check the area against 4608 sq in (kerf adds roughly 3% on top) and state the
number explicitly, then isolate the spillover into one contiguous edge strip so the
rest of the sheet stays whole. Record the resulting build order in the layout's `notes`
array, which renders above the drawings.
