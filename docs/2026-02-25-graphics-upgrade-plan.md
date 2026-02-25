# Graphics Upgrade — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Full visual upgrade — bloom post-processing, sky enhancement, procedural textures, clouds, dust, storm lightning, and sniper zoom — all within the single `index.html` file.

**Architecture:** All changes are made to `~/Desktop/fortnite-game/index.html`. Post-processing is added via six Three.js r128 CDN addon scripts. Textures are generated procedurally at startup using `<canvas>` elements so no external image files are needed. The render loop switches from `renderer.render()` to `composer.render()`.

**Tech Stack:** HTML5, Three.js r128 (CDN), `EffectComposer` + `UnrealBloomPass` (CDN addons), Canvas 2D API for procedural textures.

---

## Task 1: CSS additions + new overlay divs

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html` — inside `<style>` block and `<body>`

**Step 1: Add CSS for vignette, damage flash, and sniper scope**

Inside the `<style>` block (after the `#crosshair::after` rule, before `</style>`), add:

```css
    #vignette {
      position: fixed; top: 0; left: 0;
      width: 100%; height: 100%;
      pointer-events: none;
      box-shadow: inset 0 0 120px rgba(0,0,0,0.55);
      z-index: 2;
    }
    #dmgFlash {
      position: fixed; top: 0; left: 0;
      width: 100%; height: 100%;
      pointer-events: none;
      box-shadow: inset 0 0 80px rgba(255,0,0,0.0);
      transition: box-shadow 0.05s ease-in;
      z-index: 3;
    }
    #dmgFlash.hit {
      box-shadow: inset 0 0 80px rgba(255,0,0,0.65);
      transition: none;
    }
    #scopeBars {
      position: fixed; top: 0; left: 0;
      width: 100%; height: 100%;
      pointer-events: none;
      display: none;
      z-index: 4;
    }
    #scopeBars::before, #scopeBars::after {
      content: '';
      position: absolute;
      top: 0; bottom: 0;
      width: 30%;
      background: rgba(0,0,0,0.92);
    }
    #scopeBars::before { left: 0; }
    #scopeBars::after  { right: 0; }
```

**Step 2: Add the new overlay divs to `<body>`**

After `<div id="crosshair"></div>` and before the `<!-- MAIN MENU OVERLAY -->` comment, add:

```html
<div id="vignette"></div>
<div id="dmgFlash"></div>
<div id="scopeBars"></div>
```

**Step 3: Verify in browser**

Open `index.html` in Chrome. Start the game. You should see dark corners (vignette) on the screen. No red flash yet (that comes in Task 9).

---

## Task 2: Add CDN scripts for post-processing

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html` — the `<script>` tags near bottom of `<body>`

**Step 1: Add post-processing CDN scripts**

Find the existing line:
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
```

Replace it with (keeping the same three.min.js URL, adding the addon scripts after):

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/shaders/CopyShader.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/shaders/LuminosityHighPassShader.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/postprocessing/EffectComposer.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/postprocessing/RenderPass.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/postprocessing/ShaderPass.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/postprocessing/UnrealBloomPass.js"></script>
```

**Step 2: Verify scripts load**

Open `index.html` in Chrome. Open DevTools → Console. There should be zero errors about undefined classes. If you see `THREE.EffectComposer is not a constructor`, the CDN path is wrong — check the jsdelivr URL.

---

## Task 3: EffectComposer setup + bloom + resize + render loop

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html`

**Step 1: Create the EffectComposer after the renderer setup**

Find the `resize()` function definition (around line 148). Just BEFORE it, add:

```javascript
// ════════════════════════════════════════════════════════════════
//  POST-PROCESSING
// ════════════════════════════════════════════════════════════════
const composer = new THREE.EffectComposer(renderer);
const renderPass = new THREE.RenderPass(scene, camera);
composer.addPass(renderPass);

const bloomPass = new THREE.UnrealBloomPass(
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  0.55,   // strength  — lower = subtler
  0.35,   // radius
  0.82    // threshold — only pixels brighter than this glow
);
composer.addPass(bloomPass);
```

**Step 2: Update `resize()` to also resize the composer**

