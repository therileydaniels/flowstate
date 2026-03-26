# Phase 3: Polish — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add celebration moments (confetti + milestone messages) and offline resilience (read-only cache with retry) to FlowState.

**Architecture:** CSS-only confetti animation triggered by JS. Milestone messages reuse existing toast system. Offline mode caches data to localStorage and disables mutations via a CSS body class. All changes in single `index.html` file.

**Tech Stack:** Vanilla JS, CSS keyframes, localStorage

---

### Task 1: Confetti Animation

**Files:**
- Modify: `index.html` (CSS, new JS functions)

- [ ] **Step 1: Add confetti CSS**

Add before the closing `</style>` tag:

```css
    /* === CONFETTI === */
    #confetti {
      position: fixed;
      inset: 0;
      z-index: 300;
      pointer-events: none;
      overflow: hidden;
    }
    .confetti-piece {
      position: absolute;
      top: -10px;
      width: 8px;
      height: 8px;
      animation: confetti-fall 2s ease-in forwards;
    }
    .confetti-piece:nth-child(odd) { border-radius: 50%; }
    .confetti-piece:nth-child(3n) { width: 6px; height: 10px; }
    @keyframes confetti-fall {
      0% { transform: translateY(0) rotate(0deg); opacity: 1; }
      100% { transform: translateY(110vh) rotate(720deg); opacity: 0; }
    }
```

- [ ] **Step 2: Add confetti JS function**

Add in the JS section, after the undo toast functions (after `showUndoToast_msg`):

```javascript
// =======================================
// CONFETTI
// =======================================
var confettiColors = ['#C75450', '#C49A2A', '#5A9E6F', '#5E5BAE', '#A8A6D8'];

function fireConfetti(count) {
  var existing = document.getElementById('confetti');
  if (existing) existing.remove();
  var container = document.createElement('div');
  container.id = 'confetti';
  for (var i = 0; i < count; i++) {
    var piece = document.createElement('div');
    piece.className = 'confetti-piece';
    piece.style.left = (10 + Math.random() * 80) + '%';
    piece.style.animationDelay = (Math.random() * 0.5) + 's';
    piece.style.backgroundColor = confettiColors[Math.floor(Math.random() * confettiColors.length)];
    container.appendChild(piece);
  }
  document.getElementById('app').appendChild(container);
  setTimeout(function() {
    var el = document.getElementById('confetti');
    if (el) el.remove();
  }, 2500);
}
```

- [ ] **Step 3: Test confetti**

Temporarily add `fireConfetti(30);` to the end of `init()`, reload, verify colored pieces fall from top. Remove the test call.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add CSS confetti animation system"
```

---

### Task 2: Milestone Messages and Celebration Triggers

**Files:**
- Modify: `index.html` (state, milestone check function, hooks in completeTask/finishCompletion)

- [ ] **Step 1: Add celebration state**

Add after the `confettiColors` and `fireConfetti` function:

```javascript
// =======================================
// CELEBRATIONS
// =======================================
let celebratedMilestones = new Set();

function checkCelebrations(completedTaskEnergy) {
  var placements = getTodayPlacements();
  var total = placements.length;
  if (total === 0) return;
  var completedCount = placements.filter(function(p) { return isPlacementComplete(p.id); }).length;

  // Deep work completion
  if (completedTaskEnergy === 'deep') {
    fireConfetti(25);
    showUndoToast_msg('deep work done... that\'s the hard stuff');
    return;
  }

  // All done
  if (completedCount === total && !celebratedMilestones.has('alldone')) {
    celebratedMilestones.add('alldone');
    fireConfetti(40);
    showUndoToast_msg('you crushed it today');
    return;
  }

  // 50%
  if (completedCount >= total / 2 && completedCount < total && !celebratedMilestones.has('half')) {
    celebratedMilestones.add('half');
    showUndoToast_msg('halfway there... keep going');
    return;
  }

  // First task
  if (completedCount === 1 && !celebratedMilestones.has('first')) {
    celebratedMilestones.add('first');
    showUndoToast_msg('off to a good start');
    return;
  }
}
```

- [ ] **Step 2: Hook celebrations into completeTask()**

In `completeTask()`, after `if (placementId) showUndoToast(placementId);` (the last line before the closing `}`), add:

```javascript
  checkCelebrations(task.energy);
```

IMPORTANT: The celebration check must come AFTER `showUndoToast()` because `checkCelebrations` may call `showUndoToast_msg` which clears the previous toast. Actually, this means the undo toast would be replaced by the celebration message. We need to delay the celebration slightly.

Instead, change the approach: add a small delay so undo toast shows first, then celebration replaces it after 1 second:

```javascript
  var taskEnergy = task.energy;
  setTimeout(function() { checkCelebrations(taskEnergy); }, 1000);
```

So the full end of `completeTask()` becomes:

```javascript
  await loadAll();
  render();
  if (placementId) showUndoToast(placementId);
  var taskEnergy = task.energy;
  setTimeout(function() { checkCelebrations(taskEnergy); }, 1000);
```

- [ ] **Step 3: Hook celebrations into finishCompletion()**

In `finishCompletion()`, after `if (placementId) showUndoToast(placementId);`, add:

```javascript
  var taskEnergy = task ? task.energy : null;
  setTimeout(function() { checkCelebrations(taskEnergy); }, 1000);
