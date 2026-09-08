# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page IT Project Management demo/training tool for UOB's internal IT PMO: a Kanban board (Backlog / In Progress / Blocked / Done) with drag-and-drop, an Add Task modal, filtering, and email notifications via FormSubmit. This is a demo tool, not a real UOB system — it uses a neutral "UOB IT PMO" text wordmark, no real UOB branding. The dashboard chrome (header, backgrounds, buttons, typography) stays neutral/corporate-blue for readability; a rainbow accent system layers on top of it — see "Rainbow accent palette" below.

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
- No `localStorage`/`sessionStorage`/`IndexedDB`/cookies for board state. Board state lives only in the in-memory `state` object and resets on refresh — this is intentional, not a bug (there's an on-page note saying so). **Scoped exception:** a single `localStorage` key, `THEME_STORAGE_KEY` ("uobItpmTheme"), stores only the user's light/dark theme choice — never task/board data. Reads/writes to it are wrapped in try/catch (private browsing can throw). Don't widen this exception to any other data without explicit user sign-off.
- No external resources — no CDN scripts, no Google Fonts, no image files. Icons are Unicode glyphs (✕, ⚠, ▸, 🌙, ☀, 🔒) or system font stack.
- The only network call is to FormSubmit; everything else is synchronous in-memory state manipulation.
- A `Content-Security-Policy` and `referrer` meta tag live in `<head>`— keep `connect-src`/`form-action` scoped to `'self'` and `https://formsubmit.co` if editing them; don't loosen to `*` or add new external origins without reason.

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

**Rainbow accent palette**: Kanban columns, priority pills, and category tags each carry a distinct hue (columns: Backlog=violet, In Progress=blue, Blocked=red, Done=green; priority: Critical=red → High=orange → Medium=amber → Low=green as a heat scale; category: one hue per project category via `categoryTagClass()`). Every color pairing carries a text label too — color is never the only signal (WCAG 1.4.1).

Two contrast traps bit this palette during development and must not be reintroduced:
- **Dark-theme text tokens ≠ solid-fill badge colors.** `--red`/`--green`/`--violet`/etc. are *lightened* in the dark theme for text-on-dark-surface use. Reusing them as a solid background behind white text (e.g. the column `.count-badge`) drops contrast as low as 1.6:1. Solid white-text fills use the dedicated, theme-invariant `--badge-*` tokens instead (defined once in `:root`, deliberately not overridden in the dark blocks).
- **A tint/text pair must be defined together per theme, not assembled from unrelated tokens.** `.tag-application-development` originally paired `--blue-100` (bg) with `--blue-700` (fg) — both go dark in dark mode, so contrast collapsed to ~1:1. Use a dedicated bg/fg pair per theme instead (see `--tag-blue-bg`/`--tag-blue-fg`, and `--accent-text` for the equivalent single-color case).

When adding a new hue or reusing an existing token in a new pairing, verify computed contrast ≥ 4.5:1 in **both** themes before shipping — don't assume a token that looks fine in light mode still pairs correctly once the dark-theme block redefines it.

**Chart palette** (Board Analytics panel, `renderAnalytics()`): the two bar charts (`chartStatus`, `chartPriority`) use their own `--chart-status-*` / `--chart-priority-*` tokens, deliberately separate from the `--badge-*`/column-accent rainbow above. They were chosen with the dataviz skill's `validate_palette.js` and must stay validated:
- **Status chart** (Backlog/In Progress/Blocked/Done) is a categorical set — reusing the existing violet/blue/red/green column-accent hues here was tried and *failed* the CVD-separation and normal-vision-floor checks (violet↔blue Delta E as low as 2.5/11.2, well under the 8/15 floors). It uses the dataviz skill's validated 4-slot categorical order instead (`references/palette.md`: blue → orange → aqua → yellow), confirmed to pass against this app's actual `#ffffff`/`#121a2b` surfaces.
- **Priority chart** (Low→Medium→High→Critical) is an ordinal severity scale, not 4 unrelated hues — encoded as a single-hue red ramp (light→dark = Low→Critical) per the skill's "sequential/ordinal = one hue" rule, validated with `--ordinal`.
- If you change either chart's colors, re-run `node <dataviz-skill>/scripts/validate_palette.js "<hex,hex,...>" --mode light|dark --surface "#ffffff"` (or `"#121a2b"` for dark) before shipping — don't hand-pick hues by eye.
- **Comment-text trap hit while building this**: a CSS comment containing the literal substring `*/` as prose (e.g. writing `--badge-*/column-accent` to mean "badge-star, slash, column-accent") silently closes the comment early and corrupts every rule until the next real `*/` — it fails silent (no console error; custom properties just stop resolving and `fill` falls back to black). Avoid `*/` appearing in comment prose; if genuinely needed, break it up (e.g. `*` `/`) or reword.
