# Phase 1: Core UX Wins

**Date:** 2025-03-25
**Scope:** 6 improvements to FlowState focused on daily usability for ADHD workflows

## Overview

Phase 1 targets the highest-impact UX gaps in FlowState's Today view: giving users control over task order, visibility into what they're working on, forgiveness for mistakes, and honest time awareness. Also fixes a doc drift between CLAUDE.md and the actual color scheme.

## Features

### 1. CLAUDE.md Color Sync

Update the Design System section in `CLAUDE.md` to match the actual purple/indigo theme currently in `index.html`.

Current (wrong) values in CLAUDE.md use rose/pink (`--cta: #C0396B`, `--bg: #F5E6E8`).
Correct values from the live app use purple/indigo (`--cta: #5E5BAE`, `--bg: #F0EEFC`).

Replace all CSS variable values, dark mode values, and CTA glow colors to match the actual code. Font stack and component specs remain unchanged.

### 2. Drag-to-Reorder on Today

**Data model change:** Add `sortOrder` (number) field to `flowstate_placements` collection in PocketBase. Default: 0.

**Behavior:**
- Today view sorts active (uncompleted) cards by `sortOrder` ascending instead of by priority
- Each active card gets a drag handle (grip dots `\u2261`) on its left side, before the checkbox
- Dragging a card to a new position updates `sortOrder` for all affected placements via PocketBase
- New placements get `sortOrder` = max existing sortOrder + 1 (appended to bottom)
- Uses HTML5 drag-and-drop with `touchstart`/`touchmove`/`touchend` handlers for mobile support
- No external libraries... inline touch-drag implementation
- Sort order is inherently per-day since placements are per-day

**PocketBase schema change:**
- Collection: `flowstate_placements`
- New field: `sortOrder` (number, default 0)

**Migration:** Existing placements keep sortOrder=0. On render, if multiple placements share the same sortOrder, fall back to creation order (PocketBase ID sort) as tiebreaker.

**Code change in `loadAll()`:** The placements mapping must include `sortOrder: p.sortOrder || 0` so the field is available in the local cache.

### 3. Active Task Indicator

**State:** `activeTaskId` and `activeStartTime` in JS memory. Not persisted... refreshing clears active state.

**UI on Today cards:**
- Each active (uncompleted) card gets a "start" button (play triangle) in the card actions area
- Tapping "start":
  - Sets this task as active (only one at a time, starting another auto-stops the previous)
  - Highlights the card: 3px left border in the task's energy color, subtle background tint
  - Starts an elapsed timer displayed on the card meta area (mm:ss format)
- Active card shows "stop" button (square icon) instead of "start"
- Tapping "stop" clears active state, removes highlight and timer
- Timer ticks every second via `setInterval`, cleared on stop/completion/tab switch

**Styling:**
- Active card: `border-left: 3px solid {energy-color}`, background gets a very faint energy-color tint
- Timer text: Fredoka font, 13px, energy color, displayed inline in card-meta

### 4. Undo Toast on Completion

**Behavior:**
- After completing any task (checkbox tap), a toast appears above the tab nav bar
- Toast text: "done! undo?" where "undo" is a tappable link
- Auto-dismisses after 5 seconds with a fade-out animation
- Tapping "undo" calls existing uncomplete logic (`uncompleteTask()` for placed tasks)
- Only one toast at a time... new completion replaces the previous
- Toast stores the completion info needed to undo (placementId)

**Styling:**
- Fixed position, bottom: 72px (above tab nav), centered, max-width: 320px
- Background: var(--card), border: 1px solid var(--border), border-radius: 12px
- Shadow: 0 4px 16px var(--shadow)
- Text: Quicksand 500, 14px
- "undo" link: var(--cta) color, font-weight 600
- Fade-in on appear (0.2s), fade-out before dismiss (0.3s)

### 5. "Not Today" Button

**UI:**
- New text button in card actions on Today active (uncompleted) cards
- Label: "not today", styled like existing schedule-btn (small text, CTA color)
- Position: between card body and delete button: [...card...] [not today] [x]

**Behavior:**
- Deletes the placement record for today
- Task stays in its original zone:
  - Backlog tasks: placement removed, task remains zone=backlog
  - Pile tasks: placement removed, task status set back to "available"
- No confirmation needed... action is easily reversible (re-schedule from backlog/pile)
- After removing, calls `loadAll()` and `render()` to refresh

### 6. Time Estimate Total on Today

**Display:**
- New line below the progress bar showing estimated remaining time
- Format: "~Xhr Ym left" (e.g., "~1hr 45m left", "~30m left", "~3hrs left")
- Only counts active (uncompleted) placements that have a time estimate
- Time conversion: 15m=15, 30m=30, 1hr=60, 2hr+=120. Sum minutes, format as hours+minutes.
- If no active tasks have time estimates, line is hidden
- If all tasks are done, line is hidden

**Styling:**
- Same as progress-label: Fredoka 600, 13px, var(--text-muted)
- Displayed on a new line below the progress bar div, left-aligned with 4px left padding

## Out of Scope (Phase 2+)

- Quick-schedule from inbox
- Pile task visibility improvements on Today
- Future date template scheduling
- Batch triage mode
- Notes on all completions (not just pile)
- Celebration moments
- Offline resilience

## PocketBase Schema Changes Summary

| Collection | Field | Type | Default | Notes |
|---|---|---|---|---|
| flowstate_placements | sortOrder | number | 0 | New field for drag-to-reorder |

**Action required:** Add `sortOrder` number field (default 0) to `flowstate_placements` in the PocketBase admin UI before deploying the updated HTML.
