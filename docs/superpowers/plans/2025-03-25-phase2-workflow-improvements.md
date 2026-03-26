# Phase 2: Workflow Improvements — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reduce friction in inbox triage, enable future template scheduling, visually separate pile tasks on Today, and allow notes on any completed task.

**Architecture:** All changes in the single `index.html` file (embedded CSS + JS). No PocketBase schema changes. New batch triage uses a full-screen panel pattern (same as WSID). Template scheduling adds day-picker buttons to existing preview sheet.

**Tech Stack:** Vanilla JS, PocketBase JS SDK, CSS variables

---

### Task 1: Quick-Schedule from Inbox

**Files:**
- Modify: `index.html` (renderInbox ~line 2421, new inboxQuickSchedule function)

- [ ] **Step 1: Add "today" button to inbox task cards**

In `renderInbox()`, find the tasks.forEach block (~line 2421-2425). The current card HTML is:

```javascript
  tasks.forEach(function(task) {
    html += '<div class="task-card" onclick="openTriageSheet(\'' + task.id + '\')">' +
      '<div class="card-body"><div class="card-name">' + esc(task.name) + '</div><div class="card-meta">' + energyChip(task.energy) + timeChip(task.time) + '</div></div>' +
      '<div class="card-actions"><button class="card-action evolve-btn" onclick="openTriageSheet(\'' + task.id + '\');event.stopPropagation()">triage</button>' + deleteBtn(task.id) + '</div></div>';
  });
```

Replace with (add "today" button before triage):

```javascript
  tasks.forEach(function(task) {
    html += '<div class="task-card" onclick="openTriageSheet(\'' + task.id + '\')">' +
      '<div class="card-body"><div class="card-name">' + esc(task.name) + '</div><div class="card-meta">' + energyChip(task.energy) + timeChip(task.time) + '</div></div>' +
      '<div class="card-actions"><button class="card-action schedule-btn" onclick="inboxQuickSchedule(\'' + task.id + '\');event.stopPropagation()">today</button><button class="card-action evolve-btn" onclick="openTriageSheet(\'' + task.id + '\');event.stopPropagation()">triage</button>' + deleteBtn(task.id) + '</div></div>';
  });
```

- [ ] **Step 2: Add inboxQuickSchedule function**

Add after the `evolveIdea()` function (~line 2464):

```javascript
async function inboxQuickSchedule(taskId) {
  var task = data.tasks.find(function(t) { return t.id === taskId; });
  if (!task) return;
  await pb.collection('flowstate_tasks').update(task.id, {
    zone: 'backlog',
    energy: task.energy || 'steady',
    priority: task.priority || 'important',
    time: task.time || '30m',
    kind: 'task'
  });
  var maxSort = getTodayPlacements().reduce(function(max, p) { return Math.max(max, p.sortOrder || 0); }, 0);
  await pb.collection('flowstate_placements').create({ taskId: task.id, date: todayStr(), source: 'manual', sortOrder: maxSort + 1 });
  await loadAll();
  render();
}
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add quick-schedule 'today' button on inbox task cards"
```

---

### Task 2: Batch Triage Panel

**Files:**
- Modify: `index.html` (CSS, HTML, renderInbox, new batch triage functions)

- [ ] **Step 1: Add batch triage CSS**

Add before the closing `</style>` tag (after the undo-toast CSS):

