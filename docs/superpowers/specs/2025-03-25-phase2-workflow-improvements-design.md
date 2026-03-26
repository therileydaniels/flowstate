# Phase 2: Workflow Improvements

**Date:** 2025-03-25
**Scope:** 5 workflow improvements to FlowState focused on reducing friction in triage, scheduling, and completion tracking

## Overview

Phase 2 targets workflow bottlenecks: getting items out of inbox faster, planning ahead with templates, making recurring tasks visually distinct on Today, and capturing notes on any completed task. No schema changes needed.

## Features

### 1. Quick-Schedule from Inbox

**UI:**
- Each inbox task card gets a "today" button in the card actions area, styled like existing `schedule-btn` (small text, CTA color)
- Position: before the triage/evolve button and delete button
- Ideas do NOT get a "today" button (they must be evolved first, existing behavior)

**Behavior:**
- Tapping "today" on an inbox task:
  1. Updates the task: zone="backlog", energy defaults to "steady" if empty, priority defaults to "important" if empty, time defaults to "30m" if empty, kind="task"
  2. Creates a placement for today with next sortOrder
  3. Calls `loadAll()` and `render()`
- One tap, no sheet, no confirmation

### 2. Batch Triage (Swipe-Through)

**Entry point:**
- "triage all" button appears at top of inbox view when there are 2+ tasks (kind="task", not ideas) in inbox
- Styled as a secondary button (border, not filled), full width below the input bar

**Panel:**
- Full-screen overlay panel (same pattern as WSID panel)
- Shows one task at a time with:
  - Progress indicator at top: "3 of 8" (Fredoka 600, 14px, text-muted)
  - Task name (large, Fredoka 600, 20px)
  - Energy picker row (same chips as triage sheet)
  - Priority picker row
  - Time picker row
  - Destination buttons row: backlog / pile / today (same as triage sheet dest buttons)
  - "skip" button (small text below destinations, text-muted color)
  - "done" / close button in header

**Behavior:**
- Picker chips work as toggles (tap to select, tap again to deselect)
- Tapping a destination:
  1. Updates the task with selected energy/priority/time (same logic as `triageTo()`)
  2. Auto-advances to next task
  3. Pickers reset to unselected for next task
- "skip" leaves the task untriaged and advances to next
- When all tasks are processed (triaged or skipped), panel closes automatically
- Closing early (X button) is fine — already-triaged items stay triaged
- Ideas are excluded from the batch list

**State:**
- `batchTriageState = { tasks: [], index: 0, energy: null, priority: null, time: null }`
- Not persisted — closing resets

### 3. Pile Task Section on Today

**Visual split:**
- Today active cards are split into two groups:
  - Regular tasks (where task.zone !== 'pile') — rendered first, sorted by sortOrder
  - Section label: "recurring" (Fredoka 600, section-label style) — only shown if pile tasks exist on Today
  - Pile tasks (where task.zone === 'pile') — rendered below label, sorted by sortOrder

**Drag-to-reorder:**
- Reorder works within each group independently
- Dragging a regular task only reorders among regular tasks
- Dragging a pile task only reorders among pile tasks
- The `reorderPlacements()` function filters by group before reindexing

**Card rendering:**
- Both groups use the same card HTML (drag handle, checkbox, start/stop, not today, delete)
- Pile cards keep the "pile" chip in their meta area for extra clarity

### 4. Future Template Scheduling

**UI change in template preview sheet:**
- Replace the single "load onto today" button with a row of 7 day buttons
- Day buttons: "today", "tomorrow", then next 5 days as lowercase day names (e.g., "thursday", "friday", "saturday", "sunday", "monday")
- Styled as picker chips in a flex-wrap row
- Buttons show the date below the day name in smaller text (e.g., "mar 27")

**Behavior:**
- Tapping a day button calls `loadTemplate(id, dateStr)` where dateStr is the selected date
- `loadTemplate()` updated to accept an optional date parameter (defaults to todayStr())
- Placements are created with the selected date
- After loading, switch to Today tab only if the selected date is today
- If future date, close sheet and show a brief confirmation (reuse undo-toast pattern: "loaded for thursday" — auto-dismiss, no undo needed)

### 5. Notes on All Completions (Post-Completion)

**In "done today" section of Today view:**
- Each completed card shows:
  - If no note: small "add note" text link below the card name
  - If has note: italic note text + small "edit" text link
- "add note" / "edit" styled as small text (12px, CTA color, font-weight 600)

**Behavior:**
- Tapping "add note" opens the existing note sheet, pre-populated with empty text
- Tapping "edit" opens the note sheet, pre-populated with the existing note
- Saving updates the completion record's note field via PocketBase
- Need a new function: `openEditNoteSheet(completionId)` that:
  1. Finds the completion in data.completions
  2. Opens sheet with textarea pre-filled with existing note
  3. Save button calls `pb.collection('flowstate_completions').update(completionId, { note: value })`
  4. Refreshes data and re-renders

**In Done view:**
- Done entries that have notes already show them (existing behavior)
- Add same "add note" / "edit" link pattern to done entries that don't have notes yet

## No Schema Changes

All features use existing PocketBase collections and fields:
- `flowstate_tasks` (zone, energy, priority, time, status, kind)
- `flowstate_placements` (taskId, date, source, sortOrder)
- `flowstate_completions` (taskId, placementId, completedAt, note)
- `flowstate_templates` (name, icon, energy, tasks)

## Out of Scope (Phase 3)

- Celebration moments / micro-celebrations
- Offline resilience / caching
