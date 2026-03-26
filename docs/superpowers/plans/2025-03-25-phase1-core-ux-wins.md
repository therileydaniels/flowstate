# Phase 1: Core UX Wins — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add drag-to-reorder, active task indicator, undo toast, "not today" button, and time estimate totals to FlowState's Today view, plus sync CLAUDE.md colors.

**Architecture:** All features are implemented in the single `index.html` file (embedded CSS + JS). PocketBase schema is updated via API. Each task is independent and produces a working commit.

**Tech Stack:** Vanilla JS, HTML5 drag-and-drop with touch handlers, PocketBase JS SDK, CSS variables

---

### Task 1: Update CLAUDE.md Design System Colors

**Files:**
- Modify: `CLAUDE.md:20-46`

- [ ] **Step 1: Replace the design system section header and light mode variables**

Change `CLAUDE.md` lines 20-36 from the rose/pink values to match the actual purple/indigo theme in `index.html`:

```markdown
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
```

- [ ] **Step 2: Replace the dark mode variables**

Change lines 38-46 to:

```markdown
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
```

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: sync CLAUDE.md design system to actual purple/indigo theme"
```

---

### Task 2: Add sortOrder Field to PocketBase Schema via API

**Files:**
- Modify: `index.html` (loadAll placement mapping, ~line 1179-1185)

- [ ] **Step 1: Add sortOrder field to flowstate_placements via PocketBase API**

Run this command to authenticate and add the field:

```bash
# Authenticate and get admin token
TOKEN=$(curl -s -X POST http://192.168.5.204:8090/api/admins/auth-with-password \
  -H "Content-Type: application/json" \
  -d '{"identity":"admin@admin.com","password":"admin1234"}' | grep -o '"token":"[^"]*"' | cut -d'"' -f4)

# Get current collection schema
SCHEMA=$(curl -s http://192.168.5.204:8090/api/collections/flowstate_placements \
  -H "Authorization: $TOKEN")

# Extract current schema fields and append sortOrder
echo "$SCHEMA" | python3 -c "
import sys, json
col = json.load(sys.stdin)
schema = col.get('schema', col.get('fields', []))
# Check if sortOrder already exists
if not any(f.get('name') == 'sortOrder' for f in schema):
    schema.append({
        'name': 'sortOrder',
        'type': 'number',
        'required': False,
        'options': {'min': None, 'max': None, 'noDecimal': True}
    })
    # Update collection
    import urllib.request
    data = json.dumps({'schema': schema}).encode()
    req = urllib.request.Request(
        'http://192.168.5.204:8090/api/collections/flowstate_placements',
        data=data,
        headers={'Content-Type': 'application/json', 'Authorization': col['__token__']},
        method='PATCH'
    )
"
```

Actually, simpler approach — use a single curl PATCH. First get the collection to find existing schema, then append the field:

```bash
# Get admin token
TOKEN=$(curl -s -X POST http://192.168.5.204:8090/api/admins/auth-with-password \
  -H "Content-Type: application/json" \
  -d '{"identity":"admin@admin.com","password":"admin1234"}' | jq -r '.token')

# Get current schema fields
FIELDS=$(curl -s http://192.168.5.204:8090/api/collections/flowstate_placements \
  -H "Authorization: $TOKEN" | jq '.schema')

# Append sortOrder field and PATCH
UPDATED=$(echo "$FIELDS" | jq '. + [{"name":"sortOrder","type":"number","required":false,"options":{"min":null,"max":null,"noDecimal":true}}]')

curl -s -X PATCH http://192.168.5.204:8090/api/collections/flowstate_placements \
  -H "Content-Type: application/json" \
  -H "Authorization: $TOKEN" \
  -d "{\"schema\": $UPDATED}"
```

Verify: the response JSON should include `sortOrder` in the schema array.

- [ ] **Step 2: Update loadAll() to include sortOrder in placement mapping**

In `index.html`, find the placements mapping in `loadAll()` (~line 1179-1185):

```javascript
// FIND THIS:
    data.placements = placements.map(function(p) {
      return {
        id: p.id,
        taskId: p.taskId,
        date: p.date,
        source: p.source
      };
    });