```css
    /* === BATCH TRIAGE === */
    #batch-triage-panel {
      position: fixed;
      inset: 0;
      z-index: 100;
      background: var(--bg);
      display: none;
      flex-direction: column;
    }
    #batch-triage-panel.open { display: flex; }
    .batch-header {
      display: flex;
      align-items: center;
      justify-content: space-between;
      padding: 16px;
      max-width: 520px;
      width: 100%;
      margin: 0 auto;
    }
    .batch-progress {
      font-family: 'Fredoka', sans-serif;
      font-weight: 600;
      font-size: 14px;
      color: var(--text-muted);
    }
    .batch-close {
      width: 44px;
      height: 44px;
      border: none;
      border-radius: 12px;
      background: var(--bg2);
      color: var(--text);
      font-size: 18px;
      cursor: pointer;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    .batch-content {
      flex: 1;
      overflow-y: auto;
      padding: 0 16px 16px;
      max-width: 520px;
      width: 100%;
      margin: 0 auto;
    }
    .batch-task-name {
      font-family: 'Fredoka', sans-serif;
      font-weight: 600;
      font-size: 20px;
      color: var(--text);
      text-align: center;
      margin-bottom: 24px;
      line-height: 1.3;
    }
    .batch-section-label {
      font-family: 'Fredoka', sans-serif;
      font-size: 12px;
      font-weight: 600;
      color: var(--text-muted);
      margin-bottom: 6px;
    }
    .batch-section { margin-bottom: 16px; }
    .batch-skip {
      display: block;
      margin: 16px auto 0;
      background: none;
      border: none;
      color: var(--text-muted);
      font-family: 'Quicksand', sans-serif;
      font-size: 13px;
      font-weight: 600;
      cursor: pointer;
      padding: 8px 16px;
    }
    .batch-skip:active { color: var(--text); }
    .btn-triage-all {
      width: 100%;
      height: 44px;
      border: 1.5px solid var(--border);
      border-radius: 12px;
      background: var(--card);
      font-family: 'Quicksand', sans-serif;
      font-size: 14px;
      font-weight: 600;
      color: var(--cta);
      cursor: pointer;
      margin-bottom: 16px;
    }
    .btn-triage-all:active { background: var(--bg2); }
```

- [ ] **Step 2: Add batch triage panel HTML**

In the HTML body, find the WSID panel div (~line 1204):
```html
      <div id="wsid-panel"></div>
```

Add after it:
```html
      <div id="batch-triage-panel"></div>
```

- [ ] **Step 3: Add "triage all" button to renderInbox()**

In `renderInbox()`, after the hint-text line (~line 2412) and after the tasks/ideas are split (~line 2414-2415), add the triage-all button:

Find:
```javascript
  const tasks = items.filter(function(i) { return i.kind === 'task'; });
  const ideas = items.filter(function(i) { return i.kind === 'idea'; });

  if (tasks.length === 0 && ideas.length === 0) {
```

Replace with:
```javascript
  const tasks = items.filter(function(i) { return i.kind === 'task'; });
  const ideas = items.filter(function(i) { return i.kind === 'idea'; });

  if (tasks.length >= 2) {
    html += '<button class="btn-triage-all" onclick="openBatchTriage()">triage all (' + tasks.length + ')</button>';
  }

  if (tasks.length === 0 && ideas.length === 0) {
```

- [ ] **Step 4: Add batch triage state and functions**

Add after the `inboxQuickSchedule()` function:

