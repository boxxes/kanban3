# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running / developing

There is no install step, build step, linter, or test runner — the project has no `package.json` and no dependencies. It is a single static file, `index.html`, meant to be opened directly in a browser:

```
start index.html
```

To verify a change, open the file in a browser and exercise it manually (add/move/delete a task, drag-and-drop between columns, filter, submit the Add Task form).

## Architecture

Everything — markup, styles, and script — lives in the one file, `index.html`: a `<style>` block, then the page markup, then a single `<script>` block at the bottom. There are no other source files and no external CSS/JS/font/image resources loaded.

**State model:** a single `state = { tasks: [...], filters: {...} }` object is the source of truth. All mutation happens through named functions that then re-render from `state` — do not mutate card DOM directly outside of a render call:
- `renderBoard()` / `renderCard()` — render the four columns and cards from `state`
- `addTask()`, `moveTask()`, `deleteTask()` — mutate `state.tasks`, then re-render
- `applyFilters()` — derives the visible task list from `state.tasks` + `state.filters`
- `showToast()` — transient UI feedback, not part of `state`
- `escapeHtml()` — used for every user-supplied string interpolated into the DOM; don't reintroduce raw `innerHTML` interpolation of user input

**No persistence, by design:** there is deliberately no localStorage/sessionStorage/IndexedDB/cookies. A page refresh always resets `state.tasks` to the 8 seeded demo tasks, and the UI notes this. Don't add a storage layer unless explicitly asked — the in-memory-only behavior is intentional for this demo tool.

**FormSubmit integration:** `notifyNewTask()` posts to the `FORMSUBMIT_ENDPOINT` constant (near the top of the script) — the only outbound network call in the app, used to email a notification when a task is added. Notes:
- FormSubmit requires a one-time confirmation email to be clicked before it will actually deliver notifications to a new address.
- Task creation is optimistic: the card is added to `state` and rendered regardless of network outcome. A FormSubmit failure must never block or roll back the board — it only surfaces as a non-blocking toast.

## Constraints to preserve

This app was built under explicit constraints that should be preserved unless the user asks otherwise:
- Vanilla HTML/CSS/JS only — no frameworks (React, Vue, jQuery, Tailwind, etc.).
- No build step, bundler, or npm/package.json.
- No CDN or other external resource loads (fonts, scripts, icons) — the file must work standalone, opened directly from disk.
- No persistence APIs (localStorage/sessionStorage/IndexedDB/cookies) — see "No persistence, by design" above.
