# IT Project Management Kanban Board

A single-page IT Project Management demo/training tool for UOB's internal IT PMO: a Kanban board (Backlog / In Progress / Blocked / Done) with drag-and-drop, an Add Task modal, filtering, and email notifications via FormSubmit.

This is a demo tool, not a real UOB system — it uses a neutral "UOB IT PMO" text wordmark and a corporate blue palette only, no real UOB branding.

## Running it

There is no build step, dev server, or package manager. The entire app is one file:

```
open index.html
```

(or double-click it in Finder / drag into a browser). Any change to `index.html` is live on the next page load — just refresh.

There are no tests, linters, or CI configured in this repo.

## Architecture

Everything — markup, CSS, and JS — lives in `index.html`: a `<style>` block, the DOM skeleton (header, filter bar, board, Add Task modal, toast region), and a `<script>` block. There is no framework, no build tooling, no external requests except the FormSubmit call, and no persistence layer.

- Vanilla HTML/CSS/JS only — no frameworks, no bundler, no npm packages.
- No `localStorage`/`sessionStorage`/`IndexedDB`/cookies. Board state lives only in memory and resets on refresh — this is intentional.
- No external resources — no CDN scripts, no Google Fonts, no image files. Icons are Unicode glyphs or the system font stack.
- The only network call is to FormSubmit, for email notifications on new tasks.

See [CLAUDE.md](CLAUDE.md) for full implementation details.