// REPLACE WITH:
    data.placements = placements.map(function(p) {
      return {
        id: p.id,
        taskId: p.taskId,
        date: p.date,
        source: p.source,
        sortOrder: p.sortOrder || 0
      };
    });
```

- [ ] **Step 3: Update placement creation helpers to set sortOrder**

Find `todayQuickAdd()` (~line 2082), the placement create call:

```javascript
// FIND THIS:
    await pb.collection('flowstate_placements').create({ taskId: created.id, date: todayStr(), source: 'manual' });

// REPLACE WITH:
    var maxSort = getTodayPlacements().reduce(function(max, p) { return Math.max(max, p.sortOrder || 0); }, 0);
    await pb.collection('flowstate_placements').create({ taskId: created.id, date: todayStr(), source: 'manual', sortOrder: maxSort + 1 });
```

Do the same for `schedulePile()` (~line 2223):

```javascript
// FIND THIS:
  await pb.collection('flowstate_placements').create({ taskId: taskId, date: today, source: 'manual' });

// REPLACE WITH:
  var maxSort = getTodayPlacements().reduce(function(max, p) { return Math.max(max, p.sortOrder || 0); }, 0);
  await pb.collection('flowstate_placements').create({ taskId: taskId, date: today, source: 'manual', sortOrder: maxSort + 1 });
```

Do the same for `scheduleBacklog()` (~line 2313):

```javascript
// FIND THIS:
  await pb.collection('flowstate_placements').create({ taskId: taskId, date: today, source: 'manual' });

// REPLACE WITH:
  var maxSort = getTodayPlacements().reduce(function(max, p) { return Math.max(max, p.sortOrder || 0); }, 0);
  await pb.collection('flowstate_placements').create({ taskId: taskId, date: today, source: 'manual', sortOrder: maxSort + 1 });
```

Do the same for `triageTo()` when dest is 'today' (~line 1591):

```javascript
// FIND THIS:
      await pb.collection('flowstate_placements').create({ taskId: task.id, date: todayStr(), source: 'manual' });

// REPLACE WITH:
      var maxSort = getTodayPlacements().reduce(function(max, p) { return Math.max(max, p.sortOrder || 0); }, 0);
      await pb.collection('flowstate_placements').create({ taskId: task.id, date: todayStr(), source: 'manual', sortOrder: maxSort + 1 });
```

Do the same for `loadTemplate()` (~line 1734):

```javascript
// FIND THIS:
    await pb.collection('flowstate_placements').create({ taskId: created.id, date: today, source: 'template' });

// REPLACE WITH:
    var maxSort = getTodayPlacements().reduce(function(max, p) { return Math.max(max, p.sortOrder || 0); }, 0) + i + 1;
    await pb.collection('flowstate_placements').create({ taskId: created.id, date: today, source: 'template', sortOrder: maxSort });
```

Note: in `loadTemplate()`, the loop uses `for (const t of tpl.tasks)`. Change to index-based to get `i`:

```javascript
// FIND THIS:
  for (const t of tpl.tasks) {
    const created = await pb.collection('flowstate_tasks').create({
      name: t.name, zone: 'backlog',
      energy: tpl.energy, priority: 'important', time: t.time,
      status: '', kind: 'task'
    });
    await pb.collection('flowstate_placements').create({ taskId: created.id, date: today, source: 'template' });
  }

// REPLACE WITH:
  var baseSort = getTodayPlacements().reduce(function(max, p) { return Math.max(max, p.sortOrder || 0); }, 0);
  for (var i = 0; i < tpl.tasks.length; i++) {
    var t = tpl.tasks[i];
    const created = await pb.collection('flowstate_tasks').create({
      name: t.name, zone: 'backlog',
      energy: tpl.energy, priority: 'important', time: t.time,
      status: '', kind: 'task'
    });
    await pb.collection('flowstate_placements').create({ taskId: created.id, date: today, source: 'template', sortOrder: baseSort + i + 1 });
  }
