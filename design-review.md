# boxbox — Codebase Design Review

Vocabulary: **module** (interface + implementation), **interface** (everything a caller must know: signature, invariants, error modes, side effects), **seam** (where the interface lives), **depth** (behaviour per unit of interface), **leverage**, **locality**.

**Scope read:** `index.html` lines 823–1130 (dataStore/syncDrive/state), 1700–1960 (photo paths, addCardio), 2100–2165 (food AI), 2476–2680 (model router, callGemini), 2845–3180 (coach chat, chat photo flow), 3178–3251 (route). Render bodies not read line-by-line — claims about render internals come from the code map, not source. Full call-graph trace on key paths checked against source.

**Hard constraints any change must respect:**
1. Inline `onclick="fn()"` in render templates requires top-level `function` decls to stay **global**. Moving a called-from-HTML function into an object literal breaks the app silently. Seams here are function boundaries; if a namespace object is used, the old global name must still exist.
2. `dataStore.set` is monkey-patched at 1032–1033 to add `syncDrive.enqueue()`. Anything restructuring `dataStore` must preserve or absorb that wrapper.
3. No build, no test runner. Verification = manual browser exercise + hand-rolled checks like `parseGeminiJSON_tests` (1684), not a test suite.

---

## What already works — preserve it

**`callGemini(model, contents, systemInstruction, genConfig)` — 2549** is the deepest module in the file and the model to imitate. Four params hide: model registry + priority ordering, per-day quota accounting, key rotation, RPD-vs-RPM 429 discrimination (2605–2615), dead-model benching restricted to 404 + message match (2589–2602), grounding-source extraction, and multi-part text joining that skips `thought` parts (2628–2629). Callers pass `'auto'` and learn nothing else. Deletion test: delete it and that logic reappears in 6 call sites. Keep the interface exactly as is.

`parseGeminiJSON` (1645), `imageToBase64Jpeg` (1708), `dataStore.get/set/mtime` (839) are also correctly shaped: small interface, real work inside.

---

## Finding 1 — Three photo paths reimplement one deep module (highest leverage)

**Sites:** `extractFromPhoto` 1731, `extractFoodPhoto` 1851, `classifyAndExtractPhoto` 2939.

**Identical in all three:** `imageToBase64Jpeg(f, 1024, 0.8)` → `contents = [{role:'user', parts:[{text:prompt},{inlineData:{mimeType:'image/jpeg', data: base64.split(',')[1]}}]}]` → `callGemini('auto', contents, null, {temperature:0.1, maxOutputTokens:2048})` → `parseGeminiJSON` → bespoke retry ladder.

**Legitimately different (the tail, not the core):** where output goes (form fields / storage / draft object), which DOM status node is touched, whether it writes logs, and per-photo vs cross-photo aggregation.

**Accidentally different (drift):** the retry ladders. `extractFoodPhoto` (1885–1905) has a bracket-regex + quote-repair + trailing-comma fallback; `extractFromPhoto` (1785–1788) has a one-shot reformat retry; `classifyAndExtractPhoto` (2966) has none. `aiEstimateNutrition` (2130–2142) has yet a fourth ladder. Same problem solved four ways, three of them weaker.

**Current interface a caller must know:** which of three functions to call, that each takes a *file input element* (not a file), that each mutates specific DOM ids, that `extractFromPhoto` writes to `cardio-logs` only when ≥2 photos, and that `extractFromPhoto` calls `renderCardio()` so it must never be called from the coach tab.

**Proposed interface — the shape already exists:** `classifyAndExtractPhoto(base64) → {kind:'cardio'|'food', entry, items?} | null` is the correct seam. Make the other two thin callers of it. Constraints and decisions:
- **Prompt merging required.** The cardio prompt (1756–1774) carries treadmill hints (`INCL L20`, the pace→distance worked example, date-format normalisation) the unified prompt (2940–2960) lacks. Merge treadmill hints into the unified prompt — dropping them degrades extraction for treadmill photos.
- **Aggregation stays in the caller.** `extractFoodPhoto` sums across photos (1924–1926); `classifyAndExtractPhoto` sums per photo. Shared module returns per-photo results; the food tab keeps its cross-photo sum.
- **Guard threshold: accept the stricter behaviour.** `classifyAndExtractPhoto` returns `null` when `!distanceKm && !durationMin` (2973); `extractFromPhoto` has no such guard and builds an entry coercing missing values to 0. Photos with neither distance nor duration are meaningless entries — the guard is correct. Accepting it means the cardio tab's `extractFromPhoto` will reject photos it currently accepts (silently), which is an improvement.
- **`callGeminiJSON` absorb retry ladders.** New helper `callGeminiJSON(contents, systemInstruction, genConfig)` → `{json, model, text} | {error, raw?}`. Owns the strongest retry ladder (bracket-extract → quote-repair → trailing-comma removal → one-shot reformat). Returns parsed JSON or structured error. `aiEstimateNutrition` also adopts it, retiring the fourth ladder.

