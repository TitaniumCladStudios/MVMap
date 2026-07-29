# MVMap

A metroidvania map editor. Paint rooms onto grid paper, colour them by area, connect them with doors, and model progression with locks and keys — then grey-box the inside of any room without leaving the tool.

It's a single self-contained `index.html`. No build step, no dependencies, no server, no account.

Built to design an unannounced metroidvania.

---

## Running it

Double-click `index.html`. That's it.

For real IndexedDB storage instead of the localStorage fallback, serve it over HTTP:

```sh
python -m http.server 8080
# then open http://localhost:8080
```

Chrome denies IndexedDB to `file://` pages, so opening the file directly falls back to localStorage. The app detects this and shows an amber notice at the bottom of the window. Everything still works — localStorage is just smaller (~5 MB) and easier to clear by accident. **Export often if you're working from `file://`.**

### Deploying

Commit `index.html` to a repo and enable GitHub Pages on the branch. There's nothing to configure — the file is fully self-contained, and Pages serves over HTTPS so you get proper IndexedDB.

---

## Why locks and keys

The core metroidvania principle is that the world is non-linear and gated. New areas are *locked*, and to get through a gate the player has to go find the thing that opens it — which usually isn't a key at all. It's a movement ability, an item, or a story beat.

So the interesting part of the design isn't the level geometry. It's the dependency graph underneath it, and graph paper can't show you that. MVMap models it directly:

- A **lock** is a concept, not a door. Put the same lock on every gap in the world that the grapple opens.
- A **key** is placed where the player finds it, and is rarely a literal key.
- The relationship is many-to-many. One lock can accept several keys — that's a sequence break you designed on purpose instead of one a speedrunner found for you. One key can open many locks.
- Selecting a lock or key draws dashed arrows from each key's location to every door it opens. Toggle **Show all lock ↔ key links** to see the whole progression graph laid over the geography.
- A lock with doors but no key that opens it is flagged as a soft-lock in the inspector.

---

## Features

**World grid** — unbounded grid paper, heavy line every 8 cells. Pan, zoom to cursor, zoom to fit. HUD shows the cell under the pointer.

**Rooms** — painted as arbitrary cell sets, so an L-shaped shaft, a ring, or a long corridor is *one* room rather than several boxes glued together. Only the silhouette is outlined, so it reads like an in-game map. Drag a room in select mode and its keys — plus the doors it solely owns — move with it; doors shared with a neighbouring room stay anchored.

**Areas** — colour-coded biomes. Rooms inherit the area colour, or override it individually. The room list groups by area. Adding an area opens a room inside it immediately.

**Doors** — placed on the edge *between* cells, not in them. Three kinds: passage, door, and one-way (with a flippable direction arrow). The inspector always names the two rooms a door actually connects.

**Locks and keys** — as above.

**Grey-box view** — double-click a room to open its cells subdivided 4×, 8×, or 16×, with the world-cell boundaries still drawn as heavy lines and the border doors rendered as anchors. Block out the interior with typed boxes: solid, platform, hazard, water, ladder, exit, item, enemy, note. Move, resize, and label them. Changing subdivision rescales existing boxes in place.

**Storage** — autosaves to IndexedDB (localStorage fallback) with a **Maps…** library for multiple projects. Export/import JSON, export the whole map as a PNG.

**Undo/redo** — snapshot-based, covers everything including grey-box edits.

---

## Shortcuts

### Map

| Key | Action |
|---|---|
| `V` | Select / move |
| `B` | Paint cells |
| `R` | Rectangle fill |
| `E` | Erase cells |
| `D` | Door |
| `K` | Key |
| `H` or hold `Space` | Pan |
| `Shift` + paint | Start a new room |
| `Alt` + paint | Erase instead |
| `Alt` + click door | Delete door |
| `F` | Zoom to fit |
| `Enter` | Grey-box selected room |
| `Del` | Delete selection |
| `Ctrl+Z` / `Ctrl+Shift+Z` | Undo / redo |
| `Ctrl+S` | Export JSON |
| `Esc` | Deselect |

### Grey-box

| Key | Action |
|---|---|
| `B` | Draw box |
| `V` | Select / move / resize |
| `E` | Erase box |
| `H` or hold `Space` | Pan |
| `1`–`9` | Pick box type |
| `F` | Zoom to fit |
| `Del` | Delete selected box |
| `Esc` | Deselect, then back to the map |

Mouse: scroll to zoom, middle-drag to pan, double-click a room to grey-box it.

---

## File format

`Export` produces a plain JSON document. It's stable, diffable, and safe to commit next to your game's source.

```jsonc
{
  "id": "map_a1b2c3",
  "version": 1,
  "name": "Ruined Keep",
  "updated": 1753660800000,

  "areas": [
    { "id": "area_x1", "name": "Sunken Cistern", "color": "#4c8dd8" }
  ],

  "rooms": [
    {
      "id": "room_y1",
      "name": "Flooded Shaft",
      "areaId": "area_x1",
      "color": null,              // null = inherit the area colour
      "notes": "Ambush after the water drains.",
      "cells": ["3,4", "4,4", "4,5"]
    }
  ],

  "doors": [
    {
      "id": "door_z1",
      "x": 5, "y": 4,
      "o": "v",                   // "v" or "h" — see edge convention below
      "kind": "door",             // "passage" | "door" | "oneway"
      "dir": 1,                   // one-way only: 1 = +axis, -1 = -axis
      "lockId": "lock_q1"         // null when unlocked
    }
  ],

  "locks": [
    { "id": "lock_q1", "name": "Grapple Gate", "color": "#e0555f",
      "keyIds": ["key_k1", "key_k2"] }   // any one of these opens it
  ],

  "keys": [
    { "id": "key_k1", "name": "Grapple", "color": "#5aa9f0", "x": 9, "y": 2 }
  ],

  "detail": {
    "room_y1": {
      "sub": 8,                   // sub-cells per world cell
      "boxes": [
        { "id": "box_b1", "x": 24, "y": 32, "w": 8, "h": 2,
          "type": "platform", "label": "" }
      ]
    }
  }
}
```

### Conventions

**Cells** are `"x,y"` strings in world grid coordinates. Both axes are unbounded and may go negative.

**Door edges** are identified by an orientation plus a cell coordinate:

- `"v"` at `(x, y)` is the vertical edge dividing cells `(x-1, y)` and `(x, y)`
- `"h"` at `(x, y)` is the horizontal edge dividing cells `(x, y-1)` and `(x, y)`

**Grey-box boxes** use absolute sub-cell coordinates, i.e. `worldCell * sub`. They aren't relative to the room's bounding box, so repainting the room's cells never shifts its interior geometry. Changing `sub` rescales all boxes to keep them in place.

**Import** always assigns a fresh `id`, so importing a file can never overwrite a map already in your library.

### Browser storage

| | |
|---|---|
| IndexedDB | database `mvmap`, object store `maps`, keyed on `id` |
| localStorage fallback | `mvmap:docs` — all documents, keyed by id |
| localStorage | `mvmap:last` — id of the last-opened map, restored on load |

---

## Repo

```
index.html   the entire editor
README.md
```

---

## Notes

Tested in Chromium-based browsers. Requires Canvas 2D, IndexedDB or localStorage, and pointer events — nothing exotic, but there's no polyfill layer, so a very old browser will simply fail rather than degrade.

The whole thing is about 3,100 lines of plain HTML, CSS, and JavaScript in one file, organised into 18 numbered sections. `Ctrl+F` for `═══` to jump between them.