```

Also update `wsidLetsGo()` (~line 2509):

```javascript
// FIND THIS:
    await pb.collection('flowstate_placements').create({ taskId: taskId, date: today, source: 'manual' });

// REPLACE WITH:
    var maxSort = getTodayPlacements().reduce(function(max, p) { return Math.max(max, p.sortOrder || 0); }, 0);
    await pb.collection('flowstate_placements').create({ taskId: taskId, date: today, source: 'manual', sortOrder: maxSort + 1 });
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add sortOrder to placements for drag-to-reorder support"
```

---

### Task 3: Implement Drag-to-Reorder on Today View

**Files:**
- Modify: `index.html` (CSS ~line 285-300, renderToday ~line 2001-2066, new drag functions)

- [ ] **Step 1: Add drag-and-drop CSS**

Add after the `.task-card:active` rule (~line 300):

```css
    .drag-handle {
      width: 28px;
      min-width: 28px;
      height: 28px;
      display: flex;
      align-items: center;
      justify-content: center;
      color: var(--accent);
      font-size: 18px;
      cursor: grab;
      touch-action: none;
      user-select: none;
      -webkit-user-select: none;
    }
    .drag-handle:active { cursor: grabbing; }
    .task-card.dragging {
      opacity: 0.4;
      transform: scale(0.96);
    }
    .task-card.drag-over {
      border-top: 2px solid var(--cta);
      margin-top: -2px;
    }
```

- [ ] **Step 2: Add drag handle to Today active cards in renderToday()**

In `renderToday()`, find the active card HTML generation (~line 2031-2034). Replace:

```javascript
// FIND THIS:
    html += '<div class="task-card" data-task-id="' + task.id + '" data-zone="' + task.zone + '" onclick="cardTap(\'' + task.id + '\',\'' + task.zone + '\',event)">' +
      '<div class="card-checkbox energy-' + (task.energy || '') + '" role="checkbox" aria-checked="false" aria-label="complete ' + esc(task.name) + '" onclick="completeTask(\'' + task.id + '\',\'' + p.id + '\');event.stopPropagation()" tabindex="0" onkeydown="if(event.key===\'Enter\'||event.key===\' \'){completeTask(\'' + task.id + '\',\'' + p.id + '\');event.preventDefault()}"></div>' +
      '<div class="card-body"><div class="card-name">' + esc(task.name) + '</div><div class="card-meta">' + energyChip(task.energy) + timeChip(task.time) + (task.zone === 'pile' ? '<span class="chip chip-time">pile</span>' : '') + '</div></div>' +
      '<div class="card-actions">' + deleteBtn(task.id) + '</div></div>';

// REPLACE WITH:
    html += '<div class="task-card" data-task-id="' + task.id + '" data-placement-id="' + p.id + '" data-zone="' + task.zone + '" draggable="true" ondragstart="onDragStart(event)" ondragend="onDragEnd(event)" ondragover="onDragOver(event)" ondrop="onDrop(event)" onclick="cardTap(\'' + task.id + '\',\'' + task.zone + '\',event)">' +
      '<div class="drag-handle" ontouchstart="onTouchDragStart(event)" ontouchmove="onTouchDragMove(event)" ontouchend="onTouchDragEnd(event)">\u2261</div>' +
      '<div class="card-checkbox energy-' + (task.energy || '') + '" role="checkbox" aria-checked="false" aria-label="complete ' + esc(task.name) + '" onclick="completeTask(\'' + task.id + '\',\'' + p.id + '\');event.stopPropagation()" tabindex="0" onkeydown="if(event.key===\'Enter\'||event.key===\' \'){completeTask(\'' + task.id + '\',\'' + p.id + '\');event.preventDefault()}"></div>' +
      '<div class="card-body"><div class="card-name">' + esc(task.name) + '</div><div class="card-meta">' + energyChip(task.energy) + timeChip(task.time) + (task.zone === 'pile' ? '<span class="chip chip-time">pile</span>' : '') + '</div></div>' +
      '<div class="card-actions">' + deleteBtn(task.id) + '</div></div>';