```javascript
// =======================================
// BATCH TRIAGE
// =======================================
let batchTriageState = { tasks: [], index: 0, energy: null, priority: null, time: null };

function openBatchTriage() {
  var inboxTasks = data.tasks.filter(function(t) { return t.zone === 'inbox' && t.kind === 'task'; });
  if (inboxTasks.length === 0) return;
  batchTriageState = { tasks: inboxTasks.map(function(t) { return t.id; }), index: 0, energy: null, priority: null, time: null };
  renderBatchTriage();
  document.getElementById('batch-triage-panel').classList.add('open');
}

function closeBatchTriage() {
  document.getElementById('batch-triage-panel').classList.remove('open');
  render();
}

function batchPick(type, val) {
  batchTriageState[type] = batchTriageState[type] === val ? null : val;
  renderBatchTriage();
}

function renderBatchTriage() {
  var panel = document.getElementById('batch-triage-panel');
  var idx = batchTriageState.index;
  var total = batchTriageState.tasks.length;

  if (idx >= total) {
    closeBatchTriage();
    return;
  }

  var taskId = batchTriageState.tasks[idx];
  var task = data.tasks.find(function(t) { return t.id === taskId; });

  if (!task || task.zone !== 'inbox') {
    batchTriageState.index++;
    renderBatchTriage();
    return;
  }

  var html = '<div class="batch-header"><span class="batch-progress">' + (idx + 1) + ' of ' + total + '</span><button class="batch-close" onclick="closeBatchTriage()">\u00d7</button></div>';
  html += '<div class="batch-content">';
  html += '<div class="batch-task-name">' + esc(task.name) + '</div>';

  var energyOpts = [
    { val: 'deep', label: '\uD83D\uDD34 deep work' },
    { val: 'steady', label: '\uD83D\uDFE1 steady' },
    { val: 'easy', label: '\uD83D\uDFE2 easy' }
  ];
  var priorityOpts = [
    { val: 'first', label: 'first' },
    { val: 'important', label: 'important' },
    { val: 'whenever', label: 'whenever' }
  ];
  var timeOpts = [
    { val: '15m', label: '15m' },
    { val: '30m', label: '30m' },
    { val: '1hr', label: '1hr' },
    { val: '2hr+', label: '2hr+' }
  ];

  html += '<div class="batch-section"><div class="batch-section-label">energy</div><div class="picker-row">';
  energyOpts.forEach(function(o) {
    html += '<button class="picker-chip energy-' + o.val + ' ' + (batchTriageState.energy === o.val ? 'selected' : '') + '" onclick="batchPick(\'energy\',\'' + o.val + '\')">' + o.label + '</button>';
  });
  html += '</div></div>';

  html += '<div class="batch-section"><div class="batch-section-label">priority</div><div class="picker-row">';
  priorityOpts.forEach(function(o) {
    html += '<button class="picker-chip ' + (batchTriageState.priority === o.val ? 'selected' : '') + '" onclick="batchPick(\'priority\',\'' + o.val + '\')">' + o.label + '</button>';
  });
  html += '</div></div>';

  html += '<div class="batch-section"><div class="batch-section-label">time</div><div class="picker-row">';
  timeOpts.forEach(function(o) {
    html += '<button class="picker-chip ' + (batchTriageState.time === o.val ? 'selected' : '') + '" onclick="batchPick(\'time\',\'' + o.val + '\')">' + o.label + '</button>';
  });
  html += '</div></div>';

  html += '<div class="sheet-dest-row"><button class="sheet-dest-btn" onclick="batchTriageTo(\'backlog\')">backlog</button><button class="sheet-dest-btn" onclick="batchTriageTo(\'pile\')">pile</button><button class="sheet-dest-btn" onclick="batchTriageTo(\'today\')">today</button></div>';
  html += '<button class="batch-skip" onclick="batchSkip()">skip</button>';
  html += '</div>';

  panel.innerHTML = html;
}

async function batchTriageTo(dest) {
  var taskId = batchTriageState.tasks[batchTriageState.index];
  var task = data.tasks.find(function(t) { return t.id === taskId; });
  if (!task) { batchAdvance(); return; }

  var updates = {
    energy: batchTriageState.energy || '',
    priority: batchTriageState.priority || '',
    time: batchTriageState.time || '',
    kind: 'task'
  };

  if (dest === 'today') {
    updates.zone = 'backlog';
    await pb.collection('flowstate_tasks').update(task.id, updates);
    if (!hasActivePlacement(task.id, todayStr())) {
      var maxSort = getTodayPlacements().reduce(function(max, p) { return Math.max(max, p.sortOrder || 0); }, 0);
      await pb.collection('flowstate_placements').create({ taskId: task.id, date: todayStr(), source: 'manual', sortOrder: maxSort + 1 });
    }
  } else if (dest === 'pile') {
    updates.zone = 'pile';
    updates.status = 'available';
    await pb.collection('flowstate_tasks').update(task.id, updates);
  } else {
    updates.zone = 'backlog';
    await pb.collection('flowstate_tasks').update(task.id, updates);
  }

  await loadAll();
  batchAdvance();
}

function batchSkip() {
  batchAdvance();
}

function batchAdvance() {
  batchTriageState.index++;
  batchTriageState.energy = null;
  batchTriageState.priority = null;
  batchTriageState.time = null;
  renderBatchTriage();
}
```

- [ ] **Step 5: Test batch triage**

