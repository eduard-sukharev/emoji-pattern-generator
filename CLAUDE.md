# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, backend-free browser app that generates seamless icon/pattern images (emoji-pattern-generator style, but with custom SVG/PNG icons — e.g. from Flaticon) for both screen formats (iPhone, desktop, Instagram) and print (A4/A5, envelopes, with DPI and bleed). Everything — markup, CSS, and JS — lives in `index.html`. There is no build step, no bundler, no package.json, and no dependencies.

## Running / testing

There is no build or test tooling. To work on this app:

```bash
# Open directly — works for most things, but icon-by-URL fetches are more likely
# to be blocked by the browser under file://
xdg-open index.html   # or just double-click it

# Preferred while developing: serve it so URL-based icon loading and Google Fonts
# behave the same as a real deployment
python3 -m http.server 8934
```

To sanity-check edits without a browser, extract the inline `<script>` and run it through Node's parser (there's no test suite, this only catches syntax errors):

```bash
python3 -c "
import re
html = open('index.html', encoding='utf-8').read()
m = re.search(r'<script>(.*)</script>', html, re.S)
open('/tmp/pp.js','w', encoding='utf-8').write(m.group(1))
"
node --check /tmp/pp.js
```

For interactive verification (this is how the app has actually been debugged in this repo), drive it with the Playwright MCP tools against the `http.server` above: `browser_navigate`, then `browser_evaluate` to poke `config`/`icons` directly and call internal functions, then `browser_take_screenshot`/`Read` the PNG to check visually. There's no automated test runner — visual/behavioral checks via Playwright are the closest thing to a test suite here.

## Architecture

Everything is in `index.html`, organized into clearly delimited `/* ===== SECTION ===== */` comment blocks inside the one `<script>` tag (grep for `==============` to jump between them: СТИЛИ, ФОРМАТЫ, RNG, СОСТОЯНИЕ, ВСТРОЕННЫЕ ИКОНКИ, ДОБАВЛЕНИЕ ИКОНОК, РАСТЕРИЗАЦИЯ ИКОНОК, ПОСТРОЕНИЕ СЦЕНЫ, РЕНДЕР В CANVAS, SVG-ЭКСПОРТ, UI: *, ПРИВЯЗКА ЭЛЕМЕНТОВ, ПЕРСИСТЕНТНОСТЬ, PERMALINK, ИНИЦИАЛИЗАЦИЯ).

The core design principle: **geometry is computed once, in final-output pixels, independent of where it's drawn.** The same scene data drives the on-screen preview (scaled down) and the full-resolution export (PNG/JPEG/SVG), so what you see in the preview is exactly what you get in the exported file, including random layouts.

### Data flow

```
config (plain, JSON-serializable object — DEFAULT_CONFIG shape)
icons  (separate global array — NOT part of config; see below)
   │
   ├─ buildScene(config, icons) → { canvasW, canvasH, bounds, placements[], sceneRotRad }
   │     Placement = { iconId, x, y, size, rot, opacity, color, layer }
   │     Pure function. All randomness comes from a seeded PRNG (mulberry32,
   │     seeded by config.seed) via makeRng() — this is what makes "shuffle"
   │     (new seed) and preview↔export consistency possible.
   │
   ├─ renderScene(ctx, scene, cfg, iconsList, scale, opts)   → draws pattern + text
   │     onto any 2D canvas context; used for both the live preview canvas
   │     (downscaled) and the full-res export canvas (scale=1).
   │
   └─ serializeSvg(scene, cfg, iconsList) → SVG string for vector export
         (icons embedded as <symbol>/<use>, raster icons as data: URIs,
         text as real <text> with an SVG filter for shadow)
```

- `config`: one plain object matching `DEFAULT_CONFIG` (format, layout, rotation, color, text, seed, jpgQuality). All UI controls read/write directly into this object.
- `icons`: a separate global array, **not** nested in `config`, because icons carry non-serializable runtime state (`Image` objects, rasterization caches) and can be large (raster data). Icon shape: `{ id, kind: 'svg'|'raster'|'emoji', name, svgText?, viewBox?, innerSvg?, img?, naturalW?, naturalH?, ch?, enabled, tainted, sourceUrl?, _rastCache }`. `_rastCache` is a per-icon `Map` keyed by rasterized-size "bucket" (see `bucketSize()`/`rasterizeIcon()`) so re-rendering at the same size doesn't re-rasterize SVGs.
- UI bindings (`bindPair`/`bindSimple` in the ПРИВЯЗКА ЭЛЕМЕНТОВ section) wire `<input>`/`<select>` elements straight to `config.*` fields and call `onConfigChanged()` → `scheduleRender()` (rAF-debounced) + `persistDebounced()`.

### Icons: sources, rasterization, recoloring

Icons can come from: the built-in inline-SVG set (`BUILTIN_ICONS`), user file upload/drag-drop/paste, a direct URL (with an optional CORS-proxy fallback), or an emoji fallback (rendered via `fillText`, explicitly *not* recommended for print — font-dependent).