**Deletion test:** delete the shared module → the base64/contents/call/parse/retry chain reappears in 3–4 places. It earns its keep. Adapters: 3 callers = a real seam, not a hypothetical one.

### Verification

- Run 2–3 known-good cardio treadmill screenshots and 2–3 known-good food photos through each path. Compare parse success rate and extracted values against baseline (current behaviour).
- Run one borderline photo (blurry receipt) through all paths — `callGeminiJSON`'s stronger retry ladder should match or exceed `extractFoodPhoto`'s current best rate.
- Manual smoke: use all three features normally — cardio form photo upload, food form photo upload, coach chat photo upload. Each should produce correct entries.

---

## Finding 2 — The cardio entry is built three times and has already drifted

**Sites:** 1795–1804 (`extractFromPhoto`), 2974–2983 (`classifyAndExtractPhoto`), 1945–1952 (`addCardio`).

All three produce `{id,date,type,distanceKm,durationMin,pace,avgHr,notes}`. The first two call `paceStr()`; `addCardio` recomputes pace inline (1943–1949) instead. That is drift that has already happened, not hypothetical drift.

**Concrete consequence — live `"M:60"` bug, same bug in both copies:**
```js
`${Math.floor(paceMin)}:${Math.round((paceMin % 1) * 60).toString().padStart(2,'0')}`
```
When the fractional part ≥ 0.99167 (e.g. 9.995 min/km) this renders `"9:60"` instead of `"10:00"`. Present at `paceStr` 1704 and again in `addCardio` 1943–1949. Fixing one copy leaves the other.

**Proposed:** `makeCardioEntry({date,type,distanceKm,durationMin,avgHr,notes}) → entry` as the single constructor; every site calls it. Implementation:
```js
function makeCardioEntry({date, type, distanceKm, durationMin, avgHr, notes}) {
  return {
    id: uid(),
    date: date || todayStr(),
    type: type || 'วิ่ง',
    distanceKm: Number(distanceKm) || 0,
    durationMin: Number(durationMin) || 0,
    pace: paceStr(Number(distanceKm) || 0, Number(durationMin) || 0),
    avgHr: avgHr || null,
    notes: notes || ''
  };
}
```
Also fix `paceStr` (1704) to handle the carry: `const totalSec = Math.round(paceMin * 60);` then recompute mm:ss from totalSec (same pattern as `fmtDuration` 1054–1061).

Same pattern for `makeFoodEntry` — `addFood` 2103–2106 and `classifyAndExtractPhoto` 2991–2998 differ on whether `calories`/`protein` are numbers (classify) or raw input strings (addFood). Standardise on numbers; callers that read from form inputs coerce first.

**Deletion test:** delete it → three constructors, and the drift resumes.

### Verification

- Console test: `makeCardioEntry({date:'2026-08-10', distanceKm:5, durationMin:49.99})`. Pace must not be `"9:60"`; must be `"10:00"`.
- Existing flow: `addCardio()` via cardio tab — UI unchanged, entry saved correctly.
- Photo flow: both `extractFromPhoto` (1-photo fill-form and 2+-photo save) and coach `classifyAndExtractPhoto` produce correct entries.

---

## Finding 3 — Non-render modules reach into the render layer

**Sites:** `syncDrive.connect` 949, `.disconnect` 961, `.push` 987/999, `.pull` 1005/1016/1019 → call `renderSettings()`, and `pull` additionally calls `route('overview')`. `extractFromPhoto` 1819/1839 → `renderCardio()`.

`syncDrive.push()`'s real interface includes "repaints the Settings tab over whatever tab you are on." That fact is nowhere in its signature — it lives in the code map as a *trap* ("any `render*()` call wipes the chat", "never call `extractFromPhoto` from the coach tab"). A trap in a memory file is a symptom of an interface that lies.

**Confirmed bug, not latent.** Traced in source: `confirmChatDraft` 3127–3128 calls `save('cardio-logs', …)` from the coach tab → `save` 1041 → `dataStore.set` → wrapper 1033 → `syncDrive.enqueue()` 919–923 → 3s debounce → `push()` 984 → `renderSettings()` 987, which writes `document.getElementById('content').innerHTML` at 2253 — the same node `renderCoach` owns (2757). Repro: connect Drive, open Coach, send a photo, confirm the draft, wait 3s → chat is replaced by Settings. Precondition is "Drive connected", the feature's normal state, not an edge case. `push()` repaints twice (987 and 999); `pull()` additionally calls `route('overview')` at 1016.

