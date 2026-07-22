# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

This repo is a single file: `index.html`. That file *is* the application — "Strength v5" ("v5 Tracker"), a mobile-first workout logger for a fixed 7-day strength + conditioning program (Mon/Wed/Fri/Sat strength lifts, Tue Zone-4 rowing intervals, Thu Zone-2 aerobic work, Sun rest). There is no other source, no server, and no backend beyond an optional user-configured Google Sheet.

## Commands

There is no build step, package manager, linter, or test suite — `index.html` is the complete, dependency-free deliverable.

- **Run/preview**: open `index.html` directly in a browser, or serve it (e.g. `python3 -m http.server`) if you need to test behavior that differs under `file://`.
- React 18 (production UMD build) and the DM Sans / DM Mono fonts are pulled from CDNs (`unpkg.com`, `fonts.googleapis.com`) via `<script>`/`<link>` tags in `<head>` — there is nothing to `npm install`.
- No automated tests exist. Validate changes by opening the file in a browser and exercising the flow you changed; inspect `localStorage` (DevTools → Application → Local Storage) for the `v5_*` keys to check persisted state.

## Code structure and conventions

- Everything lives in one `<script>` block at the bottom of `index.html`, in roughly this order: storage helpers → program data (`P`) → math/formatting helpers → session-building logic → components (leaves first, `App` last) → `ReactDOM.createRoot(...).render(...)`.
- **No JSX, no build step.** Components are written directly as `React.createElement(...)` calls (the `_extends` helper at the top is a hand-rolled `Object.assign` spread, the kind of thing Babel emits for JSX spread props). When adding or editing components, keep writing plain `React.createElement` calls — don't introduce JSX syntax, since nothing in this repo compiles it before it reaches the browser.
- All styling is inline `style={}` objects computed per render. The only CSS is a handful of keyframe animations (`slide-up`, `pop`, `pulsing`, `fade-in`) and resets in the `<style>` tag in `<head>` — there's no CSS framework, no CSS modules, no external stylesheet.
- All state lives in the top-level `App()` component (`screen`, `sessions`, `queue`, `sheetsUrl`, `unit`, `activeSess`) and is threaded down as props. There's no context, no state management library, and no router — screen switching is a plain string in state (`'home' | 'session' | 'history' | 'settings'`).

## Domain model

- `P`, defined near the top of the script, is the hardcoded weekly program: one entry per day (`mon`…`sun`) with a `kind` of `strength`, `z4`, `z2`, or `rest`. Strength days carry an `exercises` array (block, name, prescription text, sets/reps/rest seconds, coaching cue); the `z4` day (Tuesday) carries interval `pieces`; the `z2` day (Thursday) is duration/HR based. `P` is the single source of truth for what a session looks like — changing the program means editing this object directly.
- `makeSession(dayKey, date, lastSess, unit)` builds a fresh in-progress session from `P`, pre-filling each set's weight from the previous logged session for that day (converting units if the unit preference changed between sessions).
- A finished session is flattened into spreadsheet rows by `sessionToRows()`, using the fixed `COLS` column schema. That schema is the contract with the Google Sheet on the other end of sync — if you change it, keep it in sync with the Apps Script `doPost` handler embedded as the `SCRIPT` template string inside `SettingsScreen` (this is the literal source shown to the user to copy into Google Apps Script; it just appends `rows` under `header` to a "Log" sheet).
- Estimated 1-rep max (e1RM) is computed everywhere via the Epley formula (`epley(weight, reps)`). Weights are stored internally in **lb**; `toLb`/`toDisplay` (factor 2.2046) convert to/from the user's selected display unit (`lb`/`kg`).

## Persistence and sync

- All persistent state is `localStorage`, under fixed keys in `SK`: `v5_sessions`, `v5_queue`, `v5_url`, `v5_active`, `v5_unit` (`v5_synced` is defined but currently unused). The app itself has no backend.
- Sync is optional and one-directional (app → sheet): the user pastes a Google Apps Script Web App URL into Settings. Saving a session appends its rows to an offline queue (`SK.queue`) and fires a single fire-and-forget `fetch(url, {method:'POST', mode:'no-cors', ...})`. Because `no-cors` makes the response opaque, a fetch that doesn't throw is treated as success and the queue is cleared — there is no server acknowledgment. Sync also retries on the browser's `online` event and via a manual "Sync Pending Rows" button in Settings. This queue-and-retry path is the correctness-critical part of the app (it's the only thing standing between a logged set and data loss/duplication), so treat it carefully when touching `trySync`, `handleSave`, or `SK.queue`.
- Only one workout can be in progress at a time (`SK.active`); starting a new session while one is active prompts the user to discard it first.

## Screens

`App` renders exactly one of four screens based on `screen` state, plus an always-present bottom `NavBar` and modals (`StartModal` to pick a session date, `ExHistoryModal` for per-exercise e1RM history):

- **home** (`HomeScreen`) — day picker plus the selected day's program card (exercises, warm-up, finisher).
- **session** (`StrengthSession` / `Z4Session` / `Z2Session`, chosen by `activeDay.kind`) — the active logging UI for whichever session is in progress; each set/piece/metric autosaves to `activeSess` state (and thus `localStorage`) on every change.
- **history** (`HistoryScreen`) — read-only, expandable list of past logged sessions with per-exercise top-set and e1RM summaries; no edit or delete.
- **settings** (`SettingsScreen`) — Google Sheets sync setup, including the copy-pasteable Apps Script source (`SCRIPT`) and a URL test/save flow.