```

- [ ] **Step 3: Change Today active card sort from priority to sortOrder**

In `renderToday()`, find the sort logic (~line 2023-2027). Replace:

```javascript
// FIND THIS:
  const sortedActive = active.map(function(p) {
    return { p: p, task: data.tasks.find(function(t) { return t.id === p.taskId; }) };
  }).filter(function(x) { return x.task; }).sort(function(a, b) {
    return (priorityOrder[a.task.priority] != null ? priorityOrder[a.task.priority] : 3) - (priorityOrder[b.task.priority] != null ? priorityOrder[b.task.priority] : 3);
  });

// REPLACE WITH:
  const sortedActive = active.map(function(p) {
    return { p: p, task: data.tasks.find(function(t) { return t.id === p.taskId; }) };
  }).filter(function(x) { return x.task; }).sort(function(a, b) {
    var sa = a.p.sortOrder || 0, sb = b.p.sortOrder || 0;
    if (sa !== sb) return sa - sb;
    return a.p.id.localeCompare(b.p.id);
  });
```

- [ ] **Step 4: Add HTML5 drag-and-drop handler functions**

Add after the `clearDeleteState()` function (~line 1989), before the TODAY VIEW section:

```javascript
// =======================================
// DRAG-TO-REORDER
// =======================================
let dragState = { draggedId: null };

function onDragStart(e) {
  var card = e.target.closest('.task-card');
  if (!card) return;
  dragState.draggedId = card.dataset.placementId;
  card.classList.add('dragging');
  e.dataTransfer.effectAllowed = 'move';
  e.dataTransfer.setData('text/plain', card.dataset.placementId);
}

function onDragEnd(e) {
  var card = e.target.closest('.task-card');
  if (card) card.classList.remove('dragging');
  document.querySelectorAll('.drag-over').forEach(function(el) { el.classList.remove('drag-over'); });
  dragState.draggedId = null;
}

function onDragOver(e) {
  e.preventDefault();
  e.dataTransfer.dropEffect = 'move';
  var card = e.target.closest('.task-card');
  if (!card || card.dataset.placementId === dragState.draggedId) return;
  document.querySelectorAll('.drag-over').forEach(function(el) { el.classList.remove('drag-over'); });
  card.classList.add('drag-over');
}

async function onDrop(e) {
  e.preventDefault();
  document.querySelectorAll('.drag-over').forEach(function(el) { el.classList.remove('drag-over'); });
  var targetCard = e.target.closest('.task-card');
  if (!targetCard) return;
  var fromId = dragState.draggedId;
  var toId = targetCard.dataset.placementId;
  if (!fromId || !toId || fromId === toId) return;
  await reorderPlacements(fromId, toId);
}

// Touch drag support for mobile
let touchDrag = { el: null, clone: null, startY: 0, placementId: null };

function onTouchDragStart(e) {
  var card = e.target.closest('.task-card');
  if (!card) return;
  e.preventDefault();
  touchDrag.placementId = card.dataset.placementId;
  touchDrag.el = card;
  touchDrag.startY = e.touches[0].clientY;
  card.classList.add('dragging');
}

function onTouchDragMove(e) {
  if (!touchDrag.el) return;
  e.preventDefault();
  var touch = e.touches[0];
  var target = document.elementFromPoint(touch.clientX, touch.clientY);
  if (!target) return;
  var card = target.closest('.task-card');
  document.querySelectorAll('.drag-over').forEach(function(el) { el.classList.remove('drag-over'); });
  if (card && card.dataset.placementId !== touchDrag.placementId) {
    card.classList.add('drag-over');
  }
}

