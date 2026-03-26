# Phase 3: Polish

**Date:** 2025-03-25
**Scope:** 2 features — celebration moments and offline resilience for FlowState

## Overview

Phase 3 adds dopamine-friendly feedback loops (confetti + milestone messages) and graceful degradation when the PocketBase backend is unreachable.

## Features

### 1. Celebration Moments

#### Confetti Animation

CSS-only confetti burst. 30-40 small colored squares and circles that scatter from center-top and fall with randomized horizontal positions, rotation, and animation delays. No external libraries.

**Triggers:**
- Completing a deep work task (any zone)
- Completing all tasks for the day (progress bar hits 100%)

**Implementation:**
- A container div `#confetti` is appended to `#app` dynamically when triggered
- Contains 30-40 `<div class="confetti-piece">` elements with randomized inline styles for left position (10-90%), animation-delay (0-0.5s), background-color, and size
- CSS `@keyframes confetti-fall`: starts at top (-10px), moves down to 110vh with horizontal drift and rotation
- Animation duration: ~2 seconds
- Container has `pointer-events: none` and `position: fixed; inset: 0; z-index: 300; overflow: hidden`
- After 2.5 seconds, the container is removed from DOM via setTimeout
- Colors: `#C75450` (red), `#C49A2A` (amber), `#5A9E6F` (green), `var(--cta)`, `var(--accent)` — uses the energy colors plus brand colors

**All-done confetti:**
- Fires when `completedCount === total && total > 0` and `completedCount` just increased
- Larger burst: 40 pieces
- Accompanied by "you crushed it today" message

**Deep work confetti:**
- Fires when completing a task with `energy === 'deep'`
- Smaller burst: 25 pieces
- Accompanied by "deep work done... that's the hard stuff" message

#### Milestone Messages

Reuse `showUndoToast_msg()` for brief auto-dismissing messages (3 seconds, no undo link).

**Milestones tracked per session:**
- `celebratedMilestones` — a `Set()` in JS, not persisted. Resets on page load.
- Keys: `'first'`, `'half'`, `'alldone'`, `'deep-{taskId}'`

**Trigger points and messages:**
| Milestone | Condition | Message |
|-----------|-----------|---------|
| First task | `completedCount === 1 && !celebratedMilestones.has('first')` | "off to a good start" |
| 50% | `completedCount >= total/2 && completedCount < total && !celebratedMilestones.has('half')` | "halfway there... keep going" |
| All done | `completedCount === total && total > 0 && !celebratedMilestones.has('alldone')` | "you crushed it today" |
| Deep work | completing a task with energy='deep', keyed by taskId | "deep work done... that's the hard stuff" |

**Where to hook:**
- In `completeTask()` after `render()` — check milestones against current progress
- In `finishCompletion()` after `render()` is called by the caller — same checks
- The milestone check function reads current Today placements to compute progress

### 2. Offline Resilience (Read-Only Cache)

#### Cache Strategy

- After every successful `loadAll()`, serialize the `data` object to `localStorage` key `flowstate_cache` via `JSON.stringify()`
- Cache is a snapshot — no incremental updates
- On app init, if `loadAll()` throws (PocketBase unreachable), attempt to load from cache

#### Init Flow Change

Current init:
```
seedIfEmpty() -> loadAll() -> render()
```

New init:
```
try:
  seedIfEmpty() -> loadAll() -> render()  (normal online flow)
catch:
  if cache exists in localStorage:
    data = JSON.parse(localStorage.getItem('flowstate_cache'))
    render()
    enterOfflineMode()
  else:
    show error message (current behavior)
```

#### Offline Mode State

- `let isOffline = false` — JS state flag
- When offline mode is active:
  - `isOffline = true`
  - Banner shown below header: "working offline... tap to retry"
  - All mutation controls disabled (see below)

#### Offline Banner

- HTML: `<div id="offline-banner">` placed inside `#app-content`, after `<header>` and before `<main>`
- Hidden by default (`display: none`)
- Shown when `isOffline = true`
- Styling:
  - `background: var(--bg2)`
  - `text-align: center; padding: 8px 16px`
  - `font-family: 'Quicksand'; font-size: 13px; font-weight: 500; color: var(--text-muted)`
  - `cursor: pointer`
  - `border-bottom: 1px solid var(--border)`
- `onclick="retryConnection()"` — calls init again

#### Disabled Mutations in Offline Mode

When `isOffline === true`, the `render()` function applies these restrictions:
- Input bars: hidden (`display: none`)
- Template lightning button: hidden
- "today" / "triage" / "evolve" / delete buttons on cards: hidden
- Card checkboxes: `pointer-events: none; opacity: 0.4`
- Drag handles: hidden
- Start/stop buttons: hidden
- "not today" buttons: hidden
- WSID fab: hidden
- Batch triage button: hidden

Implementation: add a CSS class `body.offline` that hides/disables these elements via CSS rules, rather than conditionally rendering HTML. This way `render()` doesn't need offline-aware branching — just toggle the body class.

```css
body.offline .input-bar,
body.offline .btn-template,
body.offline .card-actions,
body.offline .drag-handle,
body.offline #wsid-fab,
body.offline .btn-triage-all { display: none; }
body.offline .card-checkbox { pointer-events: none; opacity: 0.4; }
```

#### Retry Connection

```javascript
async function retryConnection() {
  try {
    await loadAll();
    exitOfflineMode();
    render();
  } catch(e) {
    // still offline, do nothing (banner stays)
  }
}

function enterOfflineMode() {
  isOffline = true;
  document.body.classList.add('offline');
  document.getElementById('offline-banner').style.display = 'block';
}

function exitOfflineMode() {
  isOffline = false;
  document.body.classList.remove('offline');
  document.getElementById('offline-banner').style.display = 'none';
}
```

#### Cache Freshness

- No expiry — cache is always better than nothing
- Cache is overwritten on every successful `loadAll()`, so it's always the most recent successful state
- If the app loads online successfully, the cache is silently updated in the background

## No Schema Changes

Both features use client-side JS and localStorage only. No PocketBase changes.
