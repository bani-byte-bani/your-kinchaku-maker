# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"nemukuta" — a static, single-page kinchaku bag (巾着袋) design configurator. A user picks a name, font, thread color, and fabric, sees a live SVG preview in two shapes (closed/open), can drag the name to reposition/resize it, then either downloads a PNG order sheet or opens a prefilled `mailto:` order email. No backend, no build step, no dependencies.

## Running it

There is no build/lint/test tooling in this repo. Serve the directory with any static file server (a `fetch()` for `shapes.json` and `svg/*.svg` means opening `index.html` via `file://` will fail on CORS) — e.g. `python3 -m http.server` from the repo root — and open `index.html`.

## Architecture

Everything lives in `index.html`: inline `<style>`, inline markup, inline `<script>`. It's organized into numbered sections (see the `/* === N. ... === */` comment banners) — state, validation, SVG loading/mutation, drag handling, UI construction, render, PNG export, mailto export, init. When making changes, find the right numbered section rather than searching blindly.

**Render model:** one mutable `state` object (name, font, textColor, fabricId, fabricColor, fontSize, shapeId, namePos) drives everything. Any handler that changes `state` calls `refresh()`, which re-syncs form controls, re-applies the design to both preview `<svg>` clones, re-renders the order panel, and rewrites the URL query string (`syncUrl`/`loadFromUrl` make designs shareable/reloadable via URL params). Dragging the name is the one exception: it mutates the SVG DOM directly during `pointermove` for performance and only calls `syncOrderPanel`/`syncUrl` on `pointerup` (see `onDragMove`/`onDragEnd`).

**External SVG contract** (`svg/closed.svg`, `svg/open.svg`, documented in comments inside those files): each must keep `viewBox="0 0 240 310"` (mirrored in the `VIEWBOX` const in `index.html`), mark the fabric-fillable shape(s) with `id="bag-fabric"` (or `class="bag-fabric"` for multiple), and include an empty `<text id="name-text">` where the name gets injected. Anything else (ropes, stitching, shadows) is free-form. If a shape SVG is missing or fails to parse, `buildFallbackSVG()` generates a prototype shape procedurally so the app still works.

**Fabrics** are defined in the `FABRICS` array — each entry supplies a small swatch renderer, an optional `<pattern>`/`<image>` `defs()`, and a `fill()` value applied to `#bag-fabric`. Photo fabrics (`photo-linen`, `photo-cotton`) reference files in `fabrics/`; swap those JPGs to change the texture without touching code. Solid fabric needs a user-picked color (`needsColor: true`), which shows/hides the color picker control.

**Shapes** default to `DEFAULT_SHAPES` (closed/open) but `shapes.json` is fetched at init and overrides them if present and valid — add a shape by adding an entry there plus a matching `svg/<id>.svg` file.

**PNG export** (`exportImage`/`downloadImage`) does not rasterize the whole SVG directly, because the browser's SVG-to-Image path doesn't reliably load Google Web Fonts. Instead it draws the bag SVG (with the `<text>` and drag handle stripped out) via `drawSVG()`, then draws the name text separately on the `<canvas>` with `ctx.fillText`, plus a hand-drawn order-info panel below it.

**Order email**: `ORDER_EMAIL` near the top of the `<script>` is a placeholder (`order@example.com`) and needs to be set to a real address before this is used for actual orders. `buildOrderMailto()` builds a `mailto:` link summarizing the order; it does not attach the exported PNG (the UI tells the user to attach it manually).

## Localization

All user-facing strings are Japanese; this is a Japanese-market product page, not an i18n'd app.
