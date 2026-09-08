# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page IT Project Management demo/training tool for UOB's internal IT PMO: a Kanban board (Backlog / In Progress / Blocked / Done) with drag-and-drop, an Add Task modal, filtering, and email notifications via FormSubmit. This is a demo tool, not a real UOB system — it uses a neutral "UOB IT PMO" text wordmark and a corporate blue palette only, no real UOB branding.

## Running it

There is no build step, dev server, or package manager. The entire app is one file:

```
open index.html
```

(or double-click it in Finder / drag into a browser). Any change to `index.html` is live on the next page load — just refresh.

There are no tests, linters, or CI configured in this repo.

## Architecture

Everything — markup, CSS, and JS — lives in `index.html`: a `<style>` block, the DOM skeleton (header, filter bar, board, Add Task modal, toast region), and a `<script>` block. There is no framework, no build tooling, no external requests except the FormSubmit call, and no persistence layer.

**Hard constraints that shape the code** (do not violate these when editing):
- Vanilla HTML/CSS/JS only — no React/Vue/jQuery/Tailwind, no bundler, no npm packages.
- No `localStorage`/`sessionStorage`/`IndexedDB`/cookies. Board state lives only in the in-memory `state` object and resets on refresh — this is intentional, not a bug (there's an on-page note saying so).
- No external resources — no CDN scripts, no Google Fonts, no image files. Icons are Unicode glyphs (✕, ⚠, ▸) or system font stack.
- The only network call is to FormSubmit; everything else is synchronous in-memory state manipulation.

**State model** (`state` object, ~line 640 in the script):
```js
state = {
  tasks: [],              // array of task objects, the single source of truth for the board
  filters: { project, assignee, priority },
  nextIdNum: 1,            // counter used to generate UOB-ITPM-#### IDs
  deleteConfirmingId: null // id of the card currently showing its inline Delete? Yes/No toggle
}
```
Each task: `{ id, title, description, project, category, assignee, priority, dueDate, status }`, where `status` is one of the four column names and doubles as the Kanban column key.

**Render flow**: All UI updates go through `renderBoard()`, which rebuilds the four columns from `applyFilters(state.tasks)` and calls `renderCard()` per task and `renderSummary()` for the header stats. There is no direct DOM mutation of card content outside this render path — mutations always go through a state-changing function (`moveTask`, `deleteTask`, `addTask`) followed by a re-render. Event handling on cards (move-select, delete toggle) uses event delegation on the `#board` container (`attachBoardDelegation()`), not per-card listeners, because cards are destroyed and recreated on every render. Drag-and-drop listeners, however, are re-attached per element in `attachDragAndDropHandlers()`, called at the end of every `renderBoard()`.

**FormSubmit integration** (`notifyNewTask()`): fire-and-forget from `addTask()` — the card is added to `state.tasks` and rendered optimistically before the network call resolves. A failure is caught and surfaced as a non-blocking toast ("Card added locally — email notification failed"); it must never remove the card or throw. The destination address is a single constant, `FORMSUBMIT_ENDPOINT`, near the top of the `<script>` block — change it there only. FormSubmit requires a one-time activation per destination email (first submission triggers a confirmation email; notifications don't deliver until that link is clicked).

**Security**: all user-supplied strings (task title, description, assignee, etc.) must go through the `escapeHtml()` helper before being interpolated into any `innerHTML` string — every existing render function does this; keep doing it for new fields.
