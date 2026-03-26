# FlowState

## Purpose
Personal energy-aware task management app. Uses priority x energy as its decision engine instead of deadlines or urgency. No overdue flags, no guilt, no nagging. Built for someone with ADHD who needs honest options matched to current capacity.

## Stack
- Single-file HTML (embedded CSS + JS, no build tools)
- PocketBase backend on Synology NAS (192.168.5.204:8090)
- PocketBase JS SDK via CDN
- Google Fonts loaded via CDN (Pacifico, Fredoka, Quicksand)
- No authentication (local network only)
- localStorage used ONLY for theme preference (`flowstate_theme`)

## Deployment
- File: `index.html` in project root
- Deploy path: `\\192.168.5.204\docker\pocketbase\pb_public\flowstate\index.html`
- Access URL: `http://192.168.5.204:8090/flowstate/`
- Admin UI: `http://192.168.5.204:8090/_/`

## Design System: Purple/Indigo
CSS variables — no hardcoded hex values anywhere except energy colors (functional, not brand):

:root {
  --bg: #F0EEFC;
  --bg2: #E0DCF8;
  --cta: #5E5BAE;
  --cta-hover: #4E4A9A;
  --accent: #A8A6D8;
  --success: #8AAEAA;
  --text: #2A2850;
  --text-muted: #6A68A0;
  --border: #C8C4EC;
  --card: #FFFFFF;
  --shadow: rgba(94,91,174,0.08);
  --focus: rgba(94,91,174,0.15);
}

Dark mode (html.dark):
  --bg: #1a1832;
  --bg2: #2a2850;
  --cta: #7B78C8;
  --cta-hover: #9390D8;
  --accent: #A8A6D8;
  --success: #6A9E8A;
  --text: #FFFFFF;
  --text-muted: #A8A6D8;
  --border: #3a3868;
  --card: #2a2850;
  --shadow: rgba(94,91,174,0.2);
  --focus: rgba(94,91,174,0.3);
  CTA buttons get glow: box-shadow: 0 0 20px rgba(94,91,174,0.4)

Fonts:
- Display/logo: Pacifico 400
- Headings/section labels: Fredoka 600
- Body/UI/buttons/inputs: Quicksand 400-600

Component specs:
- Buttons: height 48px, padding 0 16px, border-radius 8px, bg var(--cta), white text, Quicksand 600
- Inputs: height 48px, border 1.5px solid var(--border), border-radius 8px, focus ring: 0 0 0 3px var(--focus)
- Cards: bg var(--card), border 1px solid var(--border), border-radius 16px, shadow 0 2px 12px var(--shadow)
- Bottom sheets: border-radius 24px 24px 0 0, backdrop blur 4px, overlay rgba(42,40,80,0.4)
- Touch targets: minimum 44x44px
- Spacing: 8px base unit (4/8/16/24/48)

Energy colors (these are functional, not brand — keep as hardcoded):
- Deep Work: #C75450 (red), bg #FDF0EF
- Steady: #C49A2A (amber), bg #FDF8EC
- Easy: #5A9E6F (green), bg #EFF8F2

