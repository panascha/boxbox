# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

No build, compilation, or linting pipeline exists.
- **Run local server**: `python -m http.server 8000` or `npx serve .`
- **Manual verification**: Open `http://localhost:8000` in browser.

## Architecture

Single-file PWA (`index.html`, ~2500 lines) — a personal fitness & training log tracker for iOS/Android home screens. All vanilla CSS, HTML, and JS, with no external package manager dependencies or framework. Hosted on GitHub Pages.

### File Layout in `index.html`
- **Styles**: CSS design tokens in `:root`, layouts, mobile-first responsive definitions (max-width `480px`, centered).
- **Data Layer & CRUD**: Global mutable `state` synced directly with `dataStore` wrapper around local storage tables (`['profile','cardio-logs','gym-logs','food-logs','cycle-settings','eating-seq','weight-log','coach-chats','gemini-keys']`).
- **syncDrive (Google Drive)**: GIS OAuth 2.0 popup sync engine storing a single file `boxbox-backup.json` in the user's Drive root. Enqueues auto-sync with a 3s debounce after `dataStore.set()`.
- **Render Engine**: Hand-authored JS template-literal components rendering into viewports (`renderOverview`, `renderCardio`, `renderGym`, `renderFood`, `renderCycle`, `renderSettings`, `renderCoach`). Simple client-side router (`route(tab)` maps tab IDs to dynamic UI replacements).
- **AI Coach**: Gemini API integrations under Coach tab using `gemini-3.5-flash-lite`. Builds customized Thai fitness insights by feeding personal stats, menstrual cycle phase (`cycleInfo()`), trend history, and food entries to the model. Also includes a canvas-based photo extraction tool parsing workout dashboard photos.

### Data Model & Sync Contract

- **Storage**: Read via `dataStore.get(table)`, write via `dataStore.set(table, data)` which auto-records table mtime timestamp and enqueues Google Drive backup. Do not directly mutate `state` properties without executing `save()` serialization or `dataStore.set()`.
- **Duration Representation**: Stores decimal minutes internally. Renders and accepts standard `"MM:SS"` or `"H:MM:SS"` formats via `fmtDuration()` and `parseDuration()`.
- **Cycle Info**: Tracks active phase (Menstruation, Follicular, Ovulation, Luteal) to customize target calorie metrics and AI recommendations.

## Gotchas & Guidelines

- **UI Language**: Always write user-visible labels, error messages, and logs in **Thai** (ภาษาไทย).
- **Secrets**: API keys (`gemini-keys`) are stored in local storage only and explicitly exempted from all export/import functions or Google Drive backups to prevent exposure.
- **Handoffs & State**: A `.gitignore`d `handoff.md` is used to capture context mid-task without saving personal health metrics to history. Keep CLAUDE.md additions minimal to save context token fees.