1. Add 3+ tasks to inbox
2. "triage all (3)" button should appear
3. Tap it — full-screen panel shows first task
4. Pick energy/priority/time, tap "backlog" — advances to next, pickers reset
5. Tap "skip" — advances without triaging
6. After last item, panel closes

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add batch triage swipe-through panel for inbox tasks"
```

---

### Task 3: Pile Task Section on Today View

**Files:**
- Modify: `index.html` (renderToday ~line 2310-2349, reorderPlacements ~line 2243)

- [ ] **Step 1: Split Today active cards into regular and pile groups**

In `renderToday()`, find the `sortedActive` definition and the card rendering loop (~line 2310-2349). Replace the sortedActive definition, time estimate block, and card forEach with:

```javascript
  const sortedActive = active.map(function(p) {
    return { p: p, task: data.tasks.find(function(t) { return t.id === p.taskId; }) };
  }).filter(function(x) { return x.task; });

  var regularCards = sortedActive.filter(function(x) { return x.task.zone !== 'pile'; }).sort(function(a, b) {
    var sa = a.p.sortOrder || 0, sb = b.p.sortOrder || 0;
    if (sa !== sb) return sa - sb;
    return a.p.id.localeCompare(b.p.id);
  });
  var pileCards = sortedActive.filter(function(x) { return x.task.zone === 'pile'; }).sort(function(a, b) {
    var sa = a.p.sortOrder || 0, sb = b.p.sortOrder || 0;
    if (sa !== sb) return sa - sb;
    return a.p.id.localeCompare(b.p.id);
  });

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

  function renderTodayCard(item) {
    var p = item.p, task = item.task;
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
      '<button class="card-action not-today-btn" onclick="notToday(\'' + task.id + '\',\'' + p.id + '\',event)">not today</button>' +
      deleteBtn(task.id) + '</div></div>';
  }

  regularCards.forEach(renderTodayCard);

  if (pileCards.length > 0) {
    html += '<div class="section-label">recurring</div>';
    pileCards.forEach(renderTodayCard);
  }
```

- [ ] **Step 2: Update reorderPlacements to respect groups**

Find `reorderPlacements()` (~line 2243). Replace it with a version that only reorders within the same group (pile vs non-pile):

```javascript
async function reorderPlacements(fromId, toId) {
  var today = todayStr();
  var allActive = getTodayPlacements().filter(function(p) {
    return !isPlacementComplete(p.id);
  });

  var fromPlacement = allActive.find(function(p) { return p.id === fromId; });
  var toPlacement = allActive.find(function(p) { return p.id === toId; });
  if (!fromPlacement || !toPlacement) return;

  var fromTask = data.tasks.find(function(t) { return t.id === fromPlacement.taskId; });
  var toTask = data.tasks.find(function(t) { return t.id === toPlacement.taskId; });
  if (!fromTask || !toTask) return;

  // Only reorder within same group
  var fromIsPile = fromTask.zone === 'pile';
  var toIsPile = toTask.zone === 'pile';
  if (fromIsPile !== toIsPile) return;

  var groupPlacements = allActive.filter(function(p) {
    var t = data.tasks.find(function(tk) { return tk.id === p.taskId; });
    return t && (t.zone === 'pile') === fromIsPile;
  }).sort(function(a, b) {
    var sa = a.sortOrder || 0, sb = b.sortOrder || 0;
    if (sa !== sb) return sa - sb;
    return a.id.localeCompare(b.id);
  });

  var fromIdx = groupPlacements.findIndex(function(p) { return p.id === fromId; });
  var toIdx = groupPlacements.findIndex(function(p) { return p.id === toId; });
  if (fromIdx === -1 || toIdx === -1) return;

  var moved = groupPlacements.splice(fromIdx, 1)[0];
  groupPlacements.splice(toIdx, 0, moved);

  for (var i = 0; i < groupPlacements.length; i++) {
    if (groupPlacements[i].sortOrder !== i) {
      await pb.collection('flowstate_placements').update(groupPlacements[i].id, { sortOrder: i });
    }
  }
  await loadAll();
  render();
}
```

- [ ] **Step 3: Test pile section**

1. Schedule a pile task and a backlog task onto Today
2. Regular task should appear above "recurring" section label
3. Pile task should appear below the label
4. Drag reorder within each section works
5. Dragging across sections does nothing

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: split Today view into regular and recurring task sections"
```

