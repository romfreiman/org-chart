# Reorg Planning — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add interactive reorg planning to the org chart viewer — edit titles, move people between managers, visualize diffs, and export as CSV.

**Architecture:** Extend the existing vanilla JS app with in-memory state management (original vs. current people arrays), a slide-in side panel for editing, CSS-based diff highlighting, and browser-native CSV download. No new dependencies.

**Tech Stack:** Vanilla HTML/CSS/JS (no build step, no dependencies)

**Design doc:** `docs/plans/2026-04-23-reorg-planning-design.md`

---

### Task 1: State Management

Add the core data layer that tracks original and current org state, plus a diff engine.

**Files:**
- Modify: `app.js:1-10` (add state variables after existing element refs)
- Modify: `app.js:433-451` (update `loadCSVText` to snapshot state)

**Step 1: Add state variables and helper functions**

After line 9 (`const collapseBtn = ...`), add:

```javascript
let originalPeople = [];
let currentPeople = [];
let selectedPerson = null;
```

**Step 2: Add `getChanges()` diff function**

After the state variables, add:

```javascript
function getChanges() {
  const changes = new Map();
  const origMap = new Map(originalPeople.map(function (p) {
    return [p.name, p];
  }));

  for (var i = 0; i < currentPeople.length; i++) {
    var curr = currentPeople[i];
    var orig = origMap.get(curr.name);
    if (!orig) continue;

    var moved = curr.manager !== orig.manager;
    var edited = curr.title !== orig.title;

    if (moved || edited) {
      changes.set(curr.name, {
        moved: moved,
        edited: edited,
        originalManager: orig.manager
      });
    }
  }

  return changes;
}

function hasChanges() {
  return getChanges().size > 0;
}

function getSubtreeNames(name) {
  var names = [];
  var person = currentPeople.find(function (p) { return p.name === name; });
  if (!person) return names;

  var directReports = currentPeople.filter(function (p) { return p.manager === name; });
  for (var i = 0; i < directReports.length; i++) {
    names.push(directReports[i].name);
    names = names.concat(getSubtreeNames(directReports[i].name));
  }
  return names;
}
```

**Step 3: Update `loadCSVText` to snapshot state**

Replace the existing `loadCSVText` function (lines 433-452) with:

```javascript
function loadCSVText(text) {
  if (!text || text.trim() === "") {
    showError("Please upload a CSV file");
    return;
  }

  var result = parseCSV(text);

  if (result.error === "empty") {
    showError("Please upload a CSV file");
  } else if (result.error === "columns") {
    showError("CSV must contain columns: Name, Title, Manager");
  } else if (result.error === "no_data") {
    showError("No valid data found in CSV");
  } else {
    originalPeople = result.people.map(function (p) {
      return { name: p.name, title: p.title, manager: p.manager };
    });
    currentPeople = result.people.map(function (p) {
      return { name: p.name, title: p.title, manager: p.manager };
    });
    selectedPerson = null;
    closePanel();
    rebuildAndRender();
    updateToolbarButtons();
  }
}

function rebuildAndRender() {
  var roots = buildTree(currentPeople);
  renderTree(roots);
  applyDefaultExpansion(getDefaultDepth());
}
```

**Step 4: Verify the app still loads and displays correctly**

Open `index.html` in a browser with `?file=sample.csv`. The chart should render identically to before.

**Step 5: Commit**

```
feat: add state management and diff engine for reorg planning
```

---

### Task 2: HTML Structure — Side Panel, Legend, Toolbar Buttons

Add the DOM structure for the side panel, change legend, and new toolbar buttons.

**Files:**
- Modify: `index.html:11-22`

**Step 1: Add Save CSV and Reset buttons to the toolbar**

After the Collapse All button (line 17), add:

```html
<button id="save-csv-btn" class="btn btn-save" hidden>Save CSV</button>
<button id="reset-btn" class="btn btn-reset" hidden>Reset</button>
```

**Step 2: Add the legend markup**

After the opening `<main id="chart">` tag (line 20), add:

```html
<div id="change-legend" class="change-legend" hidden>
  <span class="legend-item legend-moved">Moved</span>
  <span class="legend-item legend-edited">Edited</span>
  <span class="legend-item legend-both">Moved + Edited</span>
</div>
```

**Step 3: Add the side panel markup**

After `</main>` (before the script tag), add:

