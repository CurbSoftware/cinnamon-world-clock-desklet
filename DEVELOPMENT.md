# World Clock Desklet - Development Guide

## Overview

The World Clock desklet displays a homogeneous grid of clocks on the Cinnamon desktop. Each tile has its own name and timezone; time/date formats and font sizes are desklet-wide. Layout and tile chrome match the Workspace Grid desklet.

**UUID**: `cinnamon-world-clock-desklet@curbsoftware`

**Type**: Cinnamon Desklet

---

## Features

- Grid of clock tiles (time, date, label) with auto or fixed rows×cols
- Trailing `+` tile to add a clock
- Right-click a tile → Edit… / Remove / Add clock
- Shared strftime formats and font sizes via Cinnamon desklet settings
- `timezone === "local"` tiles use the `:outlined` pseudo-class
- Live update every second (label text only; the grid is not rebuilt)
- Last clock cannot be removed; named clocks can prompt before remove

---

## Project Structure

```
cinnamon-world-clock-desklet@curbsoftware/
├── desklet.js              # Grid UI, tick, per-tile menus
├── clockActions.js         # Pure layout maths + clock-list mutations
├── clockDialog.js          # Modal name + timezone dialog
├── metadata.json           # UUID, name, legacy migration seeds
├── settings-schema.json    # Layout, management, display, style
├── stylesheet.css          # Tile chrome (workspace-grid style)
├── README.md
└── DEVELOPMENT.md
```

Cinnamon cannot import across xlets. `computeGridDims` / `planCells` are copied from the Workspace Grid desklet rather than shared.

---

## Development Setup

```bash
./dev-tools/install-extensions.sh -m symlink -n '*world-clock*'
```

The install directory name must match the UUID: `~/.local/share/cinnamon/desklets/cinnamon-world-clock-desklet@curbsoftware/`.

Reload without restarting Cinnamon:

```bash
gdbus call --session --dest org.Cinnamon --object-path /org/Cinnamon \
  --method org.Cinnamon.ReloadXlet 'cinnamon-world-clock-desklet@curbsoftware' DESKLET
```

Logs:

```bash
tail -f ~/.xsession-errors | grep -i world-clock
```

---

## Architecture

```
Desklet.Desklet
    └── MyDesklet
            ├── _rebuildGrid()          St.Table of clock / add tiles
            ├── _scheduleUpdate()       1s tick → ClockWidget.update()
            └── _cleanup()              destroy + on_desklet_removed

ClockWidget
    └── St.Button.world-clock-tile
            └── St.BoxLayout
                    ├── time label
                    ├── date label
                    └── timezone / name label

clockActions.js     computeGridDims, planCells, normalize/add/update/remove
clockDialog.js      promptClock(title, {name, timezone}, callback)
```

Modules are loaded lazily:

```javascript
const dir = imports.ui.deskletManager.desklets[uuid];
ClockActions = dir.clockActions;
ClockDialog = dir.clockDialog;
```

### Grid rebuild

Settings or the `clocks` array change → `_rebuildGrid()`:

1. Destroy the tile menu (parented to `Main.uiGroup`).
2. `destroy_all_children()` on the container.
3. `computeGridDims` + `planCells` (clock cells + optional `+`).
4. Homogeneous `St.Table`; clock tiles are `St.Button` with `ButtonMask.ONE | THREE`.
5. `_updateClocks()` then `_updateHighlight()`.

The 1s tick only writes label text. Do not rebuild from the tick.

### Menu actions

`PopupMenuBase` closes the menu after the `activate` handler. Destroying the menu or opening a modal in that handler fights the grab. Defer with `Mainloop.timeout_add(0)` (`_deferAction`), not `idle_add`.

---

## Settings

| Key | Type | Default | Notes |
|-----|------|---------|-------|
| `layout-mode` | combobox | `auto` | `auto` or `fixed` |
| `fixed-rows` / `fixed-cols` | spinbutton | `2` | Shown when layout is fixed |
| `enable-clock-editing` | checkbox | `true` | Menus + add tile |
| `show-add-tile` | checkbox | `true` | Trailing `+` |
| `confirm-remove` | checkbox | `true` | Prompt for named clocks |
| `time-format` / `date-format` | entry | `%H:%M:%S` / `%A, %e %B` | strftime via `toLocaleFormat` |
| `time-size` / `date-size` / `timezone-size` | spinbutton | `40` / `15` / `12` | pt, shared by all tiles |
| `tile-spacing` | spinbutton | `4` | Button margin px |
| `width` / `height` | spinbutton | `600` / `400` | Desklet bounds |
| `clocks` | generic | one Local clock | `[{ id, name, timezone }]` |

Per-clock records are `{ id, name, timezone }`. `"local"` or an IANA id. Legacy per-clock format/size fields are dropped on load.

`metadata.json` still carries old single-clock fields (`timezone`, formats, sizes) as migration seeds when the stored list is empty or the placeholder `id: "default"` clock.

---

## Clock list API (`clockActions.js`)

All functions are pure (no St/Clutter):

- `computeGridDims(cellCount, mode, fixedRows, fixedCols)` → `{ rows, cols }`
- `planCells(clockCount, showAddTile, rows, cols)` → `[{ kind: "clock"|"add", index }]`
- `normalizeClockList(list, { generateId })` → `{ clocks, changed }`
- `buildClock(values, existingId, opts)` → `{ id, name, timezone }`
- `addClock` / `updateClock` / `removeClock` (remove refuses the last clock)
- `canAdd` (cap 36) / `canRemove` / `isValidIndex` / `getClockLabel`

---

## Testing

```bash
gjs dev-tools/test-clock-actions.js
```

Headless coverage: grid dims, `planCells` overflow (add tile stays last), normalize/add/update/remove guards. Widget behaviour needs a live Cinnamon session.

### Manual check

1. Install with the symlink command above and reload the desklet.
2. Auto vs fixed grid; width/height/spacing/font sizes in settings.
3. `+` add; per-tile edit/remove; last-clock refuse.
4. Local tile outlined; 1s tick without flicker.

---

## Gotchas

- Tile menus on `Main.uiGroup` survive `destroy_all_children()` — destroy them explicitly.
- `Desklet.destroy()` emits `destroy` immediately but defers `on_desklet_removed()` by a 500 ms fade. Hook both; `_cleanup()` is idempotent.
- Never `Mainloop.idle_add` for user-visible actions; use `timeout_add(0)`.
- Invalid IANA ids are logged and that tile falls back to local time.
- `toLocaleFormat` is a GJS `Date` extension (strftime).

### Valid timezone examples

```
local
America/New_York     America/Los_Angeles    America/Chicago
Europe/London        Europe/Paris           Europe/Berlin
Asia/Tokyo           Asia/Shanghai          Asia/Singapore
Australia/Sydney     Pacific/Auckland
```
