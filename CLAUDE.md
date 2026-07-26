# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

This repo (`nana-keke/kakeibo-family`, published via GitHub Pages at
`https://nana-keke.github.io/kakeibo-family/`) is a collection of small, unrelated single-page
apps for personal/family use. There is no shared app shell or router — each `*.html` file is an
independent app:

| File | App |
|---|---|
| `index.html` | 家族の家計簿 (family household budget) |
| `kaji.html` | ふたりの家事メーター (housework split tracker) |
| `kondate.html` | 今日のごはん (meal/shopping-list planner) |
| `grandmother-care-agreement.html` | 祖母への育児委任 すり合わせノート (childcare-delegation notes) |
| `odate-buta-app.html` | おだてブタ2 実戦記録 (pachinko session logger/analyzer) |
| `mission_8_7.html` | standalone decision simulator, no backend |

`odate-buta-app.html` has its own deep-dive doc, `odate-buta-app.README.md` — read it before
touching that app, and treat it as the canonical place to record new design decisions for it
(the same way this file serves the repo as a whole).

## No build system — commands

There is no `package.json`, bundler, linter, or test suite. Each HTML file is fully self-contained
(HTML + CSS + JS inline); the only external dependency is the Firebase compat SDK loaded from a
CDN (except `mission_8_7.html`, which has no backend at all).

- **Syntax-check after editing a file's script**: extract the contents of its `<script>` block
  (excluding the Firebase `<script src=...>` tags) and run `node --check` on it — the app code is
  plain non-module JS, so this validates directly.
- **Preview**: open the HTML file directly in a browser, or serve the repo root with any static
  file server (e.g. `npx serve .`, `python -m http.server`). There is no dev/build step, and no
  separate local vs. deployed behavior other than the URL.

## Shared Firebase backend

All apps except `mission_8_7.html` share **one** Firebase Realtime Database project
(`kakeibo-family-874a4`), using anonymous auth (`firebase.auth().signInAnonymously()`). Because
the project is shared and there is no separate dev/staging database, running an app locally reads
and writes the *same live data* as the deployed version — be careful with destructive changes
(e.g. "delete all data" buttons) while testing.

Each app is namespaced under its own top-level RTDB key(s) so they don't collide:
`entries` / `weeklyBudget` (index.html), `kaji/*` (kaji.html), `shoppingList*` / `mealRecords*`
(kondate.html), `grandmotherAgreement` (grandmother-care-agreement.html), `odatteButaApp/v1`
(odate-buta-app.html).

Sync pattern differs by app:
- `index.html`, `kaji.html`, `kondate.html`, `grandmother-care-agreement.html` bind directly to
  Firebase with realtime `.on("value", …)` listeners — straightforward online-first sync.
- `odate-buta-app.html` is offline-first instead (used in venues with unreliable connectivity):
  it writes to `localStorage` immediately, pushes to Firebase asynchronously, queues failed
  writes and retries on `online`/interval, and merges local vs. remote per-session/per-event using
  `updatedAt` timestamps on load. See its README (§2, §10) before changing this logic.

## Access gating

Several apps implement a lightweight "keep strangers from stumbling onto this" gate rather than
real auth: a required `?key=...` query param unlocks the app and sets a per-app localStorage flag
(`kakeibo-auth`, `kondate-auth`, etc.) so the key isn't needed again on that device/browser. This
is explicitly not real security — don't treat it as an auth boundary when reasoning about data
exposure.

## Data-model conventions worth knowing before editing

These are established in `odate-buta-app.html` (see its README for the full rationale) but reflect
the general philosophy across the family of apps:
- Never delete or auto-migrate old data/event schemas when requirements change; add new
  event types/fields and teach the aggregation/read logic to fold in the old ones.
- Rates/ratios/unit prices are always shown as "numerator / denominator", with a denominator of 0
  rendered as "－" rather than causing a division error.
- Aggregates are recomputed from raw events on read rather than cached and trusted; if a cached
  snapshot is stored, it must be reconstructable from the underlying events.

## Git workflow

There's no CI. Pushing to `main` on `origin` (`https://github.com/nana-keke/kakeibo-family.git`)
is what deploys via GitHub Pages.