```html
<aside id="edit-panel" class="edit-panel" hidden>
  <div class="panel-header">
    <h2>Edit Person</h2>
    <button id="panel-close-btn" class="panel-close" aria-label="Close">&times;</button>
  </div>
  <div class="panel-body">
    <div class="panel-field">
      <label>Name</label>
      <div id="panel-name" class="panel-readonly"></div>
    </div>
    <div class="panel-field">
      <label for="panel-title">Title</label>
      <input type="text" id="panel-title" class="panel-input">
    </div>
    <div class="panel-field">
      <label for="panel-manager">Manager</label>
      <select id="panel-manager" class="panel-select"></select>
    </div>
    <div class="panel-field">
      <label>Direct Reports</label>
      <ul id="panel-reports" class="panel-reports-list"></ul>
    </div>
  </div>
  <div class="panel-actions">
    <button id="panel-save-btn" class="btn btn-panel-save">Save</button>
    <button id="panel-cancel-btn" class="btn btn-panel-cancel">Cancel</button>
  </div>
</aside>
```

**Step 4: Verify HTML renders without errors**

Open in browser — the panel should be hidden, legend hidden, new buttons hidden. Existing UI unchanged.

**Step 5: Commit**

```
feat: add side panel, legend, and toolbar button markup
```

---

### Task 3: CSS — Side Panel, Diff Colors, Legend

Add all the styling for the new features.

**Files:**
- Modify: `style.css` (append new sections)

**Step 1: Add chart area transition and panel-open adjustment**

Append to `style.css`:

```css
/* ── Chart area transition for panel ── */
main#chart {
  transition: margin-right 0.25s ease;
}

body.panel-open main#chart {
  margin-right: 350px;
}
```

**Step 2: Add side panel styles**

```css
/* ── Side panel ── */
.edit-panel {
  position: fixed;
  top: 56px;
  right: 0;
  bottom: 0;
  width: 350px;
  background: #fff;
  border-left: 1px solid #e2e8f0;
  box-shadow: -2px 0 8px rgba(0, 0, 0, 0.08);
  z-index: 90;
  display: flex;
  flex-direction: column;
  transform: translateX(100%);
  transition: transform 0.25s ease;
}

.edit-panel.visible {
  transform: translateX(0);
}

.panel-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 20px;
  border-bottom: 1px solid #e2e8f0;
}

.panel-header h2 {
  font-size: 16px;
  font-weight: 600;
  color: #2d3748;
}

.panel-close {
  background: none;
  border: none;
  font-size: 22px;
  color: #a0aec0;
  cursor: pointer;
  line-height: 1;
  padding: 4px;
}

.panel-close:hover {
  color: #2d3748;
}

.panel-body {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
}

.panel-field {
  margin-bottom: 20px;
}

.panel-field label {
  display: block;
  font-size: 12px;
  font-weight: 600;
  color: #718096;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 6px;
}

.panel-readonly {
  font-size: 15px;
  font-weight: 600;
  color: #2d3748;
  padding: 8px 0;
}

.panel-input,
.panel-select {
  width: 100%;
  padding: 8px 12px;
  font-size: 14px;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  color: #2d3748;
  background: #fff;
  font-family: inherit;
}

.panel-input:focus,
.panel-select:focus {
  outline: none;
  border-color: #4299e1;
  box-shadow: 0 0 0 3px rgba(66, 153, 225, 0.15);
}

.panel-reports-list {
  list-style: none;
  font-size: 13px;
  color: #4a5568;
}

.panel-reports-list li {
  padding: 4px 0;
  border-bottom: 1px solid #f7fafc;
}

.panel-reports-list:empty::after {
  content: "None";
  color: #a0aec0;
  font-style: italic;
}

.panel-actions {
  display: flex;
  gap: 8px;
  padding: 16px 20px;
  border-top: 1px solid #e2e8f0;
}

.btn-panel-save {
  flex: 1;
  background: #4299e1;
  color: #fff;
}

.btn-panel-cancel {
  flex: 1;
  background: transparent;
  color: #718096;
  border: 1px solid #e2e8f0;
}

.btn-panel-cancel:hover {
  background: #f7fafc;
}
```

**Step 3: Add diff color styles**

```css
/* ── Diff highlighting ── */
.node-card.diff-moved {
  border-left: 3px solid #dd6b20 !important;
  background: #fffaf0;
}

.node-card.diff-edited {
  border-left: 3px solid #805ad5 !important;
  background: #faf5ff;
}

.node-card.diff-both {
  border-left: 3px solid #dd6b20 !important;
  background: #faf5ff;
}

.node-card .was-under {
  font-size: 11px;
  color: #dd6b20;
  margin-top: 4px;
  font-style: italic;
}

.node-card.selected {
  outline: 2px solid #4299e1;
  outline-offset: 2px;
}

.node-card {
  cursor: pointer;
}
```

