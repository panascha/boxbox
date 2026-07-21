# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Architecture

Single-file PWA (`index.html`) — a personal training log tracker. No build step, no npm, no framework. All logic is inline vanilla JS + CSS in one HTML file. Hosted on GitHub Pages (repo: `boxbox`).

**Persistence**: `localStorage` via a thin wrapper `const S = { get, set, delete }` that JSON-serializes. `S.get(k)` returns `{ value: v }` (or `null`). Keys: `profile`, `cardio-logs`, `gym-logs`, `food-logs`, `cycle-settings`, `eating-seq`, `coach-chats`, `gemini-keys`, `weight-log`.

**Data model** — global `state` object:
- `state.profile` — weight, goals, TDEE, calorie/protein targets, deadline
- `state.cardio[]` — `{ id, date, type, distanceKm, durationMin, pace, avgHr, rpe, notes }`
- `state.gym[]` — `{ id, date, dayLabel, exercises: [{ name, sets: [{ reps, weight }] }] }`
- `state.food[]` — `{ id, date, time, label, calories, protein }`
- `state.cycle` — menstrual cycle tracking: `{ cycleLength, periodLength, starts[] }`
- `state.eatingSeq` — 4 booleans tracking eating order checklist

**UI**: Tab-based SPA (`route(tab)` → `routers[tab]()`). 7 tabs: overview, cardio, gym, food, cycle, settings, coach. Quick-log buttons on the overview tab for 1-click entries.

**Coach tab**: AI chat using Gemini API (`gemini-3.1-flash-lite` for both chat and photo extraction). Multi-key rotation on 429s. System prompt is built from live state (profile, cycle phase, today's logs, 4-week trend summary, weight pacing vs deadline). Proactive suggestion on first open if no chat history.

**Photo extraction**: Cardio form has an "อ่านจากรูป" button that resizes images to ≤1024px, sends to Gemini Vision, and auto-fills form fields from Strava/treadmill screenshots.

**Export/import**: JSON backup/restore of all localStorage keys via Settings tab.

## No build/lint/test commands

This is a static HTML file served by GitHub Pages. No build pipeline. To test locally, open `index.html` in a browser or serve with any static file server (e.g. `python -m http.server` or `npx serve .`).

## Key gotchas

- UI language is **Thai** — labels, toasts, placeholders are all in Thai. Preserve when editing; ask before translating.
- `handoff.md` is `.gitignore`d intentionally — contains personal health data and working notes.
- Defaults in `defaults` object contain real user profile data — do not commit changes that expose or alter these accidentally.
- The PWA manifest (`manifest.json`) and icons (`icons/`) enable "Add to Home Screen" on iOS/Android.
- Google Fonts are loaded via CDN `<link>` — no CSP restrictions on GitHub Pages, so this works fine.