**Two viable fixes, pick one:**

**Option A (simpler, 1-line):** Patch the `dataStore.set` wrapper to skip sync enqueue for coach-chats writes.
```js
// 1033, replace:
dataStore.set = function(k, val) { _dsSet.call(dataStore, k, val); if (navigator.onLine) syncDrive.enqueue(); };
// with:
dataStore.set = function(k, val) { _dsSet.call(dataStore, k, val); if (navigator.onLine && k !== 'coach-chats') syncDrive.enqueue(); };
```
This stops the immediate wipe path (`confirmChatDraft` → `save('cardio-logs')` on line 3127 is not the trigger; the 3s-delayed `push()` is). But wait — `save('cardio-logs')` at 3127 *does* enqueue too. Option A gates on the table key, but the wipe-triggering save is `save('cardio-logs')` and `save('food-logs')`, not `coach-chats`. So this fix is wrong for this path.

Actually traced: `confirmChatDraft` saves `cardio-logs` (3127) and `food-logs` (3128) — not `coach-chats`. These enqueue a sync. 3s later `push()` fires → `renderSettings()` → wipes `#content`. Gating on `coach-chats` alone misses both. So Option A doesn't work as stated.

**Option A corrected (broader, still small):** Gate all sync enqueues on *not being in the coach tab*:
```js
dataStore.set = function(k, val) {
  _dsSet.call(dataStore, k, val);
  if (navigator.onLine && document.querySelector('.tab-btn.active')?.dataset.tab !== 'coach')
    syncDrive.enqueue();
};
```
This prevents the deferred push from any save that originates while the coach tab is active. Downside: if the user switches tabs before the 3s debounce fires, the enqueue is missed (acceptable — next save picks it up).

**Option B (the original proposal, structural):** Keep `dataStore.set` wrapper unchanged. Instead gate the render side:
```js
// Helper, placed near routers (3185):
const activeTab = () => document.querySelector('.tab-btn.active')?.dataset.tab;
const refreshIfActive = (tab) => { if (activeTab() === tab) routers[tab](); };
```
Replace bare `renderSettings()` in `syncDrive.push` (987, 999) and `.disconnect` (961) with `refreshIfActive('settings')`. Replace `renderCardio()` in `extractFromPhoto` (1819, 1839) with `refreshIfActive('cardio')`. `pull()`'s `route('overview')` (1016) becomes `if (activeTab() === 'overview') route('overview')`. `syncDrive.connect`'s `renderSettings()` (949) is called directly by user action on the Settings tab itself — no risk of cross-tab wipe, leave as-is.

Option B is correct regardless of which table triggers the save. Option A is reactive to this specific path but brittle to new callers. **Recommend Option B** — it fixes the class of bug (non-render module calling render unconditionally), not just this instance.

**Deletion test:** remove the render calls → the decision reappears in exactly one place, and current behaviour in the coach tab is already wrong. Passes cleanly.

### Verification

- Repro from design-review: connect Drive, open Coach, send photo, confirm draft, wait 5s. Chat must remain visible. Settings tab must still repaint when selected manually.
- On cardio tab: photo upload → single photo fills form (renderCardio fires), multi-photo saves and refreshes list. Behaviour unchanged from current.
- Drift check: `confirmChatDraft` at 3127 also calls `save('cardio-logs')` and `save('food-logs')` — these enqueue sync via the wrapper. Post-fix, `push()` calling `refreshIfActive('settings')` is a no-op when not on settings tab.
- `syncDrive.pull()` manual sync from Settings tab still routes to overview (normal UX). From any other tab, it's a silent no-op on the render side — data still loads into state.

---

## Finding 4 — `state` + `save()` + `dataStore` is a three-way interface for one operation

Every write site does:
```js
state.cardio.push(entry);
save('cardio-logs', state.cardio);
```
The caller must know: the `state` key, the table name, the mapping between them, and that mutating `state` alone does not persist. That mapping is already written twice — `loadAll()` (1092–1101) and `delEntry`'s local `map` (3179).

**Undocumented interface fact:** `save()` (1041) swallows every exception into a toast and returns nothing. On QuotaExceededError the whole table write is dropped silently. Confirmed path for chat: `saveChatHistory` 2444–2453 → `save('coach-chats', persist)` → `dataStore.set` → `enqueue()` → Drive push of `exportAll()`, so a dropped local write is what gets backed up. That is why 2447–2452 strips `isDraft` and `imgs`. Today that fact lives only in a memory file.

