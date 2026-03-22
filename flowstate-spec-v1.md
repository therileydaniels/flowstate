# FlowState — Product Spec v1.0
### A chill, energy-aware task management app

---

## How This Spec Works With Other Docs

This spec is the **product document**. It explains what FlowState is, why it works the way it does, and what every feature should feel like. Read this to understand the app.

The **Claude Code Build Prompt** (`flowstate-claude-code-prompt.md`) is the implementation document. It tells Claude Code exactly what to build, including the CLAUDE.md that gets created in the project. The CLAUDE.md is the technical reference that lives inside the codebase.

If the spec and the CLAUDE.md ever disagree, the spec wins. The CLAUDE.md is derived from the spec, not the other way around.

---

## Concept

FlowState is a personal task management app built around one honest question:
**"What can I actually do right now?"**

Rather than imposing deadlines and urgency, it uses **priority × energy** as its decision engine... a relaxed take on the Eisenhower Matrix where you match tasks to your current capacity, not a countdown clock. No overdue flags. No guilt. Just a clear, honest list of what's available to you in this moment.

It's built specifically for someone with ADHD who:
- Forgets tasks exist if they're not visible
- Has wildly variable energy levels throughout the day
- Gets paralyzed by long lists and too many options
- Needs low-friction capture so nothing gets lost
- Benefits from structure that doesn't punish you for ignoring it

---

## Core Philosophy

| Traditional Task App | FlowState |
|---|---|
| Deadline-driven | Priority-driven |
| Urgent vs. Important | Energy vs. Priority |
| Flags missed tasks | Never nags |
| Schedules everything | Holds options, you choose |
| One-dimensional lists | Four zones with purpose |
| Habits = streaks | Habits = quiet dots |
| "You missed 3 tasks" | "What feels right today?" |

---

## The Decision Engine

Every task carries two attributes that power the core filter:

### Energy Level (what the task requires of you)

| Label | Color | Icon | Meaning |
|---|---|---|---|
| Deep Work | #C75450 (red) | 🔴 | High focus, creative, cognitively demanding |
| Steady | #C49A2A (amber) | 🟡 | Routine, moderate attention, familiar work |
| Easy | #5A9E6F (green) | 🟢 | Autopilot, admin, low-stakes, filling time |

### Priority (how much it matters)

| Label | Meaning |
|---|---|
| First | Blocking something, or has been waiting too long |
| Important | Real work, needs to happen, no rush |
| Whenever | Nice to do, low stakes |

### Time Estimate

Buckets, not exact times: `15m` · `30m` · `1hr` · `2hr+`

These are honest estimates, not commitments. A task with no time estimate is treated as "fits anything" by the filter... it's never excluded.

---

## The Five Zones

### 📥 Inbox
Raw capture. Zero friction. Brain dump landing zone.

- Add a task with just a name... no tagging required
- Supports **smart entry**: type `"edit thumbnails easy 30 min"` and energy + time are auto-parsed from the text via pattern matching (not AI, just regex)
- Supports **idea capture**: type `"idea: collab with another creator"` and it's tagged as an idea, not a task
- Sits unprocessed until you triage it
- Triage options: assign energy, priority, time estimate → send to Backlog, Pile, or directly onto Today

**Ideas vs. Tasks:**
Ideas are seeds. They're not actionable yet... just thoughts worth holding onto. They show as visually distinct cards (dashed border, 💡 icon, italic text) so they don't get confused with tasks. When an idea is ready, you tap "evolve" and it converts to a task, opening the triage sheet so you can tag and route it.

The inbox has a filter toggle: `tasks` | `ideas` | `all`. Default is tasks.

### 🪣 Pile
Permanent standing tasks. Never removed, only cycled.

These are tasks that are always part of your work... editing, responding, reviewing... they don't have a final end state, they just get done periodically.

**Three states:**
```
Available (in the Pile) → Scheduled (on a day) → Completed (logged, returns to Available)
```

- When scheduled, the task disappears from the Pile (only one instance at a time, no double-scheduling)
- When completed, a **note capture moment** appears: "task done ✓... add a note?" with `save` and `skip` options
- Completion is logged with timestamp + note, then the task quietly returns to Available
- Over time, completions form a lightweight work journal
- Pile tasks can be set to any priority level (First, Important, Whenever)

**Habit dots:** Each Pile task shows a row of 7 tiny dots (one per day of the last week). Filled dot = completed that day. Empty dot = not done. Hovering a filled dot shows the note if one was added. No streak counts, no color-coding the misses. Just quiet rhythm awareness.

