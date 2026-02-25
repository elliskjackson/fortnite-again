# Fortnite-Style FPS Battle Royale — Design Document

**Date:** 2026-02-23
**Status:** Approved

---

## Overview

A browser-based first-person shooter battle royale game built as a single `index.html` file using Three.js (loaded via CDN). The player competes against 99 bots on a large 3D map with a shrinking storm zone. The game has 100 campaign levels (bots get progressively harder) followed by an endless mode. Weapons are found in chests. No blood — clean, family-friendly combat.

---

## Architecture

### Tech Stack

- **Platform:** Browser — single `index.html` file, no install required
- **Renderer:** Three.js via CDN `<script>` tag (WebGL)
- **No build step** — open file directly in browser

### Rendering

- PBR (physically-based) materials for realistic surfaces (metal, wood, concrete)
- Dynamic directional sunlight (sun at angle) + ambient hemisphere light
- Point lights on chests for atmosphere
- Shadow maps enabled
- Fog for atmosphere and hiding distant pop-in
- Tone mapping (ACESFilmic) + slight vignette post-processing effect

### Game Loop

- `requestAnimationFrame` main loop
- Fixed 60 fps logic timestep, variable render rate
- Delta-time clamped to prevent spiral of death on slow frames

---

## Map

- **Size:** 500 × 500 units (open world)
- **Terrain:** `PlaneGeometry` with vertex height displacement — flat near center, rolling edges
- **Environment objects** (procedurally placed at game start):
  - Buildings (box geometry clusters, realistic proportions)
  - Trees (cylinder trunk + cone/sphere canopy)
  - Rocks (icosahedron geometry, scaled randomly)
- **Chests:** ~30 fixed spawn points, each a glowing golden box with point light and shimmer particles
- **Storm:** Shrinking circle boundary. Outside the storm → 5 HP damage per second. Storm shrinks every 90 seconds (configurable). Storm visualized as a purple/blue glowing wall.

---

## Player Controls

- **Movement:** WASD
- **Jump:** Space
- **Aim:** Mouse (Pointer Lock API — click to capture)
- **Shoot:** Left mouse button
- **Open chest:** E (within 3 units of chest)
- **Crouch:** C (reduces movement speed, improves accuracy)
- **Zoom (sniper):** Right mouse button (when sniper equipped)

---

## Weapons

All weapons are found in chests. Player starts with a Pistol.

| Weapon | Fire Rate | Damage | Range | Ammo | Notes |
|---|---|---|---|---|---|
| Pistol | Slow | 25 | Long | 12 | Starter weapon |
| Assault Rifle | Medium | 20 | Long | 30 | Balanced |
| Shotgun | Slow | 80 (close) / 10 (far) | Short | 8 | Falloff damage |
| Sniper Rifle | Very slow | 95 | Very long | 5 | Zoom on RMB |
| SMG | Very fast | 12 | Medium | 45 | Hip fire spread |

- **Combat system:** Hitscan — raycasting from camera center on fire
- **Ammo:** Limited, shown in HUD. Picking up duplicate weapon adds ammo.
- **Hit feedback:** Bots flash white/orange on hit (no blood)
- **Bot death:** Bot plays "fall" animation (tilts, collapses), then fades out after 2 seconds

---

## Bot AI

Each bot runs a state machine:

```
LOOT     → wander toward nearest unopened chest
PATROL   → random walk when no chest target
CHASE    → enemy spotted within aggression radius → move toward target
ATTACK   → within attack range (≤15 units) → stop and shoot
FLEE     → HP < 20 → move away from closest threat
```

**Line-of-sight:** Raycasting check before transitioning to CHASE/ATTACK.

**Performance culling:**
- Bots within 150 units of player: **full AI simulation**
- Bots 150–300 units away: **ghost mode** — random walk + occasional position update to simulate distant battle activity
- Bots >300 units: **frozen** until storm forces them closer

**Bot-vs-bot:** Bots also shoot each other — this naturally reduces the bot count over time even without player involvement (realistic battle royale attrition).

---

## Level System

### Campaign (Levels 1–100)

Each level is a full battle royale match vs 99 bots. Win = last player alive. Lose = player dies → retry same level.

Level progress is saved to `localStorage`.

| Level | Accuracy | Speed Multiplier | Aggression Radius |
|---|---|---|---|
| 1 | 5% | 0.5× | 20 units |
| 25 | 25% | 0.7× | 40 units |
| 50 | 50% | 1.0× | 60 units |
| 75 | 75% | 1.2× | 80 units |
| 100 | 95% | 1.5× | 100 units |

Scaling is linear interpolation between these anchor points.

### Endless Mode (Level 101+)

Unlocked after beating level 100. Each endless match:
- Bot accuracy: `95% + (endlessLevel × 0.2%)` (capped at 99%)
- Bot speed: `1.5× + (endlessLevel × 0.02×)` (capped at 2.5×)
- Endless level number displayed prominently on screen

---

## HUD

Rendered as a 2D overlay on top of the 3D canvas:

- **Health bar** — bottom left, red bar + HP number
- **Ammo counter** — bottom right, current / max
- **Weapon icon** — bottom right, above ammo
- **Minimap** — top right, 2D overhead view showing: player dot (white), bot dots (red), storm ring, chest markers (yellow)
- **Kills counter** — top left
- **Storm warning** — flashing red border when player is outside the storm
- **Level / Endless indicator** — top center

---

## Screens

- **Main Menu:** Level select grid (1–100 + Endless), best score per level
- **In-game:** HUD overlay described above
- **Death screen:** "You died" overlay with retry / main menu buttons
- **Win screen:** "Victory!" overlay with next level / main menu buttons

---

## File Structure

```
Desktop/fortnite-game/
└── index.html    ← entire game, single file
```

---

## Out of Scope

- Multiplayer (single player only — bots simulate a battle royale)
- Sound effects (no audio implementation)
- Building mechanics (Fortnite's build mechanic not included)
- Mobile support (desktop browser only)
- Blood or gore (hit feedback is flash-only)