Find:
```javascript
function resize() {
  const W = window.innerWidth, H = window.innerHeight;
  renderer.setSize(W, H);
  hud.width  = W;
```

Replace with:
```javascript
function resize() {
  const W = window.innerWidth, H = window.innerHeight;
  renderer.setSize(W, H);
  composer.setSize(W, H);
  hud.width  = W;
```

**Step 3: Swap render call in game loop**

Find (around line 808):
```javascript
  renderer.render(scene, camera);
```

Replace with:
```javascript
  composer.render();
```

**Step 4: Verify in browser**

Open game, start playing. Scene should still render correctly. If the sky looks washed out, adjust `renderer.toneMappingExposure` from `1.1` to `0.9`. The bloom isn't visible yet (emissive objects come in Task 4).

---

## Task 4: Emissive upgrades for bloom visibility

Make chests, storm wall, and muzzle flash bright enough to trigger bloom.

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html`

**Step 1: Upgrade chest gold material to higher emissive intensity**

Find:
```javascript
  const goldMat  = new THREE.MeshLambertMaterial({ color: 0xf0c040, emissive: 0x604800, emissiveIntensity: 0.4 });
```

Replace with:
```javascript
  const goldMat  = new THREE.MeshStandardMaterial({ color: 0xf0c040, emissive: 0xf0c040, emissiveIntensity: 0.6, roughness: 0.4, metalness: 0.8 });
```

**Step 2: Upgrade storm wall to emissive material**

Find:
```javascript
const stormWallMat = new THREE.MeshBasicMaterial({
  color: 0x4400ff, transparent: true, opacity: 0.18,
  side: THREE.BackSide
});
```

Replace with:
```javascript
const stormWallMat = new THREE.MeshStandardMaterial({
  color: 0x6600ff,
  emissive: 0x4400cc,
  emissiveIntensity: 1.2,
  transparent: true,
  opacity: 0.22,
  side: THREE.BackSide,
  roughness: 1.0,
  metalness: 0.0
});
```

**Step 3: Make muzzle flash brighter**

Find:
```javascript
const flashMat = new THREE.MeshBasicMaterial({ color: 0xffff80, transparent: true, opacity: 0 });
const flashMesh = new THREE.Mesh(new THREE.SphereGeometry(0.12, 6, 6), flashMat);
```

Replace with:
```javascript
const flashMat = new THREE.MeshStandardMaterial({ color: 0xffffff, emissive: 0xffff00, emissiveIntensity: 3.0, transparent: true, opacity: 0 });
const flashMesh = new THREE.Mesh(new THREE.SphereGeometry(0.18, 6, 6), flashMat);
```

**Step 4: Boost open-chest light intensity**

In `openChest()`, find:
```javascript
  chest.light.color.set(0xffffff);
  chest.light.intensity = 3;
```

Replace with:
```javascript
  chest.light.color.set(0xffffff);
  chest.light.intensity = 6;
```

**Step 5: Verify bloom in browser**

Start game, walk to a chest (yellow glow should bloom). Shoot (muzzle flash should bloom). Walk to storm edge (purple wall should glow). If bloom is too strong, reduce `bloomPass.strength` in Task 3 Step 1.

---

## Task 5: Procedural terrain texture

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html`

**Step 1: Add a `makeGrassTexture()` function**

Just before the `// TERRAIN` section comment (around line 177), add:

```javascript
// ════════════════════════════════════════════════════════════════
//  PROCEDURAL TEXTURES
// ════════════════════════════════════════════════════════════════
function makeCanvasTexture(size, drawFn) {
  const c = document.createElement('canvas');
  c.width = c.height = size;
  drawFn(c.getContext('2d'), size);
  const tex = new THREE.CanvasTexture(c);
  tex.wrapS = tex.wrapT = THREE.RepeatWrapping;
  return tex;
}

const grassTex = makeCanvasTexture(512, (ctx, s) => {
  // Base green
  ctx.fillStyle = '#4a7c3f';
  ctx.fillRect(0, 0, s, s);
  // Noise blotches
  for (let i = 0; i < 600; i++) {
    const x = Math.random() * s, y = Math.random() * s;
    const r = 4 + Math.random() * 18;
    const lightness = 30 + Math.random() * 20;
    ctx.fillStyle = `hsl(110,${28 + Math.random()*14}%,${lightness}%)`;
    ctx.beginPath(); ctx.arc(x, y, r, 0, Math.PI*2); ctx.fill();
  }
  // Fine blade lines
  ctx.strokeStyle = 'rgba(0,80,0,0.18)';
  ctx.lineWidth = 1;
  for (let i = 0; i < 300; i++) {
    const x = Math.random() * s, y = Math.random() * s;
    ctx.beginPath(); ctx.moveTo(x, y); ctx.lineTo(x + (Math.random()-0.5)*8, y - 6 - Math.random()*10); ctx.stroke();
  }
});
grassTex.repeat.set(40, 40);

const concreteTex = makeCanvasTexture(512, (ctx, s) => {
  ctx.fillStyle = '#888';
  ctx.fillRect(0, 0, s, s);
  // Panel grid
  ctx.strokeStyle = 'rgba(0,0,0,0.25)';
  ctx.lineWidth = 2;
  for (let x = 0; x < s; x += 64) { ctx.beginPath(); ctx.moveTo(x,0); ctx.lineTo(x,s); ctx.stroke(); }
  for (let y = 0; y < s; y += 64) { ctx.beginPath(); ctx.moveTo(0,y); ctx.lineTo(s,y); ctx.stroke(); }
  // Subtle noise
  for (let i = 0; i < 2000; i++) {
    const x = Math.random()*s, y = Math.random()*s;
    const v = Math.floor(120 + Math.random()*40);
    ctx.fillStyle = `rgba(${v},${v},${v},0.08)`;
    ctx.fillRect(x, y, 3, 3);
  }
});
concreteTex.repeat.set(3, 3);

const barkTex = makeCanvasTexture(256, (ctx, s) => {
  ctx.fillStyle = '#6B3A2A';
  ctx.fillRect(0, 0, s, s);
  // Vertical streaks
  for (let i = 0; i < 60; i++) {
    const x = Math.random() * s;
    const w = 1 + Math.random() * 4;
    const dark = Math.random() > 0.5;
    ctx.fillStyle = dark ? 'rgba(0,0,0,0.2)' : 'rgba(255,200,150,0.12)';
    ctx.fillRect(x, 0, w, s);
  }
});
barkTex.repeat.set(1, 2);

const leavesTex = makeCanvasTexture(256, (ctx, s) => {
  ctx.fillStyle = '#2d6e2d';
  ctx.fillRect(0, 0, s, s);
  for (let i = 0; i < 200; i++) {
    const x = Math.random()*s, y = Math.random()*s;
    const r = 6 + Math.random()*14;
    ctx.fillStyle = `hsl(120,${35+Math.random()*20}%,${22+Math.random()*15}%)`;
    ctx.beginPath(); ctx.arc(x, y, r, 0, Math.PI*2); ctx.fill();
  }
});
leavesTex.repeat.set(2, 2);
```

**Step 2: Apply grass texture to terrain material**

Find:
```javascript
const terrainMat  = new THREE.MeshLambertMaterial({ color: 0x4a7c3f });
```

Replace with:
```javascript
const terrainMat  = new THREE.MeshStandardMaterial({ color: 0x7aaf6a, map: grassTex, roughness: 0.92, metalness: 0.0 });
```

**Step 3: Verify in browser**

Ground should now show a visible grass texture with blotches and blade lines.

---

## Task 6: Procedural building and tree textures

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html` — `addBuilding()` and `addTree()` functions

**Step 1: Upgrade building material to use concrete texture**

In `addBuilding()`, find:
```javascript
  const mat = new THREE.MeshLambertMaterial({
    color: new THREE.Color().setHSL(0.08 + seededRand(seed+3)*0.05, 0.15, 0.45 + seededRand(seed+4)*0.2)
  });
```

Replace with:
```javascript
  const mat = new THREE.MeshStandardMaterial({
    color: new THREE.Color().setHSL(0.08 + seededRand(seed+3)*0.05, 0.12, 0.48 + seededRand(seed+4)*0.15),
    map: concreteTex,
    roughness: 0.85,
    metalness: 0.05
  });
```

Also upgrade the roof material. Find:
```javascript
  const roofMat = new THREE.MeshLambertMaterial({ color: 0x555555 });
