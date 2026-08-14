# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A single-page tool for laying out trading card / sticker images on an A4 sheet for printing, with crop marks and adjustable spacing. Entirely client-side, Japanese UI.

## Commands

There is no build step, package manager, linter, or test suite — the entire app is one static file: `index.html` (HTML + CSS + JS inline).

- **Develop**: edit `index.html` directly and open it in a browser. `file://` works for basic checks, but drag-and-drop/print behavior is easiest to verify via a local server, e.g. `python3 -m http.server 8080` then visit `http://localhost:8080/index.html`.
- **Deploy**: push to `main`. GitHub Pages (legacy build, configured to serve from `main` branch root) rebuilds automatically and publishes to `https://kotalow555.github.io/card-print-layout/`. There is no separate deploy command.

## Architecture

Everything lives in `index.html`. The layout is driven by CSS custom properties on `:root`, which JS mutates in response to UI state — there is no framework or virtual DOM; `render()` rebuilds the `.page` grid's DOM directly on every relevant change.

Key custom properties: `--card-w`, `--card-h` (current card size), `--col-gap`/`--row-gap` (spacing between cells, can differ per axis), `--frame-w` (cut-line outline thickness), `--mark-size` (size of the corner crop marks).

**Size groups**: size preset buttons carry a `data-group` of either `card` or `square`, read via `.size-toggle` elements. Only one button across both groups is ever active.
- `card` group (59×86mm / 63×88mm): fixed 3×3 grid, matches physical trading card sizes.
- `square` group (48×48mm / 52×52mm, labeled "シール"/stickers in the UI): grid size is computed, not fixed — `computeAutoGrid()` finds the max columns/rows that fit an A4 page within `SAFE_MARGIN_MM` of margin, and `computeSquareGaps()` decides per-axis whether there's enough leftover space to widen that axis's gap (`GAP_MAX`) without eating into the margin below `MIN_EXTRA_MARGIN_MM`, otherwise it stays at the safe minimum (`SQUARE_GAP_MM`). This split exists because real printers clipped square-template prints when gaps/margins were computed too aggressively — see git history for the specific incident.

**Template resolution flow**: `getActiveTemplate()` is the single source of truth — given the active button, the "fine adjust" width/height offsets (applies to any group; `fineAdjustDefaultFor()` derives the default offset per button from `FINE_ADJUST_DEFAULT_SCALE`, i.e. the ratio that adds +0.15mm to a 59mm width / +0.8mm to an 86mm height, applied proportionally to that button's base size), it returns `{ cols, rows, w, h, colGap, rowGap }`. `applySize()` takes that, writes the CSS custom properties, resizes `cellImages` to match the new cell count (preserving already-placed images by index), updates the title text, and calls `render()`.

**Rendering**: `render()` clears and rebuilds `#page`: dashed `.divider-v`/`.divider-h` elements mark cut lines between cells, then one `.cell` per grid position (drag-and-drop target + click-to-upload + corner crop marks). Cell count and grid track sizes come from `buildTrack()`, which is regenerated on every `applySize()` call since column/row counts vary by template.

**Printing**: `window.print()` is wrapped with `beforeprint`/`afterprint` listeners and a `matchMedia('print')` listener, all funneled through `setPrinting()`, because some mobile browsers don't apply `@media print` on the actual print output — `setPrinting()` also toggles an `html.printing` class that mirrors the print media query in regular CSS as a fallback. `setPrinting()` also swaps `document.title` to a timestamp (`formatPrintTimestamp`) so the browser's PDF-save filename is date/time-based, restoring the original title afterward.
