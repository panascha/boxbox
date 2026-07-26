# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

No build, compilation, or linting pipeline exists.
- **Run local server**: `python -m http.server 8000` or `npx serve .`
- **Manual verification**: Open `http://localhost:8000` in browser.

## Architecture

Single-file PWA (`index.html`, ~2667 lines) — a personal fitness & training log tracker for iOS/Android home screens. All vanilla CSS, HTML, and JS, with no external package manager dependencies or framework. Hosted on GitHub Pages.

### File Layout in `index.html`
- **Styles**: CSS design tokens in `:root`, layouts, mobile-first responsive definitions (max-width `480px`, centered).
- **Data Layer & CRUD**: Global mutable `state` synced directly with `dataStore` wrapper around local storage tables (profile, cardio-logs, gym-logs, food-logs, cycle-settings, eating-seq, weight-log, coach-chats, gemini-keys).
- **syncDrive (Google Drive)**: GIS OAuth 2.0 popup sync engine storing a single file `boxbox-backup.json` in the user's Drive root. Enqueues auto-sync with a 3s debounce after `dataStore.set()`.
- **Render Engine**: Hand-authored JS template-literal components rendering into viewports (`renderOverview`, `renderCardio`, `renderGym`, `renderFood`, `renderCycle`, `renderSettings`, `renderCoach`). Simple client-side router (`route(tab)` maps 7 tab IDs to dynamic UI replacements).
- **AI Coach**: All calls route through `callGemini(model, contents, systemInstruction, genConfig)` which is a priority + quota auto-router. Callers pass `'auto'` as model. `MODEL_REGISTRY` defines two tiers (`gen` / `search`) with `{model, rpd, prio}` entries; lower prio picked first. Tier auto-detected from whether caller passes `google_search` tool. Per-device daily quota in `localStorage['model-quota']` (not Drive-synced). 429 → key rotation then next-model fallback; dead/unsupported model → `markModelCap({enabled:false})` permanently skipped. Two photo OCR paths: `extractWorkoutPhoto()` (cardio tracker screenshots → distance/duration/pace/HR) and `extractFoodPhoto()` (menu/Grab/receipt images → food items JSON with calories/protein).

### Data Model & Sync Contract

- **Storage**: Read via `dataStore.get(table)`, write via `dataStore.set(table, data)` which auto-records table mtime timestamp and enqueues Google Drive backup. Do not directly mutate `state` properties without executing `save()` serialization or `dataStore.set()`.
- **Render Pipeline**: Each tab has `render<TabName>()` (e.g., `renderOverview`, `renderCardio`). Returns template-literal HTML injected into `#panel-<tab>`. Never call render functions directly from forms — they fire via `route()` or after `save()`.
- **Save Helpers**: Use `save(table, data)` for synced tables (cardio-logs, gym-logs, food-logs, profile, etc.). Wraps `dataStore.set()` and updates `state.<table>`. Direct `dataStore.set()` only for non-synced keys.
- **Sync Throttle**: Google Drive sync debounces 3s after each `dataStore.set()`. Rapid edits within 3s collapse into one push. Rate limit = 1 sync per 3s window max.
- **Duration Representation**: Stores decimal minutes internally. Renders and accepts standard `"MM:SS"` or `"H:MM:SS"` formats via `fmtDuration()` and `parseDuration()`.
- **Pace Zones**: `zoneBadge(c)` tags cardio entries. Zone 2 = 9:33–10:30 min/km (moss badge), Zone 3+ = <9:33 (ember badge). Hardcoded thresholds — not derived from HR or profile.
- **Cycle Info**: Tracks active phase (Menstruation, Follicular, Ovulation, Luteal) to customize target calorie metrics and AI recommendations.

## Gotchas & Guidelines

- **UI Language**: Always write user-visible labels, error messages, and logs in **Thai** (ภาษาไทย).
- **Secrets**: API keys (`gemini-keys`) are stored in local storage only and explicitly exempted from all export/import functions or Google Drive backups to prevent exposure.
- **Handoffs & State**: A `.gitignore`d `handoff.md` is used to capture context mid-task without saving personal health metrics to history. Keep CLAUDE.md additions minimal to save context token fees.
- **Repomix**: `repomix-output.xml` is gitignored — generated only for AI context, never committed.