async function onTouchDragEnd(e) {
  if (!touchDrag.el) return;
  touchDrag.el.classList.remove('dragging');
  var overEl = document.querySelector('.drag-over');
  document.querySelectorAll('.drag-over').forEach(function(el) { el.classList.remove('drag-over'); });
  if (overEl && overEl.dataset.placementId !== touchDrag.placementId) {
    await reorderPlacements(touchDrag.placementId, overEl.dataset.placementId);
  }
  touchDrag = { el: null, clone: null, startY: 0, placementId: null };
}

async function reorderPlacements(fromId, toId) {
  var today = todayStr();
  var placements = getTodayPlacements().filter(function(p) {
    return !isPlacementComplete(p.id);
  }).sort(function(a, b) {
    var sa = a.sortOrder || 0, sb = b.sortOrder || 0;
    if (sa !== sb) return sa - sb;
    return a.id.localeCompare(b.id);
  });

  var fromIdx = placements.findIndex(function(p) { return p.id === fromId; });
  var toIdx = placements.findIndex(function(p) { return p.id === toId; });
  if (fromIdx === -1 || toIdx === -1) return;

  var moved = placements.splice(fromIdx, 1)[0];
  placements.splice(toIdx, 0, moved);

  for (var i = 0; i < placements.length; i++) {
    if (placements[i].sortOrder !== i) {
      await pb.collection('flowstate_placements').update(placements[i].id, { sortOrder: i });
    }
  }
  await loadAll();
  render();
}
```

- [ ] **Step 5: Test drag-to-reorder**

Open the app in browser. On the Today view:
1. Verify grip handle (`≡`) appears on the left of each active card
2. Drag a card to a new position — cards should reorder
3. Refresh the page — order should persist
4. On mobile/touch: drag via the grip handle should work

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add drag-to-reorder on Today view with touch support"
```

---

### Task 4: Active Task Indicator with Timer

**Files:**
- Modify: `index.html` (CSS, state vars, renderToday, new functions)

- [ ] **Step 1: Add active task CSS**

Add after the `.drag-handle:active` CSS rule:

```css
    .task-card.active-task {
      border-left: 3px solid var(--cta);
    }
    .task-card.active-task.energy-deep-active { border-left-color: #C75450; }
    .task-card.active-task.energy-steady-active { border-left-color: #C49A2A; }
    .task-card.active-task.energy-easy-active { border-left-color: #5A9E6F; }
    .active-timer {
      font-family: 'Fredoka', sans-serif;
      font-size: 13px;
      font-weight: 600;
      color: var(--cta);
    }
    .active-timer.energy-deep { color: #C75450; }
    .active-timer.energy-steady { color: #C49A2A; }
    .active-timer.energy-easy { color: #5A9E6F; }
    .card-action.start-btn {
      color: var(--success);
      font-size: 16px;
    }
    .card-action.stop-btn {
      color: #C75450;
      font-size: 14px;
    }
```

- [ ] **Step 2: Add active task state variables**

In the STATE section (~line 1500), add after the `let doneFilter = null;` line:

```javascript
let activeTaskId = null;
let activeStartTime = null;
let activeTimerInterval = null;
```

- [ ] **Step 3: Add start/stop/timer functions**

Add after the active task state variables:

```javascript
function startTask(taskId, e) {
  if (e) e.stopPropagation();
  if (activeTaskId === taskId) { stopTask(e); return; }
  stopTask();
  activeTaskId = taskId;
  activeStartTime = Date.now();
  activeTimerInterval = setInterval(function() { updateActiveTimer(); }, 1000);
  render();
}

function stopTask(e) {
  if (e) e.stopPropagation();
  activeTaskId = null;
  activeStartTime = null;
  if (activeTimerInterval) { clearInterval(activeTimerInterval); activeTimerInterval = null; }
  render();
}

function updateActiveTimer() {
  var el = document.getElementById('active-timer-display');
  if (!el || !activeStartTime) return;
  var elapsed = Math.floor((Date.now() - activeStartTime) / 1000);
  var m = Math.floor(elapsed / 60);
  var s = elapsed % 60;
  el.textContent = String(m).padStart(2, '0') + ':' + String(s).padStart(2, '0');
}
```