---

### Task 4: Future Template Scheduling

**Files:**
- Modify: `index.html` (openTemplatePreview ~line 1872, loadTemplate ~line 1886)

- [ ] **Step 1: Add date helper for next 7 days**

Add after the `dateLabel()` function (~after line where dateLabel is defined):

```javascript
function getNext7Days() {
  var days = [];
  var now = new Date();
  for (var i = 0; i < 7; i++) {
    var d = new Date(now);
    d.setDate(d.getDate() + i);
    var ds = d.getFullYear() + '-' + String(d.getMonth()+1).padStart(2,'0') + '-' + String(d.getDate()).padStart(2,'0');
    var label, sub;
    if (i === 0) { label = 'today'; }
    else if (i === 1) { label = 'tomorrow'; }
    else { label = d.toLocaleDateString([], { weekday: 'long' }).toLowerCase(); }
    sub = d.toLocaleDateString([], { month: 'short', day: 'numeric' }).toLowerCase();
    days.push({ date: ds, label: label, sub: sub });
  }
  return days;
}
```

- [ ] **Step 2: Add day picker CSS**

Add after the existing template tray CSS (near `.btn-new-template`):

```css
    .day-picker-row {
      display: flex;
      flex-wrap: wrap;
      gap: 6px;
      margin-top: 16px;
    }
    .day-picker-btn {
      flex: 1;
      min-width: 70px;
      padding: 8px 4px;
      border: 1.5px solid var(--border);
      border-radius: 10px;
      background: var(--card);
      font-family: 'Quicksand', sans-serif;
      font-size: 13px;
      font-weight: 600;
      color: var(--text);
      cursor: pointer;
      text-align: center;
      line-height: 1.3;
    }
    .day-picker-btn:active {
      background: var(--cta);
      color: white;
      border-color: var(--cta);
    }
    .day-picker-sub {
      display: block;
      font-size: 11px;
      font-weight: 500;
      color: var(--text-muted);
    }
    .day-picker-btn:active .day-picker-sub { color: rgba(255,255,255,0.7); }
```

- [ ] **Step 3: Replace "load onto today" button with day picker row**

In `openTemplatePreview()` (~line 1882), find the sheet-actions HTML:

```javascript
  html += '<div class="sheet-actions"><button class="sheet-btn-secondary" onclick="openTemplateTray()">back</button><button class="sheet-btn-secondary" onclick="openTemplateEdit(\'' + id + '\')">edit</button><button class="sheet-btn-primary" onclick="loadTemplate(\'' + id + '\')">load onto today</button></div>';
```

Replace with:

```javascript
  html += '<div class="sheet-actions"><button class="sheet-btn-secondary" onclick="openTemplateTray()">back</button><button class="sheet-btn-secondary" onclick="openTemplateEdit(\'' + id + '\')">edit</button></div>';
  var days = getNext7Days();
  html += '<div class="day-picker-row">';
  days.forEach(function(d) {
    html += '<button class="day-picker-btn" onclick="loadTemplate(\'' + id + '\',\'' + d.date + '\')">' + d.label + '<span class="day-picker-sub">' + d.sub + '</span></button>';
  });
  html += '</div>';
```

- [ ] **Step 4: Update loadTemplate() to accept a date parameter**

Find `loadTemplate()` (~line 1886):

```javascript
async function loadTemplate(id) {
  const tpl = data.templates.find(function(t) { return t.id === id; });
  if (!tpl) return;
  const today = todayStr();
```

Replace with:

```javascript
async function loadTemplate(id, targetDate) {
  const tpl = data.templates.find(function(t) { return t.id === id; });
  if (!tpl) return;
  var loadDate = targetDate || todayStr();
  var isToday = loadDate === todayStr();
```

Then update all references to `today` within the function to use `loadDate`:

