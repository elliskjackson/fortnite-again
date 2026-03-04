# Building & Editing — Design Document

**Date:** 2026-03-03
**Status:** Approved

---

## Overview

Add Fortnite-style player building and editing: place walls, floors, ramps, and cones in three materials (wood/stone/metal), edit placed pieces using preset cut patterns, destroy pieces with bullets, a settings menu with rebindable keys and sensitivity, and loot drops from killed bots.

---

## Section 1: Building System

### Grid

The world uses a **4-unit grid**. All pieces are 4×4 units and snap to grid cell faces. A semi-transparent ghost piece follows the crosshair, snapping to the nearest valid grid position. Left-click places the piece.

### Piece Types

| Piece | Geometry | Scroll Order |
|-------|----------|-------------|
| Wall | 4W × 4H × 0.3D vertical panel | 1 |
| Floor | 4W × 0.3H × 4D flat slab | 2 |
| Ramp | 4W × 4H × 4D wedge (slopes up toward player) | 3 |
| Cone/Roof | 4W × 4H × 4D pyramid | 4 |

Scroll wheel cycles piece type. Dedicated keybinds (default Z/X/V/F) jump directly to each type. Right-click rotates the ghost 90° before placing.

### Materials

| Material | Color | HP | Roughness | Metalness |
|----------|-------|----|-----------|-----------|
| Wood | 0x8B5E3C (warm brown) | 150 | 0.90 | 0.0 |
| Stone | 0x888888 (grey) | 300 | 0.85 | 0.05 |
| Metal | 0x7a8a9a (blue-grey) | 450 | 0.40 | 0.6 |

Number keys 1/2/3 select material while in build mode.

### Placement Rules

- Pieces cannot overlap existing placed pieces or world buildings.
- Raycasting from camera finds nearest grid cell face; ghost snaps there.
- Placed pieces are added to `envObjects` for player/bot collision.
- Placed pieces are added to a `placedPieces[]` array for damage tracking.

---

## Section 2: Edit Mode

Press **G** while aiming at a placed piece to enter edit mode. The piece highlights and shows its edit grid. Scroll wheel cycles through valid presets for that piece type. Press **G** again to confirm and apply the cut. Press **Esc** to cancel.

Removed panels are gone permanently — you can apply further edits to remove more panels but cannot restore removed ones. Edited pieces still take bullet damage and can be fully destroyed.

### Wall Edit Presets (3×3 grid)

The 3×3 grid is labeled:

```
[TL][TC][TR]
[ML][MC][MR]
[BL][BC][BR]
```

| Preset | Panels Removed |
|--------|---------------|
| Door left | BL + ML |
| Door center | BC + MC |
| Door right | BR + MR |
| Window left | ML |
| Window center | MC |
| Window right | MR |
| L-opening left | BL + BC + ML |
| L-opening right | BC + BR + MR |
| Arch left | BL + BC + ML + MC |
| Arch right | BC + BR + MC + MR |

### Floor Edit Presets (2×2 diagonal quadrants)

Four triangular quadrants viewed from above: top-left (TL), top-right (TR), bottom-left (BL), bottom-right (BR). Toggle any combination to create diagonal cut-outs or L-shaped openings. Presets: remove TL, remove TR, remove BL, remove BR, remove TL+BL, remove TR+BR, remove TL+TR, remove BL+BR.

### Ramp Edit Presets (2×2 front-facing grid)

```
[TL ramp][TR ramp]   ← angled ramp sections
[BL floor][BR floor] ← flat floor sections
```

Toggle any combination of the 4 panels. Common results: half-ramp (left or right), ramp+floor hybrid, stair step (diagonal remove).

### Cone Edit Presets (4 triangular panels from top)

Four triangles meeting at the peak: front-left (FL), front-right (FR), back-left (BL), back-right (BR). Toggle any combination. Common results: U-shape (remove FL+FR), L-shape (remove FL+BL), single triangle, open corner.

---

## Section 3: Destruction

Each entry in `placedPieces[]` stores `{ mesh, hp, maxHp, type, material, envObj }`. Bullet raycasts check placed pieces (via `ray.intersectObjects`) before checking bots. On hit, `hp` decreases by bullet damage. Visual feedback: color tint darkens linearly from full color at 100% HP to 40% brightness at 0% HP. At 0 HP the mesh is removed from the scene and its `envObj` is spliced from `envObjects`.

---

## Section 4: Settings Menu

**Trigger:** Esc during gameplay opens a full-screen overlay, releases pointer lock.

### Tabs

**Controls**

| Action | Default Key |
|--------|------------|
| Move Forward | W |
| Move Back | S |
| Move Left | A |
| Move Right | D |
| Jump | Space |
| Crouch | C |
| Build Mode | Q |
| Edit Mode | G |
| Reload | R |
| Place Wall | Z |
| Place Floor | X |
| Place Ramp | V |
| Place Cone | F |

Each row has a button showing the current key. Click it → button reads "Press any key…" → next keydown binds that action. Conflicting binds highlighted red.

**Mouse**
- Look Sensitivity: slider 0.1× – 3.0×, default 1.0×

**Graphics**
- Bloom Intensity: slider 0 – 2.0, default 1.0

All settings saved to `localStorage`, loaded on page start.

---

## Section 5: Loot Drops

On bot death, the bot's current weapon spawns as a world pickup at their position. The pickup is a small box mesh (color-coded by weapon type) that bobs and rotates. A dim `PointLight` (range 4) glows beneath it.

Walking within 1.5 units auto-collects the pickup. Collection calls `giveWeapon(weaponType)`. If the player already has that weapon, ammo is topped up to max instead.

Pickups despawn after 60 seconds if uncollected. Stored in a `worldWeapons[]` array, updated each game loop tick.

---

## Out of Scope

- Building health regeneration over time
- Trap pieces
- Bot AI using placed structures for cover
- Material harvesting (axe)
- Building piece stacking limits