- [ ] **Step 4: Update renderToday() to show start/stop buttons and timer**

In the active card HTML generation in `renderToday()`, update the card actions area. Replace the card HTML (the one we updated in Task 3) with:

```javascript
    var isActive = activeTaskId === task.id;
    var activeClass = isActive ? ' active-task energy-' + (task.energy || '') + '-active' : '';
    html += '<div class="task-card' + activeClass + '" data-task-id="' + task.id + '" data-placement-id="' + p.id + '" data-zone="' + task.zone + '" draggable="true" ondragstart="onDragStart(event)" ondragend="onDragEnd(event)" ondragover="onDragOver(event)" ondrop="onDrop(event)" onclick="cardTap(\'' + task.id + '\',\'' + task.zone + '\',event)">' +
      '<div class="drag-handle" ontouchstart="onTouchDragStart(event)" ontouchmove="onTouchDragMove(event)" ontouchend="onTouchDragEnd(event)">\u2261</div>' +
      '<div class="card-checkbox energy-' + (task.energy || '') + '" role="checkbox" aria-checked="false" aria-label="complete ' + esc(task.name) + '" onclick="completeTask(\'' + task.id + '\',\'' + p.id + '\');event.stopPropagation()" tabindex="0" onkeydown="if(event.key===\'Enter\'||event.key===\' \'){completeTask(\'' + task.id + '\',\'' + p.id + '\');event.preventDefault()}"></div>' +
      '<div class="card-body"><div class="card-name">' + esc(task.name) + '</div><div class="card-meta">' + energyChip(task.energy) + timeChip(task.time) + (task.zone === 'pile' ? '<span class="chip chip-time">pile</span>' : '') +
      (isActive ? ' <span class="active-timer energy-' + (task.energy || '') + '" id="active-timer-display">00:00</span>' : '') +
      '</div></div>' +
      '<div class="card-actions">' +
      (isActive
        ? '<button class="card-action stop-btn" onclick="stopTask(event)" aria-label="stop task">\u25A0</button>'
        : '<button class="card-action start-btn" onclick="startTask(\'' + task.id + '\',event)" aria-label="start task">\u25B6</button>'
      ) +
      deleteBtn(task.id) + '</div></div>';
```

- [ ] **Step 5: Clear active task on completion**

In `completeTask()` (~line 1904), add at the top of the function, before the pile check:

```javascript
// FIND THIS:
  const task = data.tasks.find(function(t) { return t.id === taskId; });
  if (!task) return;

// REPLACE WITH:
  const task = data.tasks.find(function(t) { return t.id === taskId; });
  if (!task) return;
  if (activeTaskId === taskId) { stopTask(); }
```

- [ ] **Step 6: Test active task indicator**

1. Open Today view with tasks
2. Tap play button — card should get colored left border and show 00:00 timer counting up
3. Tap play on another card — first card stops, second becomes active
4. Tap stop (square) — timer stops, highlight removed
5. Complete an active task — timer clears

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: add active task indicator with elapsed timer on Today view"
```

---

### Task 5: Undo Toast on Completion

**Files:**
- Modify: `index.html` (CSS, HTML, completeTask/finishCompletion functions)

- [ ] **Step 1: Add toast CSS**

Add after the animation keyframes (~line 1091, before `</style>`):

```css
    /* === UNDO TOAST === */
    .undo-toast {
      position: fixed;
      bottom: 72px;
      left: 50%;
      transform: translateX(-50%);
      max-width: 320px;
      padding: 12px 20px;
      background: var(--card);
      border: 1px solid var(--border);
      border-radius: 12px;
      box-shadow: 0 4px 16px var(--shadow);
      font-family: 'Quicksand', sans-serif;
      font-size: 14px;
      font-weight: 500;
      color: var(--text);
      z-index: 200;
      display: flex;
      align-items: center;
      gap: 8px;
      animation: fadeUp 0.2s ease-out;
      transition: opacity 0.3s;
    }
    .undo-toast.fading { opacity: 0; }
    .undo-toast-link {
      color: var(--cta);
      font-weight: 600;
      cursor: pointer;
      white-space: nowrap;
    }