### 🗂 Backlog
Ready tasks waiting to be picked up or scheduled.

- Fully tagged: energy, priority, time estimate (all set during creation or triage)
- Surfaced by the "What Should I Do?" filter
- Can be scheduled onto Today
- **One-and-done:** completing a backlog task moves it to zone "done" permanently. It does not reappear the next day. It lives on only in the Done view's completion history.

### ✓ Done
Completion history. A quiet record of what you've accomplished.

- All completions grouped by date, newest first
- Date headers: "today", "yesterday", day of week for last 7 days, then "Mar 15" format
- Energy filter at the top (all / deep work / steady / easy)
- Each entry shows: task name, energy chip, timestamp, and the note if one was saved
- Notes display as indented quote blocks with a left accent border
- Running count at the top ("12 things completed")

### 📅 Calendar (Future... Phase 2)
Not built in V1. Planned as three zoom levels: Month (read-only overview), Week (planning mode with template drag), Day (execution mode with block expansion). See "Future Phases" section.

---

## The "What Should I Do?" Filter

The heart of the app. Designed for the couch moment... especially on mobile.

Accessible from any view via a floating ✦ button (bottom-right corner, subtle pulse animation). Tapping it opens the WSID view.

**You set:**
- Current energy level (one tap, toggleable)
- Available time (one tap, toggleable)

**It shows:**
- Matching tasks from Today (uncompleted) + Backlog + Available Pile tasks
- Sorted by priority (First → Important → Whenever)
- Tasks with no time estimate always pass the time filter
- No duplicates (if a task appears in both Today and Backlog, it shows once)

**"Just Pick One" Mode:**
A toggle button below the filters. When active, the list collapses to a single focused card:

- Large centered task name in Fredoka heading font
- "how about this" label with a gentle breathing animation
- Energy chip + priority badge + time estimate below
- Two action buttons: "let's go" (schedules to Today, switches to Today view) and "nah, next" (skips it, shows the next highest-priority match)
- If you skip everything: "you skipped them all... maybe rest is the move"
- Changing energy or time filters resets the skip list

This mode exists specifically for ADHD paralysis. When even a short list feels like too many decisions, this cuts it to one.

---

## Templates

Templates are pre-built groups of tasks that can be loaded onto Today in one tap. They represent recurring work sessions.

**Accessing templates:** ⚡ button next to the quick-add input on the Today view. Opens a Template Tray (bottom sheet).

**Template tray flow:**
1. See list of templates with icon, name, description, task count, and energy chip
2. Tap to preview (shows all tasks with their energy levels and time estimates)
3. "load onto today" creates all tasks as backlog items with today placements
4. Tray closes, Today view shows the loaded tasks

**Pre-built templates (from actual business workflows):**

| Template | Icon | Energy | Tasks | Description |
|---|---|---|---|---|
| Tuesday Fulfillment | 📬 | Steady | 6 | Weekly fulfillment session: game DMs, prizes, sub extensions, top spender messages, content posting, mass DM, name moan signups |
| Filming Day | 🎬 | Deep | 6 | Monthly content batch: review shots, set up gear, film personalized items, customs, scheduled videos, backup footage |
| Admin Hour | 📋 | Easy | 4 | Low-energy sweep: DMs, notifications, tracking spreadsheets, file organization |
| Edit Session | ✂️ | Deep | 4 | Batch editing: edit + watermark, cut teasers, add outros, export by platform |
| Content Planning | 🗓 | Deep | 5 | Monthly strategy: review performance, plan games, plan streams, update calendar, plan filming |
| Upload Day | 📤 | Steady | 5 | Push content: ManyVids, Clips4Sale, OF VIP, OF Free, tube site teasers |

Templates are hardcoded in V1. Custom template creation is a future feature.

---

## Smart Entry

Natural language parsing on task input. Matches against the app's own label set via regex. Not AI... just pattern matching.

**Energy patterns recognized:**
- "deep work", "deep", "focus", "focused" → Deep Work
- "steady", "routine", "moderate" → Steady
- "easy", "simple", "quick", "autopilot", "low" → Easy

**Time patterns recognized:**
- "15 min", "15m", "15" → 15m
- "30 min", "30m" → 30m
- "1 hr", "1hr", "1 hour", "60 min" → 1hr
- "2 hr", "2hrs", "2hr+", "3h", "4h" → 2hr+

