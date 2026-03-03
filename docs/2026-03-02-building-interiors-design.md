# Building Interiors — Design Document

**Date:** 2026-03-02
**Status:** Approved

---

## Overview

Replace solid one-piece `BoxGeometry` buildings with hollow 5-panel structures (4 walls + floor) that the player can physically enter. Each building gets a door gap in the front wall, an interior ambient light, and one chest placed inside. All changes are in `addBuilding()` and the chest spawning logic within `index.html`.

---

## Section 1: Building Structure

### Wall Panels

Each building is constructed from individual `BoxGeometry` slabs instead of one solid box. All panels share the same `concreteTex` `MeshStandardMaterial`. Interior-facing surfaces use a slightly lighter color tint (`lightness + 0.08`) so the inside feels different from the outside.

| Part | Geometry | World position (relative to building center) |
|------|----------|----------------------------------------------|
| Left wall | `Box(0.3, h, d)` | `x - w/2` |
| Right wall | `Box(0.3, h, d)` | `x + w/2` |
| Back wall | `Box(w, h, 0.3)` | `z - d/2` (or `+ d/2` depending on door side) |
| Front wall — top beam | `Box(w, h - doorH, 0.3)` | above door, `y + doorH/2 + (h-doorH)/2` |
| Front wall — left pier | `Box(doorSideW, doorH, 0.3)` | left of gap |
| Front wall — right pier | `Box(doorSideW, doorH, 0.3)` | right of gap |
| Floor | `Box(w - 0.6, 0.2, d - 0.6)` | `y + 0.1` (slightly above terrain) |

Door dimensions: **1.4 units wide × 2.4 units tall**.
`doorSideW = (w - 1.4) / 2`

Door side (front vs back): chosen by `seededRand(seed+7)` — if > 0.5, door faces +Z, otherwise -Z. This means buildings in a cluster face different directions.

### Collision

Each wall panel (left, right, back, front-top, front-left pier, front-right pier) is registered in `envObjects` as a thin AABB slab. The door gap has no `envObject` entry — player and bots walk through freely.

The floor and roof do NOT get collision entries (player already snaps to terrain Y; roof is out of reach).

---

## Section 2: Interior Chests & Lighting

### Interior Ambient Light

Each building gets one `PointLight` (color `0xfff5e0`, intensity `0.5`, range = `w * 1.2`) placed at the center of the building at `y + h * 0.6`. Always on. Gives the interior a warm dim glow distinct from the outdoor sunlight.

### Interior Chest

Each building spawns exactly one chest inside using the same mesh construction as outdoor chests (gold body, dark lid, gold lock, glow `PointLight`). Placed against one interior wall (seeded):

- **Chest X/Z**: offset 0.8 units from the back wall, centered on that wall's width
- **Chest Y**: `floorY + 0.4`
- Registered in the global `chests[]` array — same `openChest()` pickup logic

The 30 existing outdoor chests are unchanged.

---

## Out of Scope

- Multiple rooms / interior dividing walls
- Doors that open/close (animated)
- Windows
- Staircases or multi-floor buildings
- Bots pathfinding through doorways (they already navigate around buildings; they may now occasionally walk in)