```

Replace with:
```javascript
  const roofMat = new THREE.MeshStandardMaterial({ color: 0x444444, roughness: 0.9, metalness: 0.1 });
```

**Step 2: Upgrade tree trunk and leaves materials**

In `addTree()`, find:
```javascript
  const trunkMat = new THREE.MeshLambertMaterial({ color: 0x6B3A2A });
```

Replace with:
```javascript
  const trunkMat = new THREE.MeshStandardMaterial({ color: 0x6B3A2A, map: barkTex, roughness: 0.95, metalness: 0.0 });
```

Find:
```javascript
  const leavesMat = new THREE.MeshLambertMaterial({ color: 0x2d6e2d });
```

Replace with:
```javascript
  const leavesMat = new THREE.MeshStandardMaterial({ color: 0x3a8a3a, map: leavesTex, roughness: 0.88, metalness: 0.0 });
```

**Step 3: Verify in browser**

Buildings should show a concrete panel grid. Trees should show bark texture on trunks and leaf clumps on canopies.

---

## Task 7: Animated clouds + ground dust

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html`

**Step 1: Add cloud planes**

Just after the environment generation IIFE (after the closing `})();` of `generateEnvironment`), add:

```javascript
// ════════════════════════════════════════════════════════════════
//  CLOUDS
// ════════════════════════════════════════════════════════════════
const clouds = [];
(function spawnClouds() {
  const cloudMat = new THREE.MeshBasicMaterial({
    color: 0xffffff, transparent: true, opacity: 0.55,
    depthWrite: false
  });
  for (let i = 0; i < 14; i++) {
    const r  = 55 + Math.random() * 80;
    const geo = new THREE.CircleGeometry(r, 12);
    const mesh = new THREE.Mesh(geo, cloudMat.clone());
    mesh.rotation.x = -Math.PI / 2;
    mesh.position.set(
      (Math.random() - 0.5) * MAP_SIZE * 1.2,
      185 + Math.random() * 40,
      (Math.random() - 0.5) * MAP_SIZE * 1.2
    );
    mesh.material.opacity = 0.3 + Math.random() * 0.35;
    scene.add(mesh);
    clouds.push({ mesh, vx: (Math.random()-0.5)*0.8, vz: (Math.random()-0.5)*0.8 });
  }
})();

function updateClouds(dt) {
  for (const c of clouds) {
    c.mesh.position.x += c.vx * dt;
    c.mesh.position.z += c.vz * dt;
    // Wrap around map edges
    if (c.mesh.position.x >  MAP_SIZE * 0.7) c.mesh.position.x = -MAP_SIZE * 0.7;
    if (c.mesh.position.x < -MAP_SIZE * 0.7) c.mesh.position.x =  MAP_SIZE * 0.7;
    if (c.mesh.position.z >  MAP_SIZE * 0.7) c.mesh.position.z = -MAP_SIZE * 0.7;
    if (c.mesh.position.z < -MAP_SIZE * 0.7) c.mesh.position.z =  MAP_SIZE * 0.7;
  }
}
```

**Step 2: Add ground dust particle system**

Immediately after the cloud code above, add:

```javascript
// ════════════════════════════════════════════════════════════════
//  GROUND DUST
// ════════════════════════════════════════════════════════════════
const DUST_COUNT = 280;
const dustPositions = new Float32Array(DUST_COUNT * 3);
const dustGeo = new THREE.BufferGeometry();
dustGeo.setAttribute('position', new THREE.BufferAttribute(dustPositions, 3));
const dustMat = new THREE.PointsMaterial({
  color: 0xc8b89a, size: 0.18, transparent: true, opacity: 0.28,
  depthWrite: false, sizeAttenuation: true
});
const dustMesh = new THREE.Points(dustGeo, dustMat);
scene.add(dustMesh);
const dustOffsets = Array.from({length: DUST_COUNT}, () => ({
  ox: (Math.random()-0.5)*22,
  oy: Math.random()*3.5,
  oz: (Math.random()-0.5)*22,
  speed: 0.3 + Math.random()*0.5,
  phase: Math.random()*Math.PI*2
}));

function updateDust(dt) {
  const px = camera.position.x, pz = camera.position.z;
  for (let i = 0; i < DUST_COUNT; i++) {
    const d = dustOffsets[i];
    d.phase += d.speed * dt;
    dustPositions[i*3]   = px + d.ox + Math.sin(d.phase * 0.7) * 1.2;
    dustPositions[i*3+1] = d.oy + Math.sin(d.phase) * 0.4;
    dustPositions[i*3+2] = pz + d.oz + Math.cos(d.phase * 0.9) * 1.2;
  }
  dustGeo.attributes.position.needsUpdate = true;
}
```