Find in loadTemplate:
```javascript
  var baseSort = getTodayPlacements().reduce(function(max, p) { return Math.max(max, p.sortOrder || 0); }, 0);
```
Replace with:
```javascript
  var datePlacements = data.placements.filter(function(p) { return p.date === loadDate; });
  var baseSort = datePlacements.reduce(function(max, p) { return Math.max(max, p.sortOrder || 0); }, 0);
```

Find in loadTemplate:
```javascript
    await pb.collection('flowstate_placements').create({ taskId: created.id, date: today, source: 'template', sortOrder: baseSort + i + 1 });
```
Replace with:
```javascript
    await pb.collection('flowstate_placements').create({ taskId: created.id, date: loadDate, source: 'template', sortOrder: baseSort + i + 1 });
```

Find in loadTemplate at the end:
```javascript
  await loadAll();
  closeSheet();
  switchTab('today');
```
Replace with:
```javascript
  await loadAll();
  closeSheet();
  if (isToday) {
    switchTab('today');
  } else {
    var dayLabel = getNext7Days().find(function(d) { return d.date === loadDate; });
    showUndoToast_msg('loaded for ' + (dayLabel ? dayLabel.label : loadDate));
    render();
  }
```

- [ ] **Step 5: Add showUndoToast_msg for non-undo messages**

Add after the existing `undoFromToast()` function:

```javascript
function showUndoToast_msg(message) {
  clearUndoToast();
  var toast = document.getElementById('undo-toast');
  if (!toast) return;
  toast.innerHTML = message;
  toast.style.display = 'flex';
  toast.classList.remove('fading');
  undoToastFadeTimeout = setTimeout(function() {
    toast.classList.add('fading');
  }, 2700);
  undoToastTimeout = setTimeout(function() {
    toast.style.display = 'none';
    toast.classList.remove('fading');
  }, 3000);
}
```

- [ ] **Step 6: Test future template scheduling**

1. Open template tray, pick a template, tap to preview
2. Should see 7 day buttons instead of single "load onto today"
3. Tap "today" — loads template onto today, switches to Today view
4. Tap "thursday" — loads template for that date, shows toast "loaded for thursday"
5. Verify placements were created with the correct date

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: add future date scheduling for templates with 7-day picker"
```

---

### Task 5: Notes on All Completions

**Files:**
- Modify: `index.html` (CSS, renderToday done section ~line 2355, renderDone ~line 2674, new openEditNoteSheet function)

- [ ] **Step 1: Add note link CSS**

Add after the done-entry CSS:

```css
    .note-link {
      font-size: 12px;
      font-weight: 600;
      color: var(--cta);
      cursor: pointer;
      background: none;
      border: none;
      padding: 0;
      font-family: 'Quicksand', sans-serif;
    }
    .note-link:active { opacity: 0.7; }
```

- [ ] **Step 2: Add openEditNoteSheet function**

Add after the existing `skipNote()` function:

```javascript
function openEditNoteSheet(completionId) {
  var comp = data.completions.find(function(c) { return c.id === completionId; });
  if (!comp) return;
  var task = data.tasks.find(function(t) { return t.id === comp.taskId; });
  var name = task ? task.name : 'task';
  sheetState = { mode: 'edit-note', completionId: completionId };
  var html = '<div class="sheet-title">edit note</div>';
  html += '<p style="text-align:center;color:var(--text-muted);margin-bottom:16px">' + esc(name) + '</p>';
  html += '<textarea class="sheet-textarea" id="note-textarea" placeholder="how\'d it go...">' + esc(comp.note || '') + '</textarea>';
  html += '<div class="sheet-actions"><button class="sheet-btn-secondary" onclick="closeSheet()">cancel</button><button class="sheet-btn-primary" onclick="saveEditNote()">save</button></div>';
  openSheet(html);
}

