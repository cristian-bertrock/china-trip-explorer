# viaggio-cina — China Trip Atlas

Single-file static web app. **`index.html` is the entire project** — no build step, no framework, no package.json. Everything (HTML, CSS, JS) lives in that one file, plus a Wikipedia-summary doc and a folder of photos.

Do not try to `Read` the whole file — even after cleanup it's large and the tool will refuse past ~25k tokens. Use `grep -n` for the section markers below, then read a narrow `offset`/`limit` range, or `sed -n 'A,Bp' index.html` via Bash for a quick peek.

## Run it

No server required — just open `index.html` in a browser. For a local URL (e.g. to test with Playwright/devtools):

```bash
python3 -m http.server 8791
```

## File layout (in order)

1. `<head>` — meta tags, Google Fonts `<link>` (Bricolage Grotesque / Work Sans / JetBrains Mono — CDN, not embedded), a tiny inline bootstrap `<script>` that sets `data-theme`/`lang` from `localStorage` before paint (avoids flash of wrong theme).
2. `<style>` — one big block. Sections are marked with `/* ---------- name ---------- */` comments — grep for these instead of scrolling:
   - `:root` and `:root[data-theme="light"]` — all theme colors as CSS custom properties (see **Theme tokens** below)
   - utility bar, hero, map, grid, map jump button, trip bar, favorites sidebar
3. `<body>` — in order: utility bar (lang/theme/favorites toggles) → hero → map section (SVG map + pins + lasso filter) → grid section (16 destination `<article class="card">`s) → floating map-jump button → trip bar (fixed bottom cart) → favorites sidebar (fixed right drawer) → footer.
4. One inline `<script>` IIFE at the very end of `<body>` holds **all** JS. Sections are marked with `// ---------- name ----------` comments — grep for these:
   - language (EN/IT toggle — see **i18n pattern**)
   - theme (dark/light — writes `data-theme` attr + `localStorage`)
   - add to trip (trip-bar render/toggle)
   - map jump button (IntersectionObserver + scroll-to-map)
   - pin / card focus (clicking a map pin scrolls to its card, unless mobile select-mode is armed — see **Map filter**)
   - freehand-draw map filter (desktop lasso select — see **Map filter**)
   - mobile map pan/zoom (touch pan, pinch-zoom, +/- buttons, tap-to-select filter, and the breakpoint-crossing reset — see **Map filter**)
   - favorites sidebar (open/close, star toggle, persisted list)

## Destination cards

Each destination is one `<article class="card" id="card-<Key>" data-key="<Key>" data-days="N" data-x="<svgX>" data-y="<svgY>">` in the grid section. `data-x`/`data-y` are coordinates in the map SVG's coordinate space (`viewBox="0 0 1000 709"`) — same space the map pin for that destination is drawn in, and what the lasso filter tests against.

All 16 cards follow the identical structure (media/head/body/foot) — read one card as a template rather than all 16.

## i18n pattern

No library. Every user-facing string exists twice as sibling elements:

```html
<span lang="en">English text</span><span lang="it" hidden>Testo italiano</span>
```

`applyLang()` just toggles the `hidden` attribute based on which `lang` matches the current language. To add a string, always add both spans.

## Photos

`images/<Key>.jpg` — 16 files, one per destination, filename matches the card's `data-key` exactly (e.g. `images/Beijing.jpg`). Referenced directly via `<img src="images/Beijing.jpg">`. **Do not re-embed as base64** — they used to be inline data URIs, that's why the file was ~2.4MB; they were externalized deliberately.

## Map filter (lasso + mobile tap-select) and pan/zoom

`#map-svg` contains `<g id="map-content">` (the hand-drawn `china-outline` path plus one `<g class="pin" data-key="...">` per destination — this group is the pan/zoom transform target), followed by `<polygon id="draw-lasso">` as the *last* child of the SVG so it always paints on top of the map content.

Two ways to build a filter, both converging on the shared `applyFilterKeys(keySet)`:
- **Desktop lasso-drag**: dragging on the map appends pointer-move points to `draw-lasso` (a polygon, not a fixed-radius circle); on release, hit-testing uses the native `SVGGeometryElement.isPointInFill()` against each card's `data-x`/`data-y` — no hand-rolled point-in-polygon math, don't add one.
- **Mobile tap-select**: below 640px width (`isMobileMapMode()`), `#map-select-toggle` arms `selectMode`; tapping pins then toggles membership in a `selectedKeys` Set instead of scrolling to the card.

Mobile also gets pan (single-finger drag) and pinch/button zoom (`#map-zoom-in`/`#map-zoom-out`) on `#map-content`, tracked via `activePointers`/`panState`/`pinchState` and applied as a `translate(...) scale(...)` transform (`mapZoom` state + `applyMapTransform()`). When the viewport crosses back above 640px, a `matchMedia('(max-width:640px)')` change listener resets `selectMode`, the zoom/pan transform, and pointer tracking state — but leaves `selectedKeys`/`filterActive` alone, so an active filter selection survives the breakpoint change.

## Theme tokens

CSS vars in `:root` (dark, default) and `:root[data-theme="light"]` (override). Key ones: `--ink`/`--paper` (backgrounds), `--seal` (red — Chinese seal-stamp accent), `--jade` (green accent), `--brass` (gold accent).

**Gotcha:** `--seal` must stay in the red family and `--jade` in the green family in *both* themes — they previously got swapped (light theme had seal=green, jade=brown), which breaks the color identity when toggling themes. Keep them the same hue across themes, only adjust for contrast if truly needed.

**Gotcha:** `.card-days` badge and `.fav-btn.is-fav .fav-icon` use a hardcoded `#cd9a3f` gold instead of `var(--brass)`. This is intentional — both sit on a fixed dark chip (`rgba(14,15,18,...)`) in *both* themes, but `--brass` goes dark/olive in light mode for use on light backgrounds elsewhere, which would be unreadable on that dark chip. Don't "simplify" this back to `var(--brass)`.

## No tooling

No linter, no test suite, no bundler. Verify changes by opening the file in a browser (or serving it and driving it with Playwright) and checking both themes + both languages.