**Step 3: Call both update functions in the game loop**

Find inside `function loop(t)`:
```javascript
    updateImpacts(dt);
```

Replace with:
```javascript
    updateImpacts(dt);
    updateClouds(dt);
    updateDust(dt);
```

**Step 4: Verify in browser**

Look up — you should see white cloud planes drifting slowly above. Near the ground you should see subtle floating dust specks around the player.

---

## Task 8: Storm lightning + impact spark upgrade

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html`

**Step 1: Add lightning light + trigger it on storm shrink**

After the `stormWall` mesh setup (around line 578, after `scene.add(stormWall);`), add:

```javascript
// Storm lightning
const lightningLight = new THREE.PointLight(0xccccff, 0, 250);
lightningLight.position.set(0, 40, 0);
scene.add(lightningLight);
let lightningTimer = 0;
```

**Step 2: Add `triggerLightning()` function**

Just before the `updateStorm` function, add:

```javascript
function triggerLightning() {
  const angle = Math.random() * Math.PI * 2;
  const r = storm.radius * 0.85;
  lightningLight.position.set(
    storm.cx + Math.cos(angle) * r,
    30 + Math.random() * 30,
    storm.cz + Math.sin(angle) * r
  );
  lightningLight.intensity = 6 + Math.random() * 4;
  lightningTimer = 0.25;
}
```

**Step 3: Update the lightning in `updateStorm()`**

In `updateStorm`, find where the storm shrinks (the `storm.shrinkCount++` line). Add a lightning call right after it:

Find:
```javascript
    storm.shrinkCount++;
```

Replace with:
```javascript
    storm.shrinkCount++;
    triggerLightning();
```

Also call `updateLightning(dt)` — add this function near `triggerLightning`:

```javascript
function updateLightning(dt) {
  if (lightningTimer > 0) {
    lightningTimer -= dt;
    lightningLight.intensity = Math.max(0, lightningLight.intensity - dt * 30);
    if (lightningTimer <= 0) lightningLight.intensity = 0;
  }
}
```

And add the call in the game loop, after `updateDust(dt)`:
```javascript
    updateLightning(dt);
```

**Step 4: Upgrade impact sparks — pool size and color**

Find:
```javascript
const impactParticles = [];
const impactGeo = new THREE.SphereGeometry(0.05, 4, 4);
const impactMat = new THREE.MeshBasicMaterial({ color: 0xffcc44 });
for (let i = 0; i < 20; i++) {
  const m = new THREE.Mesh(impactGeo, impactMat.clone());
  m.visible = false;
  scene.add(m);
  impactParticles.push({ mesh: m, life: 0, vel: new THREE.Vector3() });
}
```

Replace with:
```javascript
const impactParticles = [];
const impactGeo = new THREE.SphereGeometry(0.08, 4, 4);
const impactMat = new THREE.MeshStandardMaterial({ color: 0xffffff, emissive: 0xffaa00, emissiveIntensity: 2.0, transparent: true });
for (let i = 0; i < 40; i++) {
  const m = new THREE.Mesh(impactGeo, impactMat.clone());
  m.visible = false;
  scene.add(m);
  impactParticles.push({ mesh: m, life: 0, vel: new THREE.Vector3() });
}
```

**Step 5: Upgrade `spawnImpact()` to emit more sparks**

Find:
```javascript
function spawnImpact(pos) {
  const p = impactParticles[impactIdx++ % impactParticles.length];
  p.mesh.position.copy(pos);
  p.mesh.visible = true;
  p.life = 0.4;
  p.vel.set((Math.random()-0.5)*4, Math.random()*4, (Math.random()-0.5)*4);
}
```

Replace with:
```javascript
function spawnImpact(pos) {
  for (let s = 0; s < 6; s++) {
    const p = impactParticles[impactIdx++ % impactParticles.length];
    p.mesh.position.copy(pos);
    p.mesh.visible = true;
    p.life = 0.3 + Math.random() * 0.25;
    p.vel.set((Math.random()-0.5)*6, Math.random()*5 + 1, (Math.random()-0.5)*6);
  }
}
```

**Step 6: Update `updateImpacts()` to fade color from white to orange**

Find:
```javascript
    p.mesh.material.opacity = p.life / 0.4;