**Step 4: Add legend styles**

```css
/* ── Change legend ── */
.change-legend {
  display: flex;
  gap: 16px;
  padding: 8px 16px;
  background: #fff;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  margin-bottom: 16px;
  width: fit-content;
}

.legend-item {
  font-size: 12px;
  font-weight: 500;
  padding: 2px 10px;
  border-radius: 4px;
}

.legend-moved {
  border-left: 3px solid #dd6b20;
  background: #fffaf0;
  color: #dd6b20;
}

.legend-edited {
  border-left: 3px solid #805ad5;
  background: #faf5ff;
  color: #805ad5;
}

.legend-both {
  border-left: 3px solid #dd6b20;
  background: #faf5ff;
  color: #805ad5;
}
```

**Step 5: Add toolbar button styles**

```css
/* ── New toolbar buttons ── */
.btn-save {
  background: #38a169;
  color: #fff;
}

.btn-reset {
  background: transparent;
  color: #fc8181;
  border: 1px solid #fc8181;
}

.btn-reset:hover {
  background: rgba(252, 129, 129, 0.1);
}
```

**Step 6: Add mobile responsive styles for the panel**

Inside the existing `@media (max-width: 767px)` block, add:

```css
.edit-panel {
  width: 100%;
  top: 48px;
}

body.panel-open main#chart {
  margin-right: 0;
}
```

**Step 7: Verify styles**

Open in browser. All existing styling should be unchanged. New elements are hidden.

**Step 8: Commit**

```
feat: add CSS for side panel, diff highlighting, and legend
```

---

### Task 4: Side Panel Logic

Wire up the panel open/close, card click handlers, and form population.

**Files:**
- Modify: `app.js` (add panel element refs, open/close functions, card click handlers)

**Step 1: Add panel element references**

After the existing element refs (line 9), add:

```javascript
var editPanel = document.getElementById("edit-panel");
var panelName = document.getElementById("panel-name");
var panelTitle = document.getElementById("panel-title");
var panelManager = document.getElementById("panel-manager");
var panelReports = document.getElementById("panel-reports");
var panelCloseBtn = document.getElementById("panel-close-btn");
var panelSaveBtn = document.getElementById("panel-save-btn");
var panelCancelBtn = document.getElementById("panel-cancel-btn");
var saveCsvBtn = document.getElementById("save-csv-btn");
var resetBtn = document.getElementById("reset-btn");
var legendEl = document.getElementById("change-legend");
```

**Step 2: Add `openPanel(name)` function**

```javascript
function openPanel(name) {
  var person = currentPeople.find(function (p) { return p.name === name; });
  if (!person) return;

  selectedPerson = name;

  // Populate fields
  panelName.textContent = person.name;
  panelTitle.value = person.title;

  // Populate manager dropdown — exclude self and subtree
  var excluded = getSubtreeNames(name);
  excluded.push(name);

  panelManager.innerHTML = "";

  var noMgrOption = document.createElement("option");
  noMgrOption.value = "";
  noMgrOption.textContent = "(No Manager)";
  panelManager.appendChild(noMgrOption);

  var sortedPeople = currentPeople
    .filter(function (p) { return excluded.indexOf(p.name) === -1; })
    .sort(function (a, b) { return a.name.localeCompare(b.name); });

  for (var i = 0; i < sortedPeople.length; i++) {
    var opt = document.createElement("option");
    opt.value = sortedPeople[i].name;
    opt.textContent = sortedPeople[i].name;
    panelManager.appendChild(opt);
  }

  panelManager.value = person.manager;

  // Populate direct reports
  panelReports.innerHTML = "";
  var reports = currentPeople.filter(function (p) { return p.manager === name; });
  for (var j = 0; j < reports.length; j++) {
    var li = document.createElement("li");
    li.textContent = reports[j].name + " — " + reports[j].title;
    panelReports.appendChild(li);
  }

  // Show panel
  editPanel.removeAttribute("hidden");
  requestAnimationFrame(function () {
    editPanel.classList.add("visible");
  });
  document.body.classList.add("panel-open");

  // Highlight selected card
  highlightSelectedCard();
}
```

**Step 3: Add `closePanel()` function**

