# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single static HTML file (`index.html`) — no build step, no package manager, no dependencies. It's a read-only "dashboard" that re-presents a raffle/giveaway listing (店家抽獎連結, store raffle links for Funbox/來玩聚 boxing-themed pop-up shops) scraped live from an external site, with client-side filtering, grouping, and link-tracking on top.

There is no server, no test suite, and no build/lint tooling in this repo. Development consists of editing `index.html` directly and opening it in a browser (or serving it with any static file server) to check behavior.

## Running it

Just open `index.html` in a browser, or serve the directory statically, e.g.:

```
npx serve .
```

Note: the page fetches from external URLs on load (see below), so it needs network access and will fail cross-origin fetches if opened via `file://` in some browsers — serving over `http://localhost` is more reliable for testing.

## Architecture

Everything lives in `index.html`: inline `<style>` for the UI, inline `<script>` for all logic. Key pieces, top to bottom in the script:

1. **Storage layer** (`store` object) — thin wrapper around `localStorage` that silently degrades to an in-memory object if `localStorage` is blocked (private browsing, etc.). Two persisted keys:
   - `funbox-board-v1` — filter/UI state (`S`: selected cities, codes, mode, search query, collapsed groups)
   - `funbox-seen-v1` — set of raffle URLs the user has already clicked (`SEEN`), used to gray them out

2. **Data source & scraping** (`SOURCES`, `load()`, `ingest()`) — this app has no backend of its own. It fetches the *actual* raffle-listing page HTML from a hardcoded list of mirror URLs (GitHub Pages + two raw.githubusercontent.com fallbacks for the same upstream repo `uxux11/funbox-line`), tries them in order until one succeeds, then parses the returned HTML with `DOMParser` looking for a specific DOM shape:
   - `.draw-store` — one per store, with `data-draw-city` attribute, `.draw-store-name`, `.draw-start`
   - `.draw-item` inside each store — `.draw-product` text + an `<a href>` (the raffle entry link)

   If all fetches fail (e.g. CORS/network issues), `failed()` shows a textarea so the user can paste the source HTML manually and it's run through the same `ingest()` path. **If the upstream site's markup changes, `ingest()`'s selectors and `normCode()`/`cleanName()` regexes are what will need updating** — this is the most fragile part of the app.

3. **Parsing helpers** (`normCode`, `cleanName`) — regex-based extraction of a product "code" (e.g. `AB-01`) and a cleaned display name from the raw scraped product text, which arrives in a semi-structured free-text format (emoji prefixes, bracketed codes, trailing price text).

4. **State/render cycle** — plain manual DOM diffing via full `innerHTML` re-renders, no framework. `S` (filter/UI state) and `DATA` (parsed content) are module-level mutable globals. Every state change calls `saveState()` then `render()`. `render()` picks between two grouping views:
   - `byStore()` — group by city → store → items (default mode, `mode:"store"`)
   - `byCode()` — group by product code → which stores carry it (`mode:"code"`)

   Both support: city/code multi-select filtering (chip dropdowns), free-text search, collapsible groups (state persisted via `S.closed`), per-group "copy all links" to clipboard, and per-link "seen" tracking that persists across reloads.

5. Everything is wired via plain `onclick`/`oninput` assignment at the bottom of the script — no event delegation framework, no components.

6. **連結歸類工具** (self-contained IIFE just before `loadState()`) — a separate utility opened from the bottom-right FAB (`#fab`) into a modal sheet (`#sheet`). The user pastes free-form "product name + link" text; it groups links by product code (`UX-03`, `BXG-04`…) and outputs a copyable list. Independent of `DATA`/`S`; it only reuses `store`, `copy()` and `toast()`. Its draft input persists under `funbox-tool-v1`. Its CSS classes are prefixed `tl-`/`sheet-` to avoid colliding with the board's `.panel`/`.row`.

## Working in this file

- Keep it a single self-contained HTML file — that's the deliberate design (easy to host anywhere, e.g. GitHub Pages, with zero build step).
- All UI text and content is Traditional Chinese (`lang="zh-Hant"`); keep new user-facing strings consistent with that.
- The CSS uses a small set of custom properties (`--paper`, `--ink`, `--muted`, `--gold`, etc.) defined once in `:root` — reuse these tokens rather than hardcoding new colors.
- Since there's no test suite, verify changes by actually loading the page in a browser and exercising the filter/search/copy/seen-tracking interactions, plus the "paste HTML manually" fallback path in `failed()`.