```

Replace with:
```javascript
    const t = p.life / 0.55;
    p.mesh.material.opacity = t;
    p.mesh.material.emissiveIntensity = t * 2.0;
```

**Step 7: Verify in browser**

Shoot at a surface — you should see a burst of 6 sparks per shot, bright white-hot at spawn and fading orange. When the storm shrinks (wait ~90s or reduce STORM_INTERVAL temporarily for testing), you should see a lightning flash near the storm wall.

---

## Task 9: Sniper zoom (RMB) + damage flash

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html`

**Step 1: Add RMB zoom state variable**

Near the other player state variables (around where `flashTimer` is defined), add:

```javascript
let isZoomed = false;
```

**Step 2: Add RMB mousedown/mouseup handlers**

After the existing `mousedown` shoot handler (after line 652), add:

```javascript
// Sniper zoom (RMB)
document.addEventListener('mousedown', e => {
  if (e.button === 2 && state.phase === 'playing' &&
      document.pointerLockElement === canvas &&
      player.weapon === 'sniper') {
    isZoomed = true;
    camera.fov = 22;
    camera.updateProjectionMatrix();
    document.getElementById('scopeBars').style.display = 'block';
    document.getElementById('crosshair').style.display = 'none';
  }
});
document.addEventListener('mouseup', e => {
  if (e.button === 2) {
    isZoomed = false;
    camera.fov = 75;
    camera.updateProjectionMatrix();
    document.getElementById('scopeBars').style.display = 'none';
    if (document.pointerLockElement === canvas) {
      document.getElementById('crosshair').style.display = 'block';
    }
  }
});
document.addEventListener('contextmenu', e => e.preventDefault());
```

**Step 3: Auto-exit zoom when weapon changes**

In `giveWeapon()`, after `player.weapon = key;` (the new-weapon line), add:

```javascript
    if (isZoomed) {
      isZoomed = false;
      camera.fov = 75;
      camera.updateProjectionMatrix();
      document.getElementById('scopeBars').style.display = 'none';
    }
```

**Step 4: Add damage flash logic**

In `damagePlayer()`, find:
```javascript
function damagePlayer(dmg) {
  player.hp -= dmg;
  player.hitTime = 0.3;
  if (player.hp <= 0) killPlayer();
}
```

Replace with:
```javascript
function damagePlayer(dmg) {
  player.hp -= dmg;
  player.hitTime = 0.3;
  // Red screen flash
  const df = document.getElementById('dmgFlash');
  df.classList.add('hit');
  setTimeout(() => df.classList.remove('hit'), 80);
  if (player.hp <= 0) killPlayer();
}
```

**Step 5: Verify in browser**

- Equip sniper (pick it up from a chest). Hold RMB — black bars appear on sides and FOV narrows dramatically. Release RMB — returns to normal.
- Take damage from a bot or stand in the storm — red border flash appears briefly.

---

## Completion

All graphic upgrades are complete. Final verification checklist:

1. **Bloom** — chests glow, muzzle flash blooms, storm wall glows purple
2. **Vignette** — dark corners visible during gameplay
3. **Sky** — gradient sky sphere visible, fog matches horizon
4. **Textures** — grass on ground, concrete grid on buildings, bark on trees
5. **Clouds** — 14 white planes drifting slowly at high altitude
6. **Dust** — subtle particle haze around player at ground level
7. **Lightning** — purple flash appears near storm wall when storm shrinks
8. **Sparks** — hits produce 6 sparks each, white-hot fading to orange
9. **Sniper zoom** — RMB narrows FOV + scope bars when sniper equipped
10. **Damage flash** — red border pulse on taking any damage