## Voice / Copy Rules
- Write UI copy like a warm late-night text, never corporate
- No em dashes — use ellipses instead
- No "Submit", "Proceed", "Click Here", "Navigate", "Access", "Utilize"
- Errors: warm tone ("hmm something went wrong, try again")
- Empty states: personal ("nothing here yet... check back soon")
- Never use pure black (#000000) — use var(--text) instead

## Core Concepts

### Energy Levels (required to do a task)
- Deep Work (red): high focus, creative, cognitively demanding
- Steady (amber): routine, moderate attention, familiar work
- Easy (green): autopilot, admin, low-stakes

### Priority (how much it matters)
- First: blocking something or has been waiting too long
- Important: real work, needs to happen, no rush
- Whenever: nice to do, low stakes

### Time Estimates (buckets, not exact)
15m, 30m, 1hr, 2hr+

### Task Kinds
- task: normal actionable item
- idea: a seed, not actionable yet (lives in inbox until "evolved" into a task)

## The Four Zones

### Inbox
- Raw capture, zero friction brain dump
- Supports smart entry: "edit thumbnails easy 30 min" auto-parses energy + time from text
- Also supports ideas: type "idea: your thought here" to capture as an idea
- Ideas show as visually distinct cards (dashed border, idea icon, italic text)
- Ideas have an "evolve" button that converts to task and opens triage sheet
- Triage: assign energy, priority, time -> send to Backlog, Pile, or directly to Today

### Pile
- Permanent standing tasks that cycle: Available -> Scheduled -> Completed -> back to Available
- When scheduled, task disappears from Pile (only one instance at a time)
- When completed, a note capture moment appears ("task done... add a note?" with save/skip)
- Completions are logged with timestamp + note, then task returns to Available
- Shows 7-day habit dots (filled = done that day, empty = not done, no guilt)
- Pile input includes energy, priority, AND time pickers

### Backlog
- Fully tagged tasks (energy, priority, time) waiting to be picked up
- One-and-done: completing a backlog task moves it to zone "done" permanently
- Can be scheduled onto today

### Done (completion log)
- All completions grouped by date, newest first
- Energy filter (all / deep / steady / easy)
- Shows task name, energy chip, timestamp, and note if one was saved
- Date headers: "today", "yesterday", day of week, then "Mar 15" format

## Views

### Today
- Quick-add input at top (smart entry, defaults to Steady/Important/30m if not parsed)
- Lightning button next to quick-add opens the Template Tray
- Progress bar below input ("3 of 7", fill bar, turns green with "all done" when complete)
- Checkable task cards with energy-colored checkbox borders
- Pile tasks trigger note capture sheet on completion
- "done today" section below active tasks (faded, strikethrough)
- End-of-day summary appears after 6 PM

### What Should I Do? (WSID)
- Accessible via floating button (bottom-right, subtle pulse animation)
- One-tap energy filter + one-tap time filter
- Shows matching tasks from Today + Backlog + available Pile, sorted by priority
- Tasks with no time estimate always pass the time filter (treated as "fits anything")
- "Just pick one" mode: toggle button, shows single focused card with large text

### Template Tray (bottom sheet)
- Opened via lightning button on Today view
- Full CRUD: create, edit, delete templates
- Lists templates with icon, name, task count, energy chip
- Tap to preview, "load onto today" dumps tasks as backlog items with placements
- Templates stored in flowstate_templates collection

## Data Model (PocketBase Collections)

### flowstate_tasks
- id: string (PocketBase 15-char auto-generated)
- name: text
- zone: text ("inbox" | "backlog" | "pile" | "done")
- energy: text ("deep" | "steady" | "easy" | "")
- priority: text ("first" | "important" | "whenever" | "")
- time: text ("15m" | "30m" | "1hr" | "2hr+" | "")
- status: text ("available" | "scheduled" | "")
- kind: text ("task" | "idea")

### flowstate_placements
- id: string (auto-generated)
- taskId: relation (-> flowstate_tasks)
- date: text ("YYYY-MM-DD")
- source: text ("manual" | "template")

### flowstate_completions
- id: string (auto-generated)
- taskId: relation (-> flowstate_tasks)
- placementId: relation (-> flowstate_placements)
- completedAt: text (ISO datetime string)
- note: editor

### flowstate_templates
- id: string (auto-generated)
- name: text
- icon: text
- energy: text
- tasks: json (array of {name, time} objects)

## Architecture
- Local `data` cache object holds tasks, placements, completions, templates arrays
- `loadAll()` fetches all 4 collections from PocketBase into `data`
- All mutations are async: call PocketBase API, then `await loadAll()` to refresh cache
- `render()` is synchronous, reads from local `data` cache
- Seed data auto-populates on first launch if flowstate_tasks collection is empty

## UX Patterns
- Two-tap delete: first tap turns x into red "sure?" button, auto-resets after 2.5 seconds
- Tap task card content area to open edit sheet (change name, energy, priority, time)
- Bottom sheets for all modals (triage, edit, note capture, template tray)
- Smart entry parsing: pattern-matches energy labels and time buckets from natural text
- No confirm() dialogs (blocked in iframes) — use inline confirmation patterns
- Tab nav: compact labels with icons, hidden scrollbar, right-edge fade gradient for scroll hint
- Prevent double-scheduling (check if placement already exists before creating)
- Animations: fadeUp on card entry, gentle pulse on WSID button, breathing animation on pick-one label
- Dark mode toggle in header (sun/moon icon), persisted via localStorage
- Streak counter in header showing consecutive days with completions

## Seed Data (first launch only)
Pre-populate with sample tasks so the app isn't empty:
- Pile: "respond to DMs" (easy/important/30m), "edit this week's video" (deep/first/2hr+), "post tip bait content" (steady/important/15m)
- Backlog: "review ManyVids analytics" (steady/whenever/30m), "plan next filming batch" (deep/important/1hr), "update video gallery CSV" (easy/whenever/15m)
- Inbox task: "restock ring light bulbs"
- Inbox ideas: "try a pick your adventure style tip game", "collab with another creator for cross-promo"
- 6 seed templates: Tuesday Fulfillment, Filming Day, Admin Hour, Edit Session, Content Planning, Upload Day

## Known Constraints
- This is a single-file HTML app. Everything in one file. No separate CSS or JS files.
- Mobile-first. Design for 375px width, scale up gracefully to 520px max content width.
- PocketBase on local NAS. No internet dependency for data (only Google Fonts + PB SDK CDN).
- No build tools. Vanilla JS with modern browser APIs. No React, no bundler.
