# Floor Tracker

A single-page floor tracking tool for managing Line Units (LUs) across physical floor locations. Built as a single `index.html` file deployed via GitHub Pages.

## Architecture

**Everything lives in `index.html`.** There are no build tools, no dependencies, no package.json, no bundler. CSS, HTML, and JS are all inline. To develop: edit the file, commit, push — GitHub Pages serves it directly.

GitHub Pages serves `index.html` from the root of the `main` branch. No configuration file (like `_config.yml`) is needed.

State is persisted to `localStorage` under the key `floor_tracker_v1`.

## Domain Model

**Locations** (`LOCATIONS` array): Physical spots on the floor. Each has an `id`, `label`, `group`, and `accepts` array of unit types. Groups: Safran Pogos, Max Move, FD Spots, PAX Pogos, Complete Storage.

**Unit types:**
- `FD` — First Down unit. Cannot carry a tool in strict mode.
- `PAX` — Standalone PAX unit. Can carry a `1ME` tool. Cannot carry `2ME`.
- `COMPLETE` — Mated unit (FD + PAX together). Can carry a `2ME` tool. Cannot carry `1ME`.

**Tools:** `none`, `1ME`, `2ME`. Tool readiness is tracked separately from unit readiness.

**State shape:**
```js
{
  spots: { [locationId]: { lu, type, tool, toolReady, unitReady, destination } },
  loadOrder: [ { id, lu, note } ],
  strict: bool,
  quickView: bool,
}
```

## Key Business Rules (Strict Mode)

- FD spots only accept `FD` units; PAX Pogos accept `PAX` or `COMPLETE`; Safran/Max Move/Complete Storage accept `COMPLETE` only.
- **Max Move** always implies: type=COMPLETE, unitReady=true, tool=2ME, toolReady=true. The UI hides the type/tool/ready pickers for this spot.
- **PAX Pogo + COMPLETE unit** implies: unitReady=true, tool=2ME, toolReady=true (same reason — it's bolted down).
- **Complete Storage** strips the 2ME when a unit is placed there (tool auto-set to `none`).
- These normalization rules are applied on save and on move, not just in the modal.

## Code Structure

The JS in `index.html` is organized into these sections (marked with `// ===` comments):

1. **CONFIG** — `LOCATIONS`, `GROUP_ORDER`, `DESTINATIONS` constants
2. **STATE** — `state` object, `loadState()`, `saveState()`
3. **HELPERS** — `uid()`, `getSpot()`, `getAllUnits()`, `validateUnit()`, `escapeHtml()`
4. **RENDER: BOARD** — `renderCounters()`, `renderBoard()`, `renderSpot()`
5. **RENDER: LOAD ORDER** — `renderLoad()`
6. **MODAL HELPERS** — `closeModal()`, `openModal()`
7. **SPOT EDIT MODAL** — `openSpotModal()`, `renderSpotModal()`
8. **LOAD PICKER MODAL** — `openPickerModal()`
9. **CUSTOM LOAD ENTRY MODAL** — `openCustomLoadModal()`
10. **MOVE MODAL** — `openMoveModal()`
11. **TABS + RULES TOGGLE** — `setTab()`, `refreshRulesToggle()`, etc.
12. **UNDO STACK** — `pushUndo()`, `performUndo()` (10-action limit)
13. **INIT** — wires up event listeners, calls `loadState()` + initial render

## Patterns

- **Re-render on change**: Every mutation calls `saveState()` then re-renders the affected view. There's no virtual DOM or diffing — innerHTML is replaced each time.
- **editDraft**: The spot edit modal uses a module-level `editDraft` object and `editLocationId`. Changes update the draft and call `renderSpotModal()` to re-render the modal in place.
- **Undo via snapshot**: `pushUndo()` deep-clones `state.spots` and `state.loadOrder` before any mutation. `performUndo()` restores from the top of the stack.
- **escapeHtml**: All user-supplied strings rendered into HTML go through `escapeHtml()`. Don't skip this.
- **CSS variables**: All colors defined as `--var` on `:root`. Theme is dark (`--bg: #0a0e14`). Accent is cyan (`--accent: #00d9ff`).

## Adding a New Location

1. Add an entry to `LOCATIONS` with a unique `id`, `label`, `group`, and `accepts` array.
2. If it's a new group, add the group name to `GROUP_ORDER` in the desired display position.
3. That's it — rendering is data-driven.

## Deployment

Push to `main`. GitHub Pages serves `index.html` automatically. There is no CI, no build step, no preview environment.
