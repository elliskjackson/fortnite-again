# Game Features Design — 2026-03-12

## Overview

Add sounds, weapon feel, visual polish, new content, skydive intro, and performance improvements to Battle Royale FPS. Implemented in 5 batches, each committed separately.

---

## Batch 1 — Sound + Weapon Feel

### Web Audio API Engine
- Context created on first user gesture (browser autoplay policy)
- Procedural sounds — no external files needed:
  - **Gunshots**: noise burst + bandpass filter, tuned per weapon (pistol = mid, rifle = sharp crack, shotgun = low boom, sniper = high crack, SMG = rapid tick)
  - **Footsteps**: tick every ~0.4s while player is moving on ground
  - **Chest open**: rising tone sweep (220Hz → 880Hz over 0.3s)
  - **Storm hum**: looping oscillator + LFO, volume scales with storm proximity
  - **Grenade explosion**: low rumble burst (50–200Hz noise)
  - **Death whoosh**: descending pitch sweep on player death
  - **Medkit use**: soft rising chime

### Recoil
- Each weapon gets `recoil` (pitch kick in radians) and `recoilRecovery` (radians/sec back)
- On fire: `player.pitch -= weapon.recoil`
- Each frame: pitch recovers toward pre-shot angle at `recoilRecovery` rate

### Spread
- Bullet ray direction jittered within a cone
- Cone half-angle = `weapon.baseSpread + movementSpread + sustainedFireSpread`
- Resets toward base spread when not firing

### Muzzle Flash
- `PointLight` at gun barrel tip, intensity 3, range 4, color 0xffaa44
- Visible for 2 frames then off

### Screen Shake
- `cameraShake` vector added to camera position each frame, decays exponentially
- Triggered by: taking damage (intensity 0.15), grenade explosion nearby (0.3), hard landing (0.1)

### Damage Numbers
- Pool of 20 `CanvasTexture` sprites
- On hit: spawn at target position, float upward 1.5 units over 0.8s, fade out
- Color: yellow for player hits on bots, red for bot hits on player

---

## Batch 2 — Visual Polish

### Expanding Crosshair
- Replace static CSS crosshair with Canvas2D drawn crosshair
- 4 line segments; gap = `baseCrosshair + speedContrib + recoilContrib`
- Shrinks back to base at rest

### Kill Feed
- Top-right DOM div, max 4 entries
- Each entry: "You eliminated Bot 12" with weapon icon emoji
- Fades out after 4s using CSS transition

### Kill Streak
- Centered pop-up text overlay:
  - 2 kills: "DOUBLE KILL"
  - 3 kills: "TRIPLE KILL"
  - 5+ kills: "RAMPAGE"
- 72px bold text, fades after 2s

### Bloom Re-enabled
- `bloomPass.enabled = true`
- Strength 0.3, threshold 0.85 (only very bright emissives bloom — gun barrel, chest glow, muzzle flash)

---

## Batch 3 — Content

### Grenades
- New weapon slot, thrown with `G` key
- Spawns in chests with 30% probability alongside the weapon
- 3-second cook timer shown as HUD countdown while holding G
- On release: projectile arc (parabolic, affected by gravity)
- On land: 0.5s delay then explode
- Explosion: AoE radius 5 — damages bots/player with falloff, destroys placed building pieces in range
- Visual: small sphere mesh, explosion particle burst on detonation

### Medkits
- Spawns in chests with 40% probability
- Press `F` to use (only on ground, not while in air)
- 2-second use time — HUD progress bar while using
- Heals 50HP (capped at 100)
- Interrupted if player moves or takes damage

### Bot Difficulty Tiers
| Levels | Speed | Accuracy | HP | Aggro Radius |
|--------|-------|----------|----|-------------|
| 1–5    | 3.5   | 0.35     | 100 | 40 |
| 6–15   | 4.5   | 0.55     | 100 | 55 |
| 16–30  | 5.5   | 0.70     | 125 | 65 |
| 31+    | 6.5   | 0.85     | 150 | 80 |

---

## Batch 4 — Skydive Intro

### Flow
1. `state.phase = 'skydive'` replaces immediate spawn
2. Camera spawns at Y=250, slight forward velocity
3. Player falls with gravity (faster than normal fall)
4. HUD shows "PRESS SPACE TO DEPLOY PARACHUTE"
5. First Space: deploy parachute — fall slows dramatically, parachute mesh appears above camera
6. Second Space (or auto at Y=5): land, screen shake + thud sound, transition to `'playing'`

### Visuals
- Simple inverted cone mesh above camera as parachute
- Parachute lines: 4 thin cylinders from cone edges to camera

---

## Batch 5 — Performance

### Geometry Merging (Static Scene Objects)
- After all trees/rocks generated, merge per material group using `THREE.BufferGeometryUtils.mergeBufferGeometries()`
- Tree trunks: all merged into 1 mesh
- Tree canopy tier 1 (inner): all merged into 1 mesh
- Tree canopy tiers 2–4: merged into 1 mesh each
- Rocks: all merged into 1 mesh
- Replaces ~1,200 individual meshes with ~6 merged meshes
- Note: merged meshes lose individual frustum culling — acceptable trade-off given draw call reduction

---

## Skipped
- Multi-floor buildings with stairs — requires bot pathfinding rewrite, out of scope