**Proposed:** one table registry `{table, stateKey, default}` derived from `TABLES`/`DEFAULTS` (826–837), and `addEntry(kind, entry)` / `removeEntry(kind, id)` over it. `loadAll` and `delEntry` become loops over the registry. Make quota failure loud — `save()` should return `false` and callers of chat/photo writes should surface it, since silent loss is the failure mode that already bit this codebase. Any new user-visible message must be Thai, per CLAUDE.md.

Minimum viable change if full registry is too much: just make `save()` return boolean:
```js
function save(key, val) {
  try { dataStore.set(key, val); return true; }
  catch (e) { toast('บันทึกไม่สำเร็จ ลองใหม่'); return false; }
}
```
Then audit the 2–3 callers whose failure matters most (`saveChatHistory`, `confirmChatDraft`).

**Deletion test:** delete the registry → the mapping is duplicated in 3+ places (it already is, in 2).

### Verification

- Manual: simulate QuotaExceeded by filling localStorage near capacity, attempt a save. Toast must appear. `save()` must return `false`.
- `confirmChatDraft`: if `save()` returns false, draft must not be cleared from `state.pendingChatDraft`, and an error toast must be visible.
- Normal flow: all saves succeed, no regression in any tab.

---

## Finding 5 — `callGemini`'s one wart: the `{error} | {text}` union

Every caller re-implements `if (result.error)` and then its own JSON handling: 1779, 1875, 2127, 2874, 2927, 2964. Six error branches, four retry ladders, three different "give up" behaviours. The router is deep; the *result type* is shallow and leaks work to callers.

Fixing this is the `callGeminiJSON` seam in Finding 1. Text-only callers (`sendCoachMsg` 2924, `sendProactiveSuggestion` 3167) keep using `callGemini` unchanged.

### Verification

- `callGeminiJSON` returns `{json, model, text}` for well-formed JSON responses.
- `callGeminiJSON` returns `{error, raw?}` for parse failures after retry ladder exhausted.
- All JSON-using callers (3 photo paths + `aiEstimateNutrition`) produce identical outputs for known-good inputs and better parse rates for borderline inputs.

---

## Leave alone — one adapter means a hypothetical seam

- **Storage backend.** localStorage is the only adapter and always will be. No `StorageAdapter` interface.
- **Render engine.** Template literals into `#content` work; one adapter. Do not introduce a component abstraction. (Finding 3 is about *who calls* render, not about replacing it.)
- **`syncDrive` as a provider abstraction.** Google Drive is the only backend. Keep it concrete.
- **`MODEL_REGISTRY` / quota / caps split across `_loadCaps`, `loadModelQuota`, `routerCandidates`.** These are internal seams behind `callGemini`; callers never see them. Correct as is.
- **`cycleInfo`, `computeWeeklyMetrics`, `generateWeeklyRecommendation`.** Pure functions in, data out — already the testable shape.
- **`zoneBadge` hardcoded thresholds.** Personal app, one user; parameterising is premature.
- **Splitting `index.html` into modules/files.** No build step, GitHub Pages, inline handlers. Cost far exceeds benefit.
- **`escapeHtml` / `mdToHtml` / `toast` / `uid` / `fmt*` helpers.** Small, single-purpose, correct.
- **`buildCoachSystemPrompt`.** Long, but it is one prompt with one caller shape — depth without a seam problem.

---

## Order of work

1. **Finding 3** — confirmed bug, not latent. Normal-use repro (connect Drive, confirm photo in chat, wait 3s → chat wiped). Fix with Option B (`refreshIfActive` helper, ~6 line change). Removes two code-map traps.
2. **Finding 2** — live `"M:60"` pace bug in two copies. `makeCardioEntry` + fix `paceStr`'s rounding.
3. **Finding 1 + 5** — largest leverage, biggest diff. `callGeminiJSON` first (shared retry ladder), then fold three photo paths onto `classifyAndExtractPhoto`. Merge treadmill hints into unified prompt. Accept the stricter zero-guard. `aiEstimateNutrition` adopts `callGeminiJSON`.
4. **Finding 4** — minimum: make `save()` return boolean, audit caller handling. Full: table registry with `addEntry`/`removeEntry`. Skip if appetite is limited.

Every step keeps all HTML-referenced functions global (constraint 1) and leaves the `dataStore.set` wrapper (constraint 2) intact. Finding 3 Option B does not touch the wrapper at all.

**Verdict: fix-then-ship.** Confirmed chat-wipe bug on normal-use path (Finding 3), live `"M:60"` pace formatting bug (Finding 2), and 3× duplicated AI/photo logic with inconsistent error handling (Findings 1+5). Total surface: ~6 small edits, no new dependencies, no build changes.
