# Bot Visual Upgrade — Design Document

**Date:** 2026-02-26
**Status:** Approved

---

## Overview

Replace the current flat blue `BoxGeometry` block figures with a proper articulated humanoid character: box torso/head/hips with cylinder limbs, randomized outfit colors, a visible gun in the right hand, and full animation (arm swing, leg swing, head turn toward player, crouch in attack state). All changes remain within `index.html`. No external assets.

---

## Section 1: Character Shape & Materials

All parts use `MeshStandardMaterial` (replacing the current `MeshLambertMaterial`) so they respond correctly to the scene's directional light and the bloom post-processing pipeline.

### Geometry

| Part | Geometry | Y offset from group root |
|------|----------|--------------------------|
| Hips | `BoxGeometry(0.38, 0.22, 0.25)` | 0.91 |
| Torso | `BoxGeometry(0.55, 0.55, 0.28)` | 1.30 |
| Neck | `CylinderGeometry(0.1, 0.1, 0.15, 8)` | 1.65 |
| Head | `BoxGeometry(0.38, 0.38, 0.38)` | 1.90 |
| Left / Right upper arm | `CylinderGeometry(0.09, 0.09, 0.35, 8)` | pivot at shoulder |
| Left / Right forearm | `CylinderGeometry(0.075, 0.075, 0.30, 8)` | below upper arm |
| Left / Right thigh | `CylinderGeometry(0.11, 0.10, 0.38, 8)` | pivot at hip |
| Left / Right shin | `CylinderGeometry(0.09, 0.08, 0.36, 8)` | below thigh |
| Gun | `BoxGeometry(0.08, 0.08, 0.35)` | attached to right forearm |

### Materials

- **Outfit color** (torso, hips, upper arms, thighs): `MeshStandardMaterial` with a randomly chosen hue per bot, roughness 0.8, metalness 0.1
- **Skin** (head, neck, forearms): `MeshStandardMaterial`, color `0xffdead`, roughness 0.85, metalness 0.0
- **Boots** (shins): `MeshStandardMaterial`, dark color `0x222222`, roughness 0.9, metalness 0.1
- **Gun**: `MeshStandardMaterial`, color `0x333333`, roughness 0.5, metalness 0.6
- **Hit flash**: temporary color override to `0xff8800` on all outfit parts, then restore

Outfit hue is seeded from the bot index so bots always spawn with the same color on game restart.

---

## Section 2: Pivot Hierarchy & Animation

Named parts are stored on `bot.parts` for direct reference (no fragile child index lookups).

### Hierarchy

```
group  (bot.mesh)
  ├── hips
  ├── torso
  │    ├── neck
  │    └── head          ← rotates Y toward player (lerp)
  ├── leftArmPivot       ← rotates X for arm swing / aim raise
  │    └── leftUpperArm
  │         └── leftForearmPivot
  │              └── leftForearm
  ├── rightArmPivot      ← rotates X for arm swing / aim raise
  │    └── rightUpperArm
  │         └── rightForearmPivot
  │              └── rightForearm
  │                   └── gun
  ├── leftLegPivot       ← rotates X for leg swing
  │    └── leftThigh
  │         └── leftShinPivot
  │              └── leftShin
  └── rightLegPivot      ← rotates X for leg swing
       └── rightThigh
            └── rightShinPivot
                 └── rightShin
```

### Animations

| State | Legs | Arms | Head | Body Y |
|-------|------|------|------|--------|
| Walking / chasing | Swing ±0.55 rad opposite phase | Swing ±0.35 rad opposite to legs | Faces forward | Normal |
| Idle / patrol | Gentle sway ±0.1 rad | Gentle sway ±0.06 rad | Faces forward | Normal |
| Attack (shooting) | Stop swinging | Both arm pivots raise to –0.9 rad (aiming) | Turns toward player | Drops 0.3 units (crouch) |
| Flee | Swing ±0.65 rad (faster) | Swing ±0.40 rad | Faces away from player | Normal |
| Death | All parts droop (X rotation +0.3) | Sink into ground (existing behavior) | — | — |

Animation uses `bot.walkPhase` (already exists) for walk cycle. All transitions are immediate except head turn (lerp factor 5× dt).

---

## Out of Scope

- Facial features / eyes
- Cloth/hair simulation
- Per-limb hit detection
- Shadow casting on cylinder limbs (castShadow already set to true globally)