```javascript
function closePanel() {
  selectedPerson = null;
  editPanel.classList.remove("visible");
  document.body.classList.remove("panel-open");

  // Remove selected highlight
  var prev = chartEl.querySelector(".node-card.selected");
  if (prev) prev.classList.remove("selected");

  setTimeout(function () {
    if (!editPanel.classList.contains("visible")) {
      editPanel.setAttribute("hidden", "");
    }
  }, 250);
}

function highlightSelectedCard() {
  var prev = chartEl.querySelector(".node-card.selected");
  if (prev) prev.classList.remove("selected");

  if (!selectedPerson) return;

  var cards = chartEl.querySelectorAll(".node-card");
  for (var i = 0; i < cards.length; i++) {
    var nameEl = cards[i].querySelector(".name");
    if (nameEl && nameEl.textContent === selectedPerson) {
      cards[i].classList.add("selected");
      break;
    }
  }
}

function panelHasUnsavedChanges() {
  if (!selectedPerson) return false;
  var person = currentPeople.find(function (p) { return p.name === selectedPerson; });
  if (!person) return false;
  return panelTitle.value !== person.title || panelManager.value !== person.manager;
}
```

**Step 4: Add card click handler to `renderNode`**

In `renderNode`, after `card.appendChild(titleEl)` (around line 150), add a click handler on the card:

```javascript
card.addEventListener("click", function () {
  if (selectedPerson && panelHasUnsavedChanges()) {
    if (!confirm("You have unsaved changes. Discard them?")) return;
  }
  openPanel(node.name);
});
```

**Step 5: Add panel button event listeners**

In the event listeners section, add:

```javascript
panelCloseBtn.addEventListener("click", function () {
  if (panelHasUnsavedChanges()) {
    if (!confirm("You have unsaved changes. Discard them?")) return;
  }
  closePanel();
});

panelCancelBtn.addEventListener("click", function () {
  if (panelHasUnsavedChanges()) {
    if (!confirm("You have unsaved changes. Discard them?")) return;
  }
  closePanel();
});
```

**Step 6: Verify panel opens and closes**

Load sample.csv, click a card — panel should slide in. Click X or Cancel — panel should slide out.

**Step 7: Commit**

```
feat: add side panel open/close and card click handlers
```

---

### Task 5: Edit, Move & Toolbar Logic

Wire up Save button, toolbar buttons (Save CSV, Reset), and upload warning.

**Files:**
- Modify: `app.js`

**Step 1: Add Save handler on the panel**

```javascript
panelSaveBtn.addEventListener("click", function () {
  if (!selectedPerson) return;

  var newTitle = panelTitle.value.trim();
  var newManager = panelManager.value;

  currentPeople = currentPeople.map(function (p) {
    if (p.name === selectedPerson) {
      return { name: p.name, title: newTitle, manager: newManager };
    }
    return { name: p.name, title: p.title, manager: p.manager };
  });

  var savedName = selectedPerson;
  rebuildAndRender();
  updateToolbarButtons();
  updateLegend();
  openPanel(savedName);
});
```

**Step 2: Add `updateToolbarButtons()` function**

```javascript
function updateToolbarButtons() {
  var changed = hasChanges();
  saveCsvBtn.hidden = !changed;
  resetBtn.hidden = !changed;
}
```

**Step 3: Add `updateLegend()` function**

```javascript
function updateLegend() {
  legendEl.hidden = !hasChanges();
}
```

**Step 4: Add Save CSV handler**

```javascript
saveCsvBtn.addEventListener("click", function () {
  var lines = ["Name,Title,Manager"];

  for (var i = 0; i < currentPeople.length; i++) {
    var p = currentPeople[i];
    lines.push(csvEscape(p.name) + "," + csvEscape(p.title) + "," + csvEscape(p.manager));
  }

  var csvText = lines.join("\n") + "\n";
  var blob = new Blob([csvText], { type: "text/csv;charset=utf-8;" });
  var url = URL.createObjectURL(blob);

  var today = new Date().toISOString().slice(0, 10);
  var a = document.createElement("a");
  a.href = url;
  a.download = "reorg-" + today + ".csv";
  a.click();
  URL.revokeObjectURL(url);
});

function csvEscape(value) {
  if (value.indexOf(",") !== -1 || value.indexOf('"') !== -1 || value.indexOf("\n") !== -1) {
    return '"' + value.replace(/"/g, '""') + '"';
  }
  return value;
}
```

**Step 5: Add Reset handler**

```javascript
resetBtn.addEventListener("click", function () {
  if (!confirm("Discard all changes?")) return;

  currentPeople = originalPeople.map(function (p) {
    return { name: p.name, title: p.title, manager: p.manager };
  });
  selectedPerson = null;
  closePanel();
  rebuildAndRender();
  updateToolbarButtons();
  updateLegend();
});
```

