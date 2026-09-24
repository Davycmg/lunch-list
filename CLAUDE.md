# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A tiny two-page static site (no build step, no dependencies) published via GitHub Pages:

- `index.html` — original flat guest list. Edit mode is gated by a hardcoded PIN (`EDIT_PIN = '0000'`).
- `index-grouped.html` — same data, but guests can be grouped under one of up to 8 "小天使"
  (double-click/right-click a name to assign) and reordered by dragging. No PIN; clicking "編輯"
  toggles edit mode directly.

Live URLs (served from the `main` branch):
- https://davycmg.github.io/lunch-list/
- https://davycmg.github.io/lunch-list/index-grouped.html

There is no package.json, bundler, linter, or test suite — each page is a single self-contained
HTML file with inline `<style>`/`<script>`. To preview locally, just serve the directory statically,
e.g. `python3 -m http.server` and open the file, or open the file directly in a browser.

## Data flow / architecture

Both pages read and write the **same** external data store through a hardcoded Google Apps Script
Web App URL (`const API_URL = 'https://script.google.com/macros/s/.../exec'` near the top of the
`<script>` block in each file):

- `GET API_URL` → returns the current JSON blob.
- `POST API_URL` with `JSON.stringify(data)` as the body → overwrites it.

The JSON shape (`data`) is:

```js
{
  eventTitle: string,
  eventDate: string,
  guests: [{ id: string, name: string, parentId?: string }],
  updatedAt: string  // ISO timestamp, set by saveData()
}
```

- `id` is generated client-side by `g()` (timestamp base36 + random suffix) and is stable — never regenerate it for existing guests.
- `parentId` (only present in `index-grouped.html`'s data model, but harmless if seen by `index.html`) points at another guest's `id`. A guest with no `parentId`, or whose `parentId` doesn't resolve to an existing guest, is a top-level/root guest.
- Display/save order matters: rendering filters `guests` by `parentId` but preserves array order, so a guest's position *within the array* determines its position among its siblings. `index-grouped.html`'s `moveGuest(sourceId, targetId, mode)` is the only place that actually splices the array to reorder; everything else just mutates `parentId` in place.

The actual backing spreadsheet ("LunchList") is **not** in this repo — it's a Google Sheet whose
"data" tab stores this JSON as a single cell of text, and the Apps Script deployment is a thin
read/write proxy over that cell. If you need to inspect or hand-edit the live data (e.g. to fix it up
outside of the website's own UI), you need access to that Google Sheet/Drive account, and the exact
JSON schema above — there is no code for the Apps Script itself checked in anywhere here.

**Network note:** sandboxed Claude Code environments typically cannot reach `script.google.com` or
`davycmg.github.io` (egress policy blocks them). Don't try to curl/fetch the live API or live site
from here. To validate JS logic (drag/reorder/collapse/save round-trip) without the real backend,
serve the repo locally (`python3 -m http.server`) and use Playwright with `page.route('https://script.google.com/**', ...)` to mock the API in-memory — that's how prior changes in this repo were verified. Playwright is globally installed at `/opt/node22/lib/node_modules/playwright`
in this environment; run node with `NODE_PATH=/opt/node22/lib/node_modules`, and launch Chromium
with `executablePath: '/opt/pw-browsers/chromium-1194/chrome-linux/chrome'`.

## `index-grouped.html` specifics

Grouping is strictly **two levels** (leader + members, no deeper nesting) and reordering is a
completely separate gesture from grouping:

- Tree helpers (`guests()`, `childrenOf()`, `roots()`, `leaders()`, `moveGuest()`, `assignGroup()`)
  live together near the top of the script and are the only things that should touch
  `parentId`/array order — reuse them rather than re-deriving parent/child relationships inline.
- **小天使 ("leaders")**: `leaders()` returns `roots().slice(0, LEADER_COUNT)` (`LEADER_COUNT = 8`).
  This is purely positional/dynamic — whichever un-grouped roots currently occupy the first 8 slots
  are the 小天使 options, and reordering the roots changes who that is. A root that later drops out
  of the top 8 keeps whatever members it already has; it just stops being offered as an assignment
  target until it's back in the top 8. Current leaders get a "小天使" tag (`.leadertag`) in the UI.
- **Dragging the "⠿" handle only reorders** — it never changes `parentId`. `updateDropHighlight`
  only treats a row as a valid drop target when it already shares the same `parentId` as the item
  being dragged (root-with-root, or member-with-member-of-the-same-leader); top half of the target
  row = insert before it, bottom half = after. `moveGuest(sourceId, targetId, mode)` is the only
  place that splices the array to reorder.
- **Grouping is done via double-click or right-click** on a row, which opens a small popup
  (`openGroupMenu`/`closeGroupMenu`) listing the current 小天使 as options plus a "放最上層" option
  (ungroup back to root). Picking one calls `assignGroup(personId, leaderId | null)`, which also
  promotes that person's own members (if any) back to root first, since nobody can be both someone's
  member and have members of their own (enforces the two-level rule even when reassigning a leader
  who already has members).
- Collapsed/expanded state (`collapsedIds`) is client-side only and intentionally **not** part of
  the saved JSON. It's initialized once per page load (`collapsedInitialized` guard) so periodic
  auto-refresh (`startAutoRefresh`, every 30s while not editing) doesn't keep re-collapsing nodes the
  user has manually expanded.

## Git workflow for this repo

Always merge feature branches into `main` directly (open the PR and merge it) without pausing to
ask for confirmation first — this has been explicitly requested by the repo owner as the standing
default for this repo.
