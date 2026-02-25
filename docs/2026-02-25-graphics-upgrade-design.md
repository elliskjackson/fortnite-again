# Graphics Upgrade — Design Document

**Date:** 2026-02-25
**Status:** Approved

---

## Overview

A full visual upgrade to the existing browser-based battle royale FPS. All changes remain within a single `index.html` file. Three additional Three.js addon scripts are loaded from CDN for post-processing. No external image assets — textures are generated procedurally at runtime via canvas.

---

## Section 1: Sky & Atmosphere

- **Sky shader:** Load `Sky.js` from Three.js r128 CDN. Physically-based atmospheric scattering with configurable sun position, turbidity, and Rayleigh scattering.
- **Sun linkage:** Sky sun position is kept in sync with the existing `DirectionalLight` so shadows always match the sky.
- **Fog:** Replace linear `THREE.Fog` with `THREE.FogExp2` (exponential). Fog color dynamically matches the sky horizon color.
- **Storm atmosphere:** When player is inside the storm or storm is active, gradually shift fog/sky toward dark purple-grey tint.

---

## Section 2: Bloom Post-processing & Screen Effects

- **Pipeline:** `EffectComposer` → `RenderPass` → `UnrealBloomPass` → render to screen. Load `EffectComposer.js`, `RenderPass.js`, `UnrealBloomPass.js`, `ShaderPass.js`, `LuminosityHighPassShader.js`, `CopyShader.js` from CDN.
- **Bloom tuning:** High threshold (0.8) and moderate strength (0.6) so only genuinely bright emissive objects glow — chest point lights, muzzle flash mesh, storm wall material.
- **Vignette:** CSS `box-shadow: inset` on a fixed overlay div — subtle dark corners always visible.
- **Damage flash:** On `damagePlayer()`, toggle a CSS class on a fixed red-border overlay div for 300ms.
- **Sniper zoom:** On RMB mousedown (when sniper equipped), narrow camera FOV from 75° to 25° and show black sidebar bars (CSS). Restore on RMB mouseup.

---

## Section 3: Procedural Textures

All textures generated once at startup via `document.createElement('canvas')` and `THREE.CanvasTexture`.

| Surface | Technique | Material properties |
|---|---|---|
| Terrain (ground) | Green base + darker noise blotches + subtle grid | roughness 0.9, metalness 0.0 |
| Buildings | Grey base + fine panel grid lines + edge variation | roughness 0.7, metalness 0.1 |
| Tree trunks | Brown base + vertical dark streaks | roughness 0.95, metalness 0.0 |
| Tree canopy | Green base + darker leaf clumps | roughness 0.85, metalness 0.0 |

Textures are wrapped (`RepeatWrapping`) and tiled to avoid obvious repetition.

---

## Section 4: Particles & Environmental Detail

- **Clouds:** 12 large flat `CircleGeometry` planes (radius 60–120, y = 180–220) with translucent white `MeshBasicMaterial`. Each drifts slowly on a random XZ heading, looping when it exits the map boundary.
- **Ground dust:** A `THREE.Points` system of ~300 particles within a 30-unit radius of the player, drifting upward slowly and fading. Gives ambient haze at ground level.
- **Storm lightning:** When `storm.shrinkCount` increments, spawn a white `PointLight` (intensity 8, range 200) at a random position on the storm perimeter. Fade it out over 0.3s.
- **Impact sparks:** Increase spark count from current to 8 per hit. Color interpolates from white-hot (0xffffff) at spawn to orange (0xff6600) as life fades. Slightly larger geometry (0.08 radius vs current 0.05).

---

## CDN Scripts Required

In addition to the existing `three.min.js` r128, add:

```html
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/postprocessing/EffectComposer.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/postprocessing/RenderPass.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/postprocessing/ShaderPass.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/shaders/LuminosityHighPassShader.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/postprocessing/UnrealBloomPass.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/shaders/CopyShader.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/objects/Sky.js"></script>
```

---

## Out of Scope

- Normal/bump maps (requires texture atlasing complexity)
- Water reflections
- Dynamic time-of-day cycle (sun stays fixed)
- Shadow cascades (performance concern in browser)
- Audio