SVGs are rasterized on demand to an exact target size via `createImageBitmap(blob, {resizeWidth, resizeHeight})` (falls back to an `<img>` + canvas draw) and cached per size-bucket — this is the mechanism that avoids the blurry/low-res look of naive emoji-pattern tools. Recoloring (`recolorSvgText`) rewrites `fill`/`stroke` in the SVG source; it only affects paths that don't already hardcode a color, which is a known limitation (documented in README).

An icon can become `tainted` (CORS) if loaded cross-origin without a clean CORS response. Tainted icons silently block `canvas.toBlob()` (PNG/JPEG export) — `exportPng`/`exportJpeg` check for this up front and tell the user to remove the icon or use SVG export instead, which doesn't need canvas pixel access.

### Layout schemes

`buildScene` dispatches on `config.layout.scheme` to one of: `grid`, `checker`, `brick`, `scatter`, `fillscatter` (solid background tiling + scatter on top). Shared helpers: `walkGrid()` (grid/checker/brick tiling with optional row/column brick offset) and `walkScatter()` (density-based random placement with a spatial-hash min-distance check). Icon/color cycling across placements goes through `makeCycler()`/`makeCheckerCycler()`, which implement the "order" modes (sequence/random/by-row/by-column) shared by icon selection and palette-color selection.

**Whole-pattern rotation** is handled by building the layout inside an oversized square (`bounds`, sized to the canvas diagonal) and rotating that square around the canvas center at render time — this guarantees no empty wedges appear at the corners at any rotation angle, without needing per-scheme corner-case logic.

### Text overlay

`config.text` is a draggable meme-style text layer drawn on top of the pattern (not affected by the pattern's own scene rotation). Position/rotation/scale are stored as *fractions* of canvas size (`xFrac`, `yFrac`) plus a `scale` multiplier and `sizeRatio` (font size as % of canvas width) — this is deliberate, so the text layer stays correctly positioned/sized across format changes. The on-canvas drag/rotate/scale handles are a separate DOM overlay (`#textHandles`/`#textBox`) positioned in CSS pixels, kept in sync with the canvas via `updateTextOverlay()` — it must be recomputed from the *actual* rendered canvas size (see next section), not from intended/requested dimensions, or the handles drift from the text.

Web fonts (the "circus poster" font group: Lobster, Luckiest Guy, Bangers, etc., loaded from Google Fonts) are explicitly awaited via the CSS Font Loading API (`ensureFontReady()`) before every measure/draw — `drawTextLayer()` and `renderScene()` are `async` specifically for this, otherwise canvas would silently paint with a fallback font on the first frame. Most of these decorative fonts only cover Latin glyphs; Cyrillic text falls back silently to the generic family (documented in README).

### Preview sizing (fragile area — read before touching)

`computeAvailablePreviewBox()` measures the *actual* available space inside `#stage` (accounting for padding, gaps, and the info/warning text rows) and fits the canvas aspect ratio into it exactly. **Do not** go back to sizing the preview canvas from a flat constant (e.g. "max 860px") and letting CSS `max-width`/`max-height` clamp it — those two properties clamp width and height *independently*, which silently breaks the aspect ratio whenever the viewport is short, and desyncs the text-drag overlay from the canvas (both were real bugs fixed in this codebase). `pw`/`ph` computed in `renderPreview()` must always be the *actual* on-screen canvas size, since `updateBleedOverlay()` and `updateTextOverlay()` both position DOM overlays from those exact numbers.

### Persistence

Three independent mechanisms, all going through the same serialization helpers:
- **Autosave**: `config` and `icons` (via `iconToJson(icon, embed=true)`) are separately debounced into `localStorage` (`ppg.config.v1`, `ppg.icons.v1`) on every change. Icons must be persisted separately from config — they used to not be, which meant custom/emoji icons were silently lost on every reload (fixed; don't reintroduce that gap).
- **JSON export/import**: `saveConfigJson()`/`loadConfigJsonFile()`, full `{config, icons}` dump, with an "embed icons as base64" checkbox for raster icons.
- **Permalink**: `copyPermalink()`/`loadFromHash()`, base64-in-`location.hash`.

All three restore icons through the single shared `restoreIconsFromJson()` — if you add a new icon `kind` or a new persisted field, update that function (and `iconToJson`) rather than duplicating restore logic per mechanism. `resetAllToDefaults()` is the inverse: clears both localStorage keys, clears the hash, resets `config` to `DEFAULT_CONFIG`, and reinstates the two default built-in icons.

## Docs

`README.md` is English, `README.ru.md` is Russian; each links to the other. They're kept as full independent translations (not one canonical + a stub) — if you change a user-facing feature, update both.

## Known limitations (already documented in README, don't re-litigate as bugs)

- SVG recoloring only works via CSS inheritance (`fill`/`stroke` not already hardcoded in the source).
- Very dense scatter layouts cap the *preview* placement count for responsiveness (`PREVIEW_MAX_PLACEMENTS`) — export always renders the full set.
- Extreme DPI × large format combos can exceed browser canvas size/area limits (`checkCanvasLimits`) — PNG/JPEG export is blocked with a message; SVG export has no such limit.
- Decorative Google Fonts in the text layer are Latin-only; Cyrillic silently falls back.