**Step 6: Add upload warning for unsaved changes**

Modify the upload button click handler to warn:

```javascript
uploadBtn.addEventListener("click", function () {
  if (hasChanges()) {
    if (!confirm("You have unsaved changes. Load a new file?")) return;
  }
  fileInput.click();
});
```

**Step 7: Verify edit and save flow**

1. Load sample.csv
2. Click a person → panel opens
3. Change their title → click Save → card updates
4. Save CSV and Reset buttons appear
5. Click Save CSV → downloads file
6. Click Reset → reverts to original

**Step 8: Commit**

```
feat: add edit/move save, CSV export, and reset functionality
```

---

### Task 6: Diff Rendering

Modify `renderNode` to apply diff styling and show "was under" indicators.

**Files:**
- Modify: `app.js` — inside `renderNode` function

**Step 1: Compute changes before rendering**

At the top of `renderTree`, compute changes once:

```javascript
function renderTree(roots) {
  var changes = getChanges();
  chartEl.innerHTML = "";

  var legend = document.getElementById("change-legend");
  if (legend) {
    legend.hidden = changes.size === 0;
  }

  var wrapper = document.createElement("div");
  wrapper.classList.add("children", "tree-root");

  for (var i = 0; i < roots.length; i++) {
    wrapper.appendChild(renderNode(roots[i], 0, changes));
  }

  chartEl.appendChild(wrapper);
}
```

**Step 2: Update `renderNode` signature to accept changes**

Change `renderNode(node, depth)` to `renderNode(node, depth, changes)`.

Pass `changes` through in the recursive call:

```javascript
childrenContainer.appendChild(renderNode(child, depth + 1, changes));
```

**Step 3: Apply diff classes to cards**

After creating the card element and setting its class, add:

```javascript
var change = changes.get(node.name);
if (change) {
  if (change.moved && change.edited) {
    card.classList.add("diff-both");
  } else if (change.moved) {
    card.classList.add("diff-moved");
  } else if (change.edited) {
    card.classList.add("diff-edited");
  }
}
```

**Step 4: Add "was under" subtitle for moved people**

After the title element and the diff class logic, add:

```javascript
if (change && change.moved) {
  var wasUnder = document.createElement("div");
  wasUnder.className = "was-under";
  var origMgr = change.originalManager || "no one";
  wasUnder.textContent = "\u2190 was under " + origMgr;
  card.appendChild(wasUnder);
}
```

**Step 5: Re-highlight selected card after re-render**

At the end of `rebuildAndRender`, add:

```javascript
function rebuildAndRender() {
  var roots = buildTree(currentPeople);
  renderTree(roots);
  applyDefaultExpansion(getDefaultDepth());
  highlightSelectedCard();
}
```

**Step 6: Verify diff rendering**

1. Load sample.csv
2. Move someone to a different manager → card shows amber border + "was under"
3. Edit someone's title only → card shows purple border
4. Do both → amber border + purple background
5. Reset → all colors revert to normal

**Step 7: Commit**

```
feat: add visual diff highlighting for moved and edited people
```

---

### Task 7: Final Polish & Edge Cases

Handle remaining edge cases and ensure smooth UX.

**Files:**
- Modify: `app.js`

**Step 1: Prevent expand/collapse click events from opening the panel**

The card click handler should not fire when clicking the reports count or toggle elements. These already call `e.stopPropagation()`, so this should work automatically. Verify by clicking the reports text and the +/- toggle — they should expand/collapse without opening the panel.

**Step 2: Update page title**

In `index.html`, change:

```html
<title>Org Chart Viewer</title>
```

to:

```html
<title>Org Chart</title>
```

And update the header h1 text if desired (currently already says "Org Chart").

**Step 3: Full manual test**

1. Load sample.csv via upload
2. Load via `?file=sample.csv`
3. Click person → panel opens with correct data
4. Change title → Save → diff shows purple
5. Change manager → Save → diff shows amber + "was under"
6. Change both → shows combined styling
7. Click different person → panel switches
8. Click X / Cancel → panel closes
9. Save CSV → downloads valid CSV
10. Reset → all changes reverted, panel closes
11. Upload new file with changes → warns before discarding
12. Expand All / Collapse All / Managers Only still work
13. Mobile viewport → panel takes full width

**Step 4: Commit**

```
feat: finalize reorg planning with polish and edge case handling
```