```

So the end of `finishCompletion()` becomes:

```javascript
  await loadAll();
  if (placementId) showUndoToast(placementId);
  var taskEnergy = task ? task.energy : null;
  setTimeout(function() { checkCelebrations(taskEnergy); }, 1000);
```

Note: `finishCompletion()` does NOT call `render()` itself — the callers (`saveNote`/`skipNote`) do. The celebrations read from `data` which is already refreshed by `loadAll()`.

- [ ] **Step 4: Test celebrations**

1. Complete first task of the day — should see "off to a good start" after 1 second
2. Complete enough tasks to hit 50% — "halfway there... keep going"
3. Complete a deep work task — confetti (25 pieces) + "deep work done..."
4. Complete all tasks — confetti (40 pieces) + "you crushed it today"
5. Each milestone only fires once per page load

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add milestone messages and confetti celebrations on task completion"
```

---

### Task 3: Offline Mode — Cache and Init

**Files:**
- Modify: `index.html` (CSS, HTML, loadAll, init)

- [ ] **Step 1: Add offline CSS**

Add before the closing `</style>` tag:

```css
    /* === OFFLINE MODE === */
    #offline-banner {
      display: none;
      text-align: center;
      padding: 8px 16px;
      background: var(--bg2);
      font-family: 'Quicksand', sans-serif;
      font-size: 13px;
      font-weight: 500;
      color: var(--text-muted);
      cursor: pointer;
      border-bottom: 1px solid var(--border);
    }
    body.offline .input-bar,
    body.offline .btn-template,
    body.offline .card-actions,
    body.offline .drag-handle,
    body.offline #wsid-fab,
    body.offline .btn-triage-all { display: none; }
    body.offline .card-checkbox { pointer-events: none; opacity: 0.4; }
```

- [ ] **Step 2: Add offline banner HTML**

Find the `</header>` tag (~line 1323). Add AFTER it and BEFORE `<main>`:

```html
      <div id="offline-banner" onclick="retryConnection()">working offline... tap to retry</div>
```

- [ ] **Step 3: Add offline state and functions**

Add in the JS section, after the celebrations code:

```javascript
// =======================================
// OFFLINE MODE
// =======================================
let isOffline = false;

function saveCache() {
  try {
    localStorage.setItem('flowstate_cache', JSON.stringify(data));
  } catch(e) {
    // localStorage full or unavailable, silently skip
  }
}

function loadCache() {
  try {
    var cached = localStorage.getItem('flowstate_cache');
    if (cached) {
      data = JSON.parse(cached);
      return true;
    }
  } catch(e) {}
  return false;
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

async function retryConnection() {
  try {
    await loadAll();
    exitOfflineMode();
    render();
  } catch(e) {
    // still offline
  }
}
```

- [ ] **Step 4: Add saveCache() call to loadAll()**

In `loadAll()`, find the end of the try block (after the templates mapping, before the `catch`). Add `saveCache();` as the last line inside the try block.

Find (the closing of the try block in loadAll):
```javascript
    data.templates = templates.map(function(t) {
      return {
        id: t.id,
        name: t.name,
        icon: t.icon,
        energy: t.energy,
        tasks: t.tasks || []
      };
    });
```

Add after it (still inside the try):
```javascript
    saveCache();
```

- [ ] **Step 5: Update init() to use cache on failure**

Find `init()` (~line 3271):

```javascript
async function init() {
  try {
    await seedIfEmpty();
    await loadAll();
    document.getElementById('loading').style.display = 'none';
    document.getElementById('app-content').classList.add('loaded');
    render();
  } catch(e) {
    console.error('Init failed:', e);
    document.getElementById('loading').textContent = 'hmm something went wrong... check your connection and try again';
  }
}
```

Replace with:

```javascript
async function init() {
  try {
    await seedIfEmpty();
    await loadAll();
    document.getElementById('loading').style.display = 'none';
    document.getElementById('app-content').classList.add('loaded');
    if (isOffline) exitOfflineMode();
    render();
  } catch(e) {
    console.error('Init failed:', e);
    if (loadCache()) {
      document.getElementById('loading').style.display = 'none';
      document.getElementById('app-content').classList.add('loaded');
      enterOfflineMode();
      render();
    } else {
      document.getElementById('loading').textContent = 'hmm something went wrong... check your connection and try again';
    }
  }
}
```

- [ ] **Step 6: Test offline mode**

1. Load the app normally — verify it works, check that `flowstate_cache` exists in localStorage
2. Change PocketBase URL temporarily to a bad address (e.g., `http://192.168.5.204:9999`), reload
3. App should load from cache with "working offline... tap to retry" banner
4. Input bars, buttons, WSID fab should be hidden
5. Cards render, checkboxes are dimmed
6. Fix the URL back, tap "tap to retry" — banner disappears, full functionality restored

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: add offline resilience with read-only cache and retry banner"
```

---

### Task 4: Final Verification and Deploy

- [ ] **Step 1: Full smoke test**

Open the app and verify both features:
1. Complete a deep work task — confetti + "deep work done" message
2. Complete first task — "off to a good start"
3. Hit 50% — "halfway there"
4. Complete all — big confetti + "you crushed it today"
5. Refresh — milestones reset, can trigger again
6. App loads with cached data when PocketBase is unreachable
7. Retry button reconnects successfully

- [ ] **Step 2: Deploy**

```bash
cp "index.html" "//192.168.5.204/docker/pocketbase/pb_public/flowstate/index.html"
```