**Idea prefix:**
- "idea:", "idea " → tagged as idea instead of task

**Examples:**
- `"edit thumbnails easy 30 min"` → name: "edit thumbnails", energy: Easy, time: 30m, kind: task
- `"write script deep work 2 hrs"` → name: "write script", energy: Deep Work, time: 2hr+, kind: task
- `"reply to emails 15"` → name: "reply to emails", time: 15m, energy: null (unset)
- `"idea: try a pick your adventure tip game"` → name: "try a pick your adventure tip game", kind: idea

Parsed labels are stripped from the task name. Multiple spaces are collapsed. Form-based input is always available as a fallback for precision.

---

## Today View

The daily execution surface. Shows what's on your plate right now.

**Layout (top to bottom):**
1. **Quick-add input** with smart entry support. Defaults to Steady/Important/30m for anything not parsed. The "go" button only appears when there's text in the field. Next to the input: ⚡ button to open Template Tray.
2. **Progress bar:** "3 of 7" with a fill bar. Turns green with "all done ✓" when everything is checked off.
3. **Active task cards:** Checkable cards with energy-colored checkbox borders. Tapping the checkbox completes the task. Tapping the card content opens the edit sheet.
4. **Done section:** Appears below active tasks when at least one is completed. Faded cards with strikethrough text and green checkmarks.
5. **End-of-day summary:** Appears after 6 PM if at least one task was completed. "you got 6 things done today... 2 deep work, 3 steady, 1 easy." Centered, soft background, not intrusive.

**Completion behavior:**
- Backlog/inbox tasks: immediately marked done, moved to zone "done"
- Pile tasks: opens the note capture sheet first, then marks done and returns to Available

---

## UX Patterns

### Two-tap delete
The × button on every card uses a two-tap confirmation pattern. First tap turns it into a red "sure?" button. If you tap again within 2.5 seconds, the task is deleted. If you don't, it resets to ×. This replaces `confirm()` which is silently blocked in iframe environments.

### Edit sheet
Tapping the content area of any task card (on Today, Backlog, or Pile) opens a bottom sheet where you can edit the name, energy, priority, and time. Same visual pattern as the triage sheet.

### Bottom sheets
All modals use the bottom sheet pattern: slides up from the bottom, rounded top corners (24px), backdrop blur, drag handle at top. Used for triage, edit, note capture, and template tray. Tapping the overlay closes the sheet.

### Navigation
Five tabs: Today (◉), Inbox (↓), Pile (↻), Backlog (☰), Done (✓). Compact labels with icons. Scrollable on small screens with a hidden scrollbar and a right-edge fade gradient as a scroll hint. Badge counts on Today (active tasks), Inbox (tasks + ideas), Pile (available), and Backlog (total).

### Floating WSID button
Present on every view except WSID itself. Bottom-right corner, 56×56px, rounded square (16px radius), subtle pulse animation. Tapping it opens WSID with all filters cleared.

---

## Data Model

All data lives in localStorage under the key `flowstate_v1`. The value is a JSON object with three arrays:

### tasks
```
{
  id: string,           // random 8-char alphanumeric
  name: string,
  zone: string,         // "inbox" | "backlog" | "pile" | "done"
  energy: string|null,  // "deep" | "steady" | "easy" | null
  priority: string|null,// "first" | "important" | "whenever" | null
  time: string|null,    // "15m" | "30m" | "1hr" | "2hr+" | null
  status: string|null,  // "available" | "scheduled" | null (pile tasks only)
  kind: string          // "task" | "idea"
}
```

### placements
```
{
  id: string,
  taskId: string,       // references tasks.id
  date: string,         // "YYYY-MM-DD"
  source: string        // "manual" | "template"
}
```

### completions
```
{
  id: string,
  taskId: string,       // references tasks.id
  placementId: string|null,
  completedAt: string,  // ISO 8601 datetime
  note: string          // empty string if skipped
}
```

### Key behaviors
- **Double-scheduling prevention:** Before creating a placement, check if one already exists for that task + today's date.
- **Backlog one-and-done:** When a backlog task is completed, its zone changes to "done". It never returns to the backlog.
- **Pile cycle:** When a pile task is completed, its status resets to "available" and its today placement is removed. The completion record persists in the completions array.
- **WSID null time handling:** Tasks with no time estimate (`time: null`) always pass the time filter. They're never silently excluded.
- **Triage to Today:** When triage destination is "Today", the task's zone is set to "backlog" AND a placement is created for today's date. Both must happen.