```

- [ ] **Step 2: Add toast HTML container**

In the HTML body, add after the sheet div (~line 1120, after the `<div class="sheet"...></div>` line):

```html
      <div id="undo-toast" class="undo-toast" style="display:none"></div>
```

- [ ] **Step 3: Add toast state and functions**

Add after the active task functions:

```javascript
// =======================================
// UNDO TOAST
// =======================================
let undoToastTimeout = null;
let undoToastFadeTimeout = null;

function showUndoToast(placementId) {
  clearUndoToast();
  var toast = document.getElementById('undo-toast');
  if (!toast) return;
  toast.innerHTML = 'done! <span class="undo-toast-link" onclick="undoFromToast(\'' + placementId + '\')">undo</span>';
  toast.style.display = 'flex';
  toast.classList.remove('fading');
  undoToastFadeTimeout = setTimeout(function() {
    toast.classList.add('fading');
  }, 4700);
  undoToastTimeout = setTimeout(function() {
    toast.style.display = 'none';
    toast.classList.remove('fading');
  }, 5000);
}

function clearUndoToast() {
  if (undoToastTimeout) clearTimeout(undoToastTimeout);
  if (undoToastFadeTimeout) clearTimeout(undoToastFadeTimeout);
  var toast = document.getElementById('undo-toast');
  if (toast) { toast.style.display = 'none'; toast.classList.remove('fading'); }
}

async function undoFromToast(placementId) {
  clearUndoToast();
  await uncompleteTask(placementId);
}
```

- [ ] **Step 4: Trigger toast on task completion**

In `completeTask()`, after the completion is created for non-pile tasks (~line 1913-1921), add toast trigger. Replace:

```javascript
// FIND THIS (the non-pile branch):
  await pb.collection('flowstate_completions').create({
    taskId: taskId, placementId: placementId || '',
    completedAt: localISOString(), note: ''
  });
  if (task.zone === 'backlog') {
    await pb.collection('flowstate_tasks').update(task.id, { zone: 'done' });
  }
  await loadAll();
  render();

// REPLACE WITH:
  await pb.collection('flowstate_completions').create({
    taskId: taskId, placementId: placementId || '',
    completedAt: localISOString(), note: ''
  });
  if (task.zone === 'backlog') {
    await pb.collection('flowstate_tasks').update(task.id, { zone: 'done' });
  }
  await loadAll();
  render();
  if (placementId) showUndoToast(placementId);
```

Also in `finishCompletion()` (for pile tasks after note capture), add toast. Find (~line 1935):

```javascript
// FIND THIS:
  await loadAll();

// REPLACE WITH (in finishCompletion):
  await loadAll();
  if (placementId) showUndoToast(placementId);
```

Note: `finishCompletion` is called from `saveNote()` and `skipNote()`, both of which call `render()` after. The toast will show after the sheet closes.

- [ ] **Step 5: Test toast**

1. Complete a task on Today — toast "done! undo" appears above tab bar
2. Wait 5 seconds — toast fades and disappears
3. Complete a task and tap "undo" — task returns to active
4. Complete two tasks quickly — second toast replaces first

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add undo toast on task completion with 5-second auto-dismiss"
```

---

### Task 6: "Not Today" Button on Today Cards

**Files:**
- Modify: `index.html` (renderToday card actions, new function)

- [ ] **Step 1: Add notToday CSS**

Add after the `.card-action.evolve-btn` CSS:

```css
    .card-action.not-today-btn {
      color: var(--text-muted);
      font-size: 11px;
      width: auto;
      padding: 0 8px;
    }
    .card-action.not-today-btn:active {
      color: var(--cta);
    }
```

- [ ] **Step 2: Add "not today" button to Today active cards**

In the card actions HTML from Task 4's renderToday update, add the not-today button. The card-actions section becomes:

```javascript
      '<div class="card-actions">' +
      (isActive
        ? '<button class="card-action stop-btn" onclick="stopTask(event)" aria-label="stop task">\u25A0</button>'
        : '<button class="card-action start-btn" onclick="startTask(\'' + task.id + '\',event)" aria-label="start task">\u25B6</button>'
      ) +
      '<button class="card-action not-today-btn" onclick="notToday(\'' + task.id + '\',\'' + p.id + '\');event.stopPropagation()">not today</button>' +
      deleteBtn(task.id) + '</div></div>';
```

- [ ] **Step 3: Add notToday() function**

Add after the undo toast functions:

```javascript
// =======================================
// NOT TODAY
// =======================================
async function notToday(taskId, placementId, e) {
  if (e) e.stopPropagation();
  var task = data.tasks.find(function(t) { return t.id === taskId; });
  if (!task) return;
  if (activeTaskId === taskId) stopTask();
  await pb.collection('flowstate_placements').delete(placementId);
  if (task.zone === 'pile') {
    await pb.collection('flowstate_tasks').update(task.id, { status: 'available' });
  }
  await loadAll();
  render();
}
```

- [ ] **Step 4: Test "not today"**

1. Schedule a backlog task to Today
2. Tap "not today" — card disappears from Today, still in Backlog
3. Schedule a pile task to Today
4. Tap "not today" — card disappears from Today, pile task shows as available in Pile view

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add 'not today' button to remove tasks from Today without deleting"
```

---

### Task 7: Time Estimate Total on Today View

**Files:**
- Modify: `index.html` (renderToday function)

- [ ] **Step 1: Add time estimate formatting helper**

Add after the `priorityChip()` function (~line 1488):

```javascript
function formatTimeEstimate(minutes) {
  if (minutes <= 0) return '';
  if (minutes < 60) return '~' + minutes + 'm left';
  var hrs = Math.floor(minutes / 60);
  var mins = minutes % 60;
  if (mins === 0) return '~' + hrs + 'hr' + (hrs > 1 ? 's' : '') + ' left';
  return '~' + hrs + 'hr ' + mins + 'm left';
}
```

- [ ] **Step 2: Add time total display to renderToday()**

In `renderToday()`, after the progress bar HTML (~line 2019, after the closing `}` of the `if (total > 0)` block), add:

```javascript
  // Time estimate total
  var timeMap = { '15m': 15, '30m': 30, '1hr': 60, '2hr+': 120 };
  var totalMinutes = 0;
  var hasEstimates = false;
  sortedActive.forEach(function(item) {
    if (item.task.time && timeMap[item.task.time]) {
      totalMinutes += timeMap[item.task.time];
      hasEstimates = true;
    }
  });
  if (hasEstimates && sortedActive.length > 0) {
    html += '<div class="progress-label" style="margin-bottom:16px;padding-left:4px">' + formatTimeEstimate(totalMinutes) + '</div>';
  }
```

Note: this must be placed AFTER `sortedActive` is computed (which happens around line 2023-2027 in the original, but we changed the sort in Task 3). Make sure this block is after the `sortedActive` definition and before the `sortedActive.forEach` that renders cards.

- [ ] **Step 3: Test time total**

1. Add tasks with different time estimates to Today
2. Verify "~Xhr Ym left" appears below progress bar
3. Complete tasks — total should decrease
4. Complete all — time label disappears

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: show estimated time remaining on Today view"
```

---

### Task 8: Final Verification

- [ ] **Step 1: Full smoke test**

Open the app and verify all 6 features work together:
1. CLAUDE.md has correct purple/indigo colors
2. Today cards have drag handles and can be reordered (persists on refresh)
3. Start/stop button works with timer on active task
4. Completing a task shows undo toast for 5 seconds
5. "Not today" button removes from Today without deleting
6. Time estimate total shows below progress bar

- [ ] **Step 2: Deploy**

```bash
cp "index.html" "//192.168.5.204/docker/pocketbase/pb_public/flowstate/index.html"
```