async function saveEditNote() {
  var textarea = document.getElementById('note-textarea');
  var note = textarea ? textarea.value.trim() : '';
  var completionId = sheetState.completionId;
  if (!completionId) return;
  await pb.collection('flowstate_completions').update(completionId, { note: note });
  await loadAll();
  closeSheet();
  render();
}
```

- [ ] **Step 3: Update "done today" section in renderToday()**

Find the done section in renderToday() (~line 2355-2364):

```javascript
  if (done.length > 0) {
    html += '<div class="section-label">done today</div>';
    done.forEach(function(p) {
      const task = data.tasks.find(function(t) { return t.id === p.taskId; });
      const name = task ? task.name : 'deleted task';
      const energy = task ? task.energy : null;
      html += '<div class="task-card done-card" onclick="uncompleteTask(\'' + p.id + '\')">' +
        '<div class="card-checkbox checked" role="checkbox" aria-checked="true" aria-label="undo completion">\u2713</div>' +
        '<div class="card-body"><div class="card-name">' + esc(name) + '</div><div class="card-meta">' + energyChip(energy) + '</div></div></div>';
    });
  }
```

Replace with:

```javascript
  if (done.length > 0) {
    html += '<div class="section-label">done today</div>';
    done.forEach(function(p) {
      const task = data.tasks.find(function(t) { return t.id === p.taskId; });
      const name = task ? task.name : 'deleted task';
      const energy = task ? task.energy : null;
      const comp = data.completions.find(function(c) { return c.placementId === p.id; });
      html += '<div class="task-card done-card">' +
        '<div class="card-checkbox checked" role="checkbox" aria-checked="true" aria-label="undo completion" onclick="uncompleteTask(\'' + p.id + '\')">\u2713</div>' +
        '<div class="card-body"><div class="card-name">' + esc(name) + '</div><div class="card-meta">' + energyChip(energy) + '</div>' +
        (comp && comp.note ? '<div class="done-entry-note">"' + esc(comp.note) + '" <button class="note-link" onclick="openEditNoteSheet(\'' + comp.id + '\');event.stopPropagation()">edit</button></div>' : (comp ? '<button class="note-link" onclick="openEditNoteSheet(\'' + comp.id + '\');event.stopPropagation()">add note</button>' : '')) +
        '</div></div>';
    });
  }
```

Note: moved the `onclick="uncompleteTask()"` from the card div to just the checkbox, so clicking "add note" / "edit" doesn't trigger uncomplete.

- [ ] **Step 4: Update Done view with note links**

Find the done entry HTML in renderDone() (~line 2678-2680):

```javascript
      html += '<div class="done-entry"><div class="done-entry-name">' + esc(name) + '</div>' +
        '<div class="done-entry-meta">' + energyChip(energy) + ' <span class="done-entry-time">' + formatTime(c.completedAt) + '</span></div>' +
        (c.note ? '<div class="done-entry-note">"' + esc(c.note) + '"</div>' : '') + '</div>';
```

Replace with:

```javascript
      html += '<div class="done-entry"><div class="done-entry-name">' + esc(name) + '</div>' +
        '<div class="done-entry-meta">' + energyChip(energy) + ' <span class="done-entry-time">' + formatTime(c.completedAt) + '</span></div>' +
        (c.note ? '<div class="done-entry-note">"' + esc(c.note) + '" <button class="note-link" onclick="openEditNoteSheet(\'' + c.id + '\')">edit</button></div>' : '<button class="note-link" onclick="openEditNoteSheet(\'' + c.id + '\')">add note</button>') + '</div>';
```

- [ ] **Step 5: Test notes on completions**

1. Complete a non-pile task on Today
2. In "done today" section, card should show "add note" link
3. Tap "add note" — note sheet opens, save a note
4. Card now shows the note text + "edit" link
5. Same behavior works in the Done tab

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add notes on all completions with add/edit from Today and Done views"
```

---

### Task 6: Final Verification and Deploy

- [ ] **Step 1: Full smoke test**

Open the app and verify all 5 features:
1. Inbox task cards have "today" button — tapping schedules immediately
2. "triage all" button appears with 2+ inbox tasks — swipe-through works
3. Today splits into regular tasks above, "recurring" label, pile tasks below
4. Template preview shows 7 day buttons — future dates create correct placements
5. Done cards in Today and Done views show "add note" / "edit" links

- [ ] **Step 2: Deploy**

```bash
cp "index.html" "//192.168.5.204/docker/pocketbase/pb_public/flowstate/index.html"
```