---

## Design System: Soft Periwinkle

### Palette (CSS variables)
```css
:root {
  --bg:        #F0EEFC;   /* page background */
  --bg2:       #E0DCF8;   /* section backgrounds, secondary surfaces */
  --cta:       #5E5BAE;   /* primary buttons, links, active states */
  --cta-hover: #4E4A9A;   /* button hover */
  --accent:    #A8A6D8;   /* tags, highlights, selection */
  --success:   #8AAEAA;   /* completion, positive states */
  --text:      #2A2850;   /* primary text */
  --text-muted:#6A68A0;   /* labels, timestamps, helper text */
  --border:    #C8C4EC;   /* borders, dividers */
  --card:      #FFFFFF;   /* card surfaces */
}
```

Energy colors are hardcoded (functional, not brand):
- Deep Work: `#C75450` on `#FDF0EF`
- Steady: `#C49A2A` on `#FDF8EC`
- Easy: `#5A9E6F` on `#EFF8F2`

No pure black (#000000) anywhere. No hardcoded hex for brand colors... always use CSS variables.

### Typography
| Role | Font | Weight | Usage |
|---|---|---|---|
| Display / Logo | Pacifico | 400 | "FlowState" header |
| Headings / Labels | Fredoka | 600 | Section titles, sheet headers, pick-one card |
| Body / UI | Quicksand | 400-600 | Everything else: text, buttons, inputs, labels |

All loaded via Google Fonts CDN.

### Component Specs
| Component | Specs |
|---|---|
| Button (primary) | height 48px, padding 0 16px, border-radius 12px, bg var(--cta), white text, Quicksand 600 |
| Button (secondary) | same dimensions, bg var(--card), border 1.5px solid var(--border), color var(--text-muted) |
| Input | height 48px, border 1.5px solid var(--border), border-radius 12px, focus ring: 0 0 0 3px rgba(94,91,174,0.15) |
| Card | bg var(--card), border 1px solid var(--border), border-radius 16px, shadow 0 2px 12px rgba(94,91,174,0.08) |
| Bottom sheet | border-radius 24px 24px 0 0, backdrop-filter blur(4px), overlay rgba(42,40,80,0.4) |
| Touch target | minimum 44×44px |

### Spacing
8px base unit: 4px (xs) / 8px (s) / 16px (m) / 24px (l) / 48px (xl)

### Animations
- `fadeUp`: cards and UI elements enter with opacity 0→1 and translateY 8px→0 over 0.3s
- `gentlePulse`: WSID floating button scales 1→1.04→1 over 3s, infinite
- `breathe`: "how about this" label in pick-one mode, opacity 0.7→1→0.7 over 3s

### Voice / Copy Rules
- Write like a warm late-night text... intimate and direct, never corporate
- No em dashes... use ellipses instead
- No "Submit", "Proceed", "Click Here", "Navigate", "Access", "Utilize"
- Errors: calm and warm ("hmm something went wrong, try again")
- Empty states: personal ("nothing here yet... check back soon", "inbox is clear... nice and tidy")
- Success: intimate ("all done ✓", "there you go")

---

## Seed Data

On first launch (when localStorage is empty), pre-populate with sample data so the app demonstrates its concepts immediately:

**Pile tasks:**
- "respond to DMs" (easy / important / 30m)
- "edit this week's video" (deep / first / 2hr+)
- "post tip bait content" (steady / important / 15m)

**Backlog tasks:**
- "review ManyVids analytics" (steady / whenever / 30m)
- "plan next filming batch" (deep / important / 1hr)
- "update video gallery CSV" (easy / whenever / 15m)

**Inbox task:**
- "restock ring light bulbs" (untagged)

**Inbox ideas:**
- "try a pick your adventure style tip game"
- "collab with another creator for cross-promo"

---

## What This Is Not

- Not a calendar replacement... no time-blocking or meeting scheduling
- Not a project manager... no subtask trees, no assignees, no dependencies
- Not a habit tracker... completion history is decorative, not the point
- Not deadline-driven... no due dates, no overdue states, no urgency flags
- Not automated... priority drift is manual, planning happens when you choose
- Not collaborative... this is a solo tool for one person's brain

---

## Technical Architecture (V1)

- **Single-file HTML** with embedded CSS and JS. No build tools, no bundler, no framework.
- **Vanilla JavaScript.** No React, no Vue. Modern browser APIs only.
- **localStorage** for all persistence. Key: `flowstate_v1`. JSON value.
- **Google Fonts** loaded via CDN (Pacifico, Fredoka, Quicksand). Only external dependency.
- **Mobile-first.** Designed for 375px, max content width 520px.

### Why single-file HTML?
This is a personal tool that runs on a NAS, opens from a file manager, or gets served by a Flask dashboard. Single-file means zero setup, zero dependencies, zero deployment complexity. Double-click and it works.

---

## Build Order

| Phase | What | Why |
|---|---|---|
| 1 | Data layer (localStorage read/write, seed data) | Foundation for everything |
| 2 | Today view + task cards + checkboxes + progress bar + quick-add | The screen you see every day |
| 3 | WSID filter + "just pick one" mode | Proves the core thesis |
| 4 | Inbox + smart entry + idea capture + triage sheet | Closes the capture loop |
| 5 | Pile (add with priority, schedule, complete, note capture, habit dots) | The most novel mechanic |
| 6 | Backlog (add with full tagging, schedule to today) | Completes the zone system |
| 7 | Done view (completion log, date grouping, energy filter) | The reward / journal layer |
| 8 | Template tray (pre-built templates, preview, load onto today) | Recurring workflow support |
| 9 | Edit sheet (tap card to edit any task) | Fix mistakes without deleting |
| 10 | Two-tap delete on all card types | Safe deletion without confirm() |
| 11 | Tab navigation + floating WSID button | Ties it all together |

Each phase should be verified working before moving to the next. This matches the development style of "build minimal → test → refine."

---

## Future Phases

These are not part of V1. They're documented here so the architecture doesn't accidentally block them.

### Phase 2: Calendar Views
- **Day view:** Execution mode with block expansion, checkboxes, block progress fill
- **Week view:** Planning mode. Desktop: drag template blocks onto days. Mobile: tap day → add block sheet
- **Month view:** Read-only overview with colored energy chips per day

### Phase 3: Custom Templates
- Create your own templates from any combination of tasks
- Save a current day's task list as a new template
- Edit and delete custom templates

### Phase 4: Blocks & Sticky Blocks
- Named groups of tasks with a shared energy level
- Sticky blocks recur on a rule (e.g. every weekday) and auto-populate
- Blocks can contain both regular tasks and Pile tasks

### Phase 5: Dashboard Integration
- FlowState served by the Business Dashboard (Flask app on NAS)
- Widget on Dashboard's Home tab showing today's snapshot
- Data potentially migrated from localStorage to the dashboard's JSON backend
- Widget reads FlowState data, links to full app via `/open-tool`

### Phase 6: Notion Sync
- Push completions to a Notion database as a searchable work journal
- Same pattern as Booking Manager and Custom Content Tracker

---

## Decisions Log

Decisions made during prototyping that should be preserved:

| Decision | Reasoning |
|---|---|
| No deadlines or due dates | The whole point. Deadlines create anxiety without adding value for recurring personal work. |
| Energy is required before WSID works | Forces you to be honest about your capacity before seeing options. |
| Null time passes all filters | A task you forgot to time-tag shouldn't vanish from suggestions. Inclusive by default. |
| Backlog tasks are one-and-done | Prevents zombie tasks reappearing. If it's recurring, it belongs in the Pile. |
| Two-tap delete instead of confirm() | confirm() is silently blocked in iframes and some mobile browsers. Inline pattern is more reliable and feels better. |
| Pile tasks default to "important" but allow all priorities | WSID sorts by priority, so pile tasks need a priority to rank properly. |
| Ideas are separate from tasks in the inbox | Keeps capture friction at zero. An idea doesn't need energy/priority/time. It just needs to not get lost. |
| "Just pick one" exists | ADHD executive dysfunction means even 4 options can feel paralyzing. One card, yes or skip, done. |
| Templates are hardcoded in V1 | Custom template creation adds complexity. Prove the concept with real workflows first. |
| Soft Periwinkle palette (not Raspberry Pink) | FlowState is a personal tool, not an admin dashboard. It needed its own calm, focused palette. |
| Single-file HTML, not React | For a tool that lives on a NAS and gets served by Flask, zero-dependency vanilla JS is the right call. React adds build complexity for no benefit. |

---

*FlowState Product Spec v1.0*
*Last updated: March 2026*
*Companion doc: flowstate-claude-code-prompt.md*
