# Reorg Planning: Edit, Move & Save

**Date:** 2026-04-23
**Status:** Approved

## Overview

Add interactive reorg planning capabilities to the org chart viewer. Users can edit titles, move people between managers, visualize what changed, and export the modified org as a new CSV.

## Scope

**In scope:**
- Edit a person's title via a side panel
- Move a person to a different manager via a dropdown
- Visual diff highlighting (moved, edited, both)
- Save modified org chart as a new CSV file
- Reset to original state

**Out of scope (for now):**
- Add/remove people
- Drag-and-drop moves
- Name editing (name is the immutable key identifier)

## Side Panel

A 350px panel slides in from the right when any person's card is clicked. The main chart area shrinks to accommodate it (no overlap).

### Panel contents

| Field | Type | Behavior |
|-------|------|----------|
| Name | Read-only text | Displayed as a label, not editable |
| Title | Text input | Editable, updates on save |
| Manager | `<select>` dropdown | All people in the org + "(No Manager)" option |
| Direct reports | Read-only list | Shows current direct reports for context |

### Panel actions

- **Save** — applies changes to in-memory data, re-renders tree
- **Cancel** — discards unsaved changes, closes panel
- **X (close)** — same as Cancel
- **Clicking a different card** — switches panel to that person; prompts if unsaved changes exist

### Constraints

- The dropdown excludes the selected person themselves (can't be your own manager)
- The dropdown also excludes anyone in the selected person's subtree (prevents circular references)

## Visual Diff

Cards are color-coded based on changes relative to the originally loaded CSV.

### Color scheme

| Change type | Left border | Background |
|-------------|-------------|------------|
| Moved (manager changed) | Amber/orange | Subtle amber tint |
| Edited (title changed) | Purple | Subtle purple tint |
| Moved + Edited | Amber/orange | Subtle purple tint |
| Unchanged (manager) | Blue (existing) | None |
| Unchanged (leaf) | Gray (existing) | None |

### Additional indicators

- **"was under [Original Manager]" subtitle** — shown on moved people's cards in muted text, indicating where they came from
- **Legend** — appears at the top of the chart area once any changes are made; shows what each color means
- **Legend hides** when no changes exist (after reset or fresh load)

## Toolbar Changes

Two new buttons, hidden until the first change is made:

| Button | Behavior |
|--------|----------|
| **Save CSV** | Downloads current org as `reorg-YYYY-MM-DD.csv` using browser download (Blob + anchor click) |
| **Reset** | Reverts to originally loaded data after confirmation prompt ("Discard all changes?") |

The existing **Upload CSV** button warns if unsaved changes exist before replacing the data.

## Data Architecture

### State

```
originalPeople: [{name, title, manager}, ...]   // snapshot at load time, immutable
currentPeople:  [{name, title, manager}, ...]   // working copy, mutated on edits
```

### Key functions

| Function | Purpose |
|----------|---------|
| `getChanges()` | Diffs `originalPeople` vs `currentPeople`; returns a Map of name -> {moved, edited} |
| `applyEdit(name, newTitle, newManager)` | Updates `currentPeople`, triggers re-render |
| `resetChanges()` | Deep-copies `originalPeople` back into `currentPeople`, re-renders |
| `exportCSV()` | Serializes `currentPeople` to CSV string, triggers download |
| `openPanel(name)` | Populates and shows the side panel for a given person |
| `closePanel()` | Hides the side panel, restores chart width |

### Render integration

`renderNode` checks each person against `getChanges()` to apply diff styling (border color, background tint, "was under" subtitle). The legend visibility is toggled based on whether `getChanges()` returns any entries.

## File changes

| File | Changes |
|------|---------|
| `index.html` | Add side panel markup, legend, Save CSV & Reset buttons |
| `app.js` | Add state management, panel logic, diff engine, export, edit handlers |
| `style.css` | Add panel styles, diff colors, legend styles, chart-area transition |

## UX Details

- Panel open/close is animated with a CSS transition (slide in/out)
- Chart area transitions its width smoothly when panel opens/closes
- Cards have a hover cursor indicating they're clickable
- Selected card gets a highlight outline to show which person the panel refers to
- Dropdown shows people sorted alphabetically for easy lookup
- Save CSV filename format: `reorg-YYYY-MM-DD.csv`
