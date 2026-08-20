# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file client-side web app (`index.html`, ~4400 lines) for a photography business owner (Karin) to manage projects, clients, billing, and a daily task/calendar workflow. Hebrew UI, RTL. No build step, no package manager, no test suite — it's a static HTML file deployed as-is.

`karin-manager-template.html` in the repo root is an old unrelated draft (different color scheme/title) — not part of the live app, don't treat it as a reference.

## Commands

There is no build/lint/test tooling. In practice:

- **Preview locally**: open `index.html` directly in a browser, or serve it (`python3 -m http.server`) if testing anything that needs a proper origin (e.g. `fetch` calls).
- **Syntax-check after editing**: there's no linter, so the working pattern is to extract the main `<script>` block (the one starting after `/* ===== Firebase Sync ===== */` around line 1000) to a temp file and run `node --check path/to/script.js` to catch syntax errors before reloading in the browser.
- **Deploy**: push to `main` — GitHub Pages serves straight from the repo root at `https://karinrav-web.github.io/karin-manager/`. A Vercel deployment also auto-deploys on push but is not the one Karin actually uses.
- The repo **must stay public** — GitHub Pages on this account requires it, and flipping to private silently disables Pages (re-enabling requires manually reselecting the branch under Settings → Pages, it doesn't resume on its own).

## Architecture

Everything lives in one `<script>` block in `index.html`. Structure, in order:

1. **Firebase Sync** (`FB_CONFIG`, `fbSave()`) — Firebase Realtime Database, project `family-meds-c8900` (name is a legacy reuse, not a bug), data at path `karin-manager-data`. `fbSave()` debounces (800ms) and does a full `.set(state)` of the entire app state on every save — there is no partial/field-level write.
   - **Critical gotcha**: Firebase's `.set()` throws *synchronously* if any property anywhere in the object is `undefined` (not a rejected promise — it never reaches `.catch()`). Because saves are whole-state, one bad field (e.g. a project missing `adjItems`) silently kills syncing for *everything*, not just that record. This is why saves go through `JSON.parse(JSON.stringify(state))` first — that strips stray `undefined` keys. Any new `.set()` call site must do the same round-trip.
   - Auth is Firebase Anonymous Auth (`signInAnonymously()`), required by the DB rules (`auth != null` for read/write). This is access control against random internet traffic, not against someone with the PIN — see the password lock note below.
   - `startSessionWatch()` / `SESSION_PATH` is a lightweight "someone else has this open" heartbeat (separate sibling path from `FB_PATH`, so it isn't wiped by the full-state `.set()`). It's advisory only — it warns, it doesn't lock.

2. **Password Lock** — a PIN screen gate (`BOOTSTRAP_HASH`, `hashPin()`). This is enforced client-side only; it is not backed by Firebase rules, so it's a UX deterrent, not real access control. (The DB rules just require *any* anonymous auth session, not a matching PIN.)

3. **App** (`state`, `load()`, `save()`, `render()`) — the core data/render loop.
   - `state = {projects, tasks, settings, editDays, lastReset, billing, credits, clients}` is the single in-memory source of truth.
   - `load()` reads `localStorage` (`karin_pm` key) first for instant render, then signs in to Firebase and attaches a `.on('value', ...)` listener; `mergeCloudState()` reconciles cloud vs. local. `_fbInitialized` guards against saving stale local data over Firebase before the first cloud snapshot arrives.
   - `save()` persists to `localStorage` and also writes a same-day auto-backup under key `karin_ab_YYYY-MM-DD` (separate from the live key) — this is the recovery path if a sync bug ever wipes cloud data; restorable via the sidebar "🔄 שחזר גיבוי אוטומטי" button on the same browser/device.
   - `render()` is a manual full re-render dispatcher — it calls each page's `render*()` function directly (no framework, no virtual DOM, no reactivity). Any state mutation that should reach the screen needs an explicit call into the relevant `render*()`.
   - Page switching (`showPage()` / `showPageMobile()`, separate desktop/mobile nav) toggles `.page`/`.nav-item` classes and re-renders only the page being shown (`report`, `billing`, `projects`, `calendar`).

4. **Billing / Revenue** (from ~line 1916) — the largest and most fragile section: client payments, VAT handling (`isCash` = no-VAT opt-in), supplier costs/payouts, per-project billing history, duplicate-record detection, monthly summaries and charts. Several past bugs here involved: billing rows merging across different projects that happen to share an id, double-VAT display on supplier lines, and auto-generated income records drifting out of sync with manually-edited payer amounts — when touching billing math, check whether a record is auto-generated vs. user-edited before assuming it's safe to recompute.

5. **Dashboard Customize** (from ~line 2716) — user-configurable dashboard widget layout/visibility, persisted in `state.settings`/localStorage.

## Other notable behavior

- **Print/PDF export**: functions build a full standalone HTML document as a template string (see the payment-breakdown export around line 3842) and open it in a new window with `window.onload=()=>window.print()` — there's no PDF library, it relies on the browser's print-to-PDF.
- **External integration**: `syncShootDates()` fetches shoot dates from a separate sibling project (`photo-scheduler-wine.vercel.app`) and auto-marks working days as edit days in the calendar. This is a one-way pull, not part of this repo.
- Data model helper functions to know about: `normalizeProject()` and `dedup()` (dedupe/repair projects and tasks on load), `applyDayReset()` (daily rollover logic), `recalcClientShare()` (per-project client billing recompute called at the top of every `render()`).
