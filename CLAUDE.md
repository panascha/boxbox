# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Architecture

Single-file PWA (`index.html`, ~1990 lines) — a personal training log tracker for iOS/Android home screen. No build step, no npm, no framework. All vanilla JS + CSS inline in one HTML file. Hosted on GitHub Pages (repo: `boxbox`).

**File layout in `index.html`** (line numbers approximate — grep for function names):
- Line 1–723: CSS (design tokens in `:root`, then all component styles)
- Line 772–850: Data layer — `TABLES`, `dataStore` (CRUD), `syncDrive` (Google Drive)
- Line 851–986: `syncDrive` (OAuth connect/disconnect/push/pull/enqueue, GIS popup flow)
- Line 988: `uid()` — ID generation
- Line 986, 1029–1035: `save()` / `load()` — serialize/deserialize state ↔ dataStore
- Line 1051: `cycleInfo()` — cycle phase detection + `ringSVG()` progress indicator
- Line 1166–1736: Render functions: `renderOverview`, `renderTomorrowAdvice`, `renderFeedItem`, `renderCardio`, `renderGym`, `renderExerciseList`, `renderFood`, `renderCycle`
- Line 1737–1900: `renderSettings` (profile form, drive sync UI, export/import, clear data)
- Line 1900–2193: Coach chat — `buildCoachSystemPrompt`, `renderCoach`, `renderChatMessages`, Gemini API calls, photo extraction
- Line 2194–2236: Router (`routers` map → `route(tab)` → `renderXxx()`)

**Data model** — global `state`:
| state field | localStorage key | shape |
|---|---|---|
| `state.profile` | `profile` | `{ name, age, height, currentWeight, weightGoal, deadline, tdee, calorieTarget, proteinTarget }` |
| `state.cardio[]` | `cardio-logs` | `{ id, date, type, distanceKm, durationMin, pace, avgHr, rpe, notes }` |
| `state.gym[]` | `gym-logs` | `{ id, date, dayLabel, exercises: [{ name, sets: [{ reps, weight }] }] }` |
| `state.food[]` | `food-logs` | `{ id, date, time, label, calories, protein }` |
| `state.cycle` | `cycle-settings` | `{ cycleLength, periodLength, starts[] }` |
| `state.eatingSeq` | `eating-seq` | 4 booleans — eating order checklist |
| `state.weightLog[]` | `weight-log` | daily weight entries |
| _(chat history)_ | `coach-chats` | `[{ role, text }]` — loaded in `renderCoach`, not on `state` |

`S.get(k)` returns `{ value: <string> }` or `null`. Deprecated — use `dataStore.get(k)` / `dataStore.set(k, val)` instead.

**`dataStore`** (line 785) — centralized CRUD:
- `TABLES` = `['profile','cardio-logs','gym-logs','food-logs','cycle-settings','eating-seq','weight-log','coach-chats','gemini-keys']`
- `get(k)`, `set(k, val)`, `remove(k)`, `mtime(k)` (epoch ms, 0 if never written)
- `exportAll()` → `{ exportedAt, _mtimes, tables }` — excludes `gemini-keys`
- `importAll(data, merge)` — if `merge=true`, only upgrades per-table where cloud mtime > local
- Every `set()` writes `_mtime:<k>` timestamp automatically
- Line 979: `_dsSet` monkey-patch → auto-enqueues `syncDrive.enqueue()` on every write

**`syncDrive`** (line 851) — Google Drive sync engine:
- Uses Google Identity Services (GIS) OAuth 2.0 popup flow — no redirect server
- Token in memory only (not localStorage). Re-auth each session.
- Scope: `drive.file` — only files this app creates
- Single file `boxbox-backup.json` in user's Drive root
- Auto-sync: 3s debounce after `dataStore.set()` when online + connected
- Manual: push (local→cloud), pull (cloud→local, merge by mtime)
- `navigator.onLine` + `online`/`offline` events gate sync
- OAuth Client ID configured via Settings tab, stored in `_drive_client_id` localStorage
- `gemini-keys` never exported or synced to cloud

**UI**: Tab SPA, 7 tabs: `overview, cardio, gym, food, cycle, settings, coach`. Router maps tab → render function at line 2194. Quick-log buttons on overview only (shown/hidden by `route()`).

**Coach tab** (line 2056): Gemini API chat. `buildCoachSystemPrompt()` builds context from live state (profile, cycle phase via `cycleInfo()`, today's logs, 4-week trend, weight pacing vs deadline). Multi-key rotation on 429s using keys stored under `gemini-keys`. Model: `gemini-3.1-flash-lite` for both chat and photo extraction. Chat history loaded from `coach-chats` localStorage key, capped at last 10 exchanges (send last 20 messages). Proactive suggestion fires on first open if no chat history.

**Photo extraction**: Cardio form has "อ่านจากรูป" button — resizes images to ≤1024px via canvas, sends to Gemini Vision, auto-fills form fields from Strava/treadmill screenshots.

**Duration format**: Internal storage is **decimal minutes**. UI displays as `H:MM:SS` or `M:SS` via `fmtDuration()`. `parseDuration()` accepts `"MM:SS"`, `"H:MM:SS"`, or plain numbers. `stepDuration()` increments by minutes.

**Cycle tracking** (`cycleInfo()` at line 1051): Calculates current day in cycle from last period start date. Phases: Menstruation (≤periodLength) → Follicular → Ovulation (±2 days from ovulationDay = cycleLength - 14) → Luteal. Each phase has assigned intensity advice and badge color (moss/rose). `ringSVG()` renders the cycle progress indicator. `cycleInfoOffset(offset)` (line 1255) for forecast lookahead.

**ID generation**: `uid()` = `Date.now().toString(36) + random 4-char suffix`. Not UUID. Sufficient for local-only use.

**Export/import**: JSON backup/restore of all localStorage keys via Settings tab.

## No build/lint/test commands

Static HTML served by GitHub Pages. To test locally: `python -m http.server` or `npx serve .` then open in browser. No build pipeline.

## Key gotchas

- UI language is **Thai** — labels, toasts, placeholders all Thai. Preserve when editing; ask before translating.
- `handoff.md` is `.gitignore`d intentionally — contains personal health data and working notes.
- Defaults in `defaults` object are **empty strings/nulls** in the repo — actual user values are loaded from localStorage at runtime. Don't hardcode real values there.
- The PWA manifest (`manifest.json`) and icons (`icons/`) enable "Add to Home Screen" on iOS/Android.
- Google Fonts loaded via CDN `<link>` (Space Grotesk + Noto Sans Thai + JetBrains Mono) — works fine on GitHub Pages.
- `state` is a global mutable object — render functions read it directly, save functions write through `save()` which serializes to localStorage. Don't skip `save()` to mutate state without persisting.
- All event handlers are inline (`onclick="..."`) or delegated on parent containers — no framework event system.
- CSS uses design tokens as custom properties: `--moss` (green/positive), `--ember` (orange/alert), `--rose` (red/negative), `--surface`, `--bg` etc.
- Max width: `480px` (mobile-first, centered with `margin: 0 auto`).
- `save()` overwrites `state` from its argument — never mutate `state` fields and pass them to `save()`, or unsaved mutations will be lost. Pass the full object.
- `dataStore.set()` is monkey-patched at line 979 to auto-fire `syncDrive.enqueue()`. Don't replace `dataStore.set` without re-wiring auto-sync.
