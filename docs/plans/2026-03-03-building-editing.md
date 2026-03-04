# Building, Editing, Settings & Loot Drops — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add Fortnite-style player building (wall/floor/ramp/cone in wood/stone/metal), a 10-preset edit system, bullet-destructible pieces, a settings menu (sensitivity + rebindable keys), and weapon loot drops from killed bots.

**Architecture:** All changes are in `~/Desktop/fortnite-game/index.html`. A `settings` object loaded from `localStorage` drives sensitivity and keybinds. A `placedPieces[]` array tracks all player-built structures. Build mode uses a ghost preview mesh snapped to a 4-unit world grid. Edit mode replaces a piece's single mesh with sub-panel meshes and updates `envObjects`. Loot drops live in `worldWeapons[]`, animated each game loop tick.

**Tech Stack:** Three.js r128 (already loaded). No new CDN scripts.

---

## Orientation

Key locations in `~/Desktop/fortnite-game/index.html`:

```
line   7   CSS styles block (inside <style>)
line  86   HTML body (canvas, overlays)
line 111   <script> block starts
line 117   CONSTANTS block
line 130   WEAPONS definitions
line 347   envObjects / chests declarations
line 933   INPUT section (keys object, mousemove, keydown listeners)
line 951   mousemove handler — sensitivity lives here
line 1000  startGame()
line 1200  game loop (loop function)
line 1226  updatePlayer(dt)
line 1450  killBot(bot)
line 1766  end of </script>
```

---

## Task 1: Settings — CSS, HTML, localStorage, sensitivity, and keybinds

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html`

### Step 1: Add CSS for the settings overlay

Find the closing `</style>` tag (around line 84) and insert before it:

```css
    /* ── Settings overlay ── */
    #settingsOverlay {
      position: fixed; top: 0; left: 0;
      width: 100%; height: 100%;
      background: rgba(0,0,0,0.88);
      display: none; flex-direction: column;
      align-items: center; justify-content: flex-start;
      padding-top: 40px;
      color: #fff; font-family: 'Segoe UI', sans-serif;
      z-index: 20; overflow-y: auto;
    }
    #settingsOverlay h2 { font-size: 2em; color: #f0c040; margin-bottom: 20px; }
    .settingsTabs { display: flex; gap: 10px; margin-bottom: 24px; }
    .settingsTab {
      padding: 8px 22px; background: #333; border: none;
      color: #ccc; border-radius: 6px; cursor: pointer; font-size: 1em;
    }
    .settingsTab.active { background: #f0c040; color: #111; font-weight: bold; }
    .settingsPanel { display: none; width: 480px; }
    .settingsPanel.active { display: block; }
    .settingsRow {
      display: flex; justify-content: space-between;
      align-items: center; padding: 10px 0;
      border-bottom: 1px solid #333;
    }
    .settingsRow label { font-size: 0.95em; color: #ccc; }
    .settingsRow input[type=range] { width: 180px; accent-color: #f0c040; }
    .bindBtn {
      padding: 5px 14px; background: #444; border: 1px solid #666;
      color: #fff; border-radius: 4px; cursor: pointer; font-size: 0.9em;
      min-width: 80px; text-align: center;
    }
    .bindBtn.listening { background: #f0c040; color: #111; }
    .bindBtn.conflict  { border-color: #ff4444; color: #ff4444; }
    #settingsClose {
      margin-top: 28px; padding: 8px 32px;
      background: #f0c040; color: #111; border: none;
      border-radius: 6px; cursor: pointer; font-size: 1em; font-weight: bold;
    }
```

### Step 2: Add the settings overlay HTML

Find `<!-- MAIN MENU OVERLAY -->` (around line 94) and insert this block immediately before it:

```html
<!-- SETTINGS OVERLAY -->
<div id="settingsOverlay">
  <h2>SETTINGS</h2>
  <div class="settingsTabs">
    <button class="settingsTab active" data-tab="controls">Controls</button>
    <button class="settingsTab" data-tab="mouse">Mouse</button>
    <button class="settingsTab" data-tab="graphics">Graphics</button>
  </div>

  <!-- Controls tab -->
  <div class="settingsPanel active" id="tab-controls">
    <div class="settingsRow"><label>Move Forward</label>   <button class="bindBtn" data-action="forward"></button></div>
    <div class="settingsRow"><label>Move Back</label>      <button class="bindBtn" data-action="back"></button></div>
    <div class="settingsRow"><label>Move Left</label>      <button class="bindBtn" data-action="left"></button></div>
    <div class="settingsRow"><label>Move Right</label>     <button class="bindBtn" data-action="right"></button></div>
    <div class="settingsRow"><label>Jump</label>           <button class="bindBtn" data-action="jump"></button></div>
    <div class="settingsRow"><label>Crouch</label>         <button class="bindBtn" data-action="crouch"></button></div>
    <div class="settingsRow"><label>Build Mode</label>     <button class="bindBtn" data-action="build"></button></div>
    <div class="settingsRow"><label>Edit Mode</label>      <button class="bindBtn" data-action="edit"></button></div>
    <div class="settingsRow"><label>Reload</label>         <button class="bindBtn" data-action="reload"></button></div>
    <div class="settingsRow"><label>Place Wall</label>     <button class="bindBtn" data-action="pieceWall"></button></div>
    <div class="settingsRow"><label>Place Floor</label>    <button class="bindBtn" data-action="pieceFloor"></button></div>
    <div class="settingsRow"><label>Place Ramp</label>     <button class="bindBtn" data-action="pieceRamp"></button></div>
    <div class="settingsRow"><label>Place Cone</label>     <button class="bindBtn" data-action="pieceCone"></button></div>
  </div>

  <!-- Mouse tab -->
  <div class="settingsPanel" id="tab-mouse">
    <div class="settingsRow">
      <label>Look Sensitivity</label>
      <input type="range" id="sensitivitySlider" min="0.1" max="3.0" step="0.05" value="1.0">
      <span id="sensitivityVal">1.00×</span>
    </div>
  </div>

  <!-- Graphics tab -->
  <div class="settingsPanel" id="tab-graphics">
    <div class="settingsRow">
      <label>Bloom Intensity</label>
      <input type="range" id="bloomSlider" min="0" max="2.0" step="0.05" value="1.0">
      <span id="bloomVal">1.00</span>
    </div>
  </div>

  <button id="settingsClose">CLOSE</button>
</div>
```

### Step 3: Add the settings JS block

Find the `// CONSTANTS & CONFIG` section (around line 114) and insert this entire block immediately before it (after `'use strict';`):

```javascript
// ════════════════════════════════════════════════════════════════
//  SETTINGS
// ════════════════════════════════════════════════════════════════
const DEFAULT_BINDS = {
  forward:    'KeyW',
  back:       'KeyS',
  left:       'KeyA',
  right:      'KeyD',
  jump:       'Space',
  crouch:     'KeyC',
  build:      'KeyQ',
  edit:       'KeyG',
  reload:     'KeyR',
  pieceWall:  'KeyZ',
  pieceFloor: 'KeyX',
  pieceRamp:  'KeyV',
  pieceCone:  'KeyF',
};

const settings = (() => {
  const saved = JSON.parse(localStorage.getItem('brSettings') || '{}');
  return {
    sensitivity: saved.sensitivity ?? 1.0,
    bloom:       saved.bloom       ?? 1.0,
    binds:       Object.assign({}, DEFAULT_BINDS, saved.binds || {}),
  };
})();

function saveSettings() {
  localStorage.setItem('brSettings', JSON.stringify({
    sensitivity: settings.sensitivity,
    bloom:       settings.bloom,
    binds:       settings.binds,
  }));
}

// Reverse map: keyCode → action (for quick lookup)
function buildReverseBinds() {
  const r = {};
  for (const [action, code] of Object.entries(settings.binds)) r[code] = action;
  return r;
}
let reverseBinds = buildReverseBinds();

// Key helper: is action currently held?
function isAction(action) { return !!keys[settings.binds[action]]; }

// ── Settings UI wiring ───────────────────────────────────────────
(function initSettingsUI() {
  const overlay  = document.getElementById('settingsOverlay');
  const tabs     = document.querySelectorAll('.settingsTab');
  const panels   = document.querySelectorAll('.settingsPanel');
  const bindBtns = document.querySelectorAll('.bindBtn');

  // Tab switching
  tabs.forEach(tab => {
    tab.addEventListener('click', () => {
      tabs.forEach(t => t.classList.remove('active'));
      panels.forEach(p => p.classList.remove('active'));
      tab.classList.add('active');
      document.getElementById('tab-' + tab.dataset.tab).classList.add('active');
    });
  });

  // Populate bind buttons with current key labels
  function codeToLabel(code) {
    return code.replace('Key','').replace('Digit','').replace('Arrow','↑→↓←'['Up,Right,Down,Left'.split(',').indexOf(code.replace('Arrow',''))]) || code;
  }
  function refreshBindBtns() {
    bindBtns.forEach(btn => {
      const action = btn.dataset.action;
      const code   = settings.binds[action];
      btn.textContent = codeToLabel(code);
      // Conflict detection
      const conflicts = Object.entries(settings.binds).filter(([a,c]) => c === code && a !== action);
      btn.classList.toggle('conflict', conflicts.length > 0);
    });
  }
  refreshBindBtns();

  // Rebind on click
  let listeningBtn = null;
  bindBtns.forEach(btn => {
    btn.addEventListener('click', () => {
      if (listeningBtn) { listeningBtn.classList.remove('listening'); listeningBtn.textContent = codeToLabel(settings.binds[listeningBtn.dataset.action]); }
      listeningBtn = btn;
      btn.classList.add('listening');
      btn.textContent = '…';
    });
  });
  document.addEventListener('keydown', e => {
    if (!listeningBtn) return;
    e.preventDefault();
    settings.binds[listeningBtn.dataset.action] = e.code;
    reverseBinds = buildReverseBinds();
    listeningBtn.classList.remove('listening');
    listeningBtn = null;
    refreshBindBtns();
    saveSettings();
  }, { capture: true });

  // Sensitivity slider
  const senSlider = document.getElementById('sensitivitySlider');
  const senVal    = document.getElementById('sensitivityVal');
  senSlider.value = settings.sensitivity;
  senVal.textContent = settings.sensitivity.toFixed(2) + '×';
  senSlider.addEventListener('input', () => {
    settings.sensitivity = parseFloat(senSlider.value);
    senVal.textContent = settings.sensitivity.toFixed(2) + '×';
    saveSettings();
  });

  // Bloom slider
  const bloomSlider = document.getElementById('bloomSlider');
  const bloomVal    = document.getElementById('bloomVal');
  bloomSlider.value = settings.bloom;
  bloomVal.textContent = settings.bloom.toFixed(2);
  bloomSlider.addEventListener('input', () => {
    settings.bloom = parseFloat(bloomSlider.value);
    bloomVal.textContent = settings.bloom.toFixed(2);
    if (typeof bloomPass !== 'undefined') bloomPass.strength = settings.bloom;
    saveSettings();
  });

  // Close button
  document.getElementById('settingsClose').addEventListener('click', closeSettings);
})();

let settingsOpen = false;
function openSettings() {
  settingsOpen = true;
  document.getElementById('settingsOverlay').style.display = 'flex';
  document.exitPointerLock();
}
function closeSettings() {
  settingsOpen = false;
  document.getElementById('settingsOverlay').style.display = 'none';
  if (state.phase === 'playing') canvas.requestPointerLock();
}
```

### Step 4: Wire Esc to open settings and apply sensitivity

Find the mousemove handler (around line 951):
```javascript
  player.yaw   -= e.movementX * 0.002;
  player.pitch -= e.movementY * 0.002;
```
Replace with:
```javascript
  player.yaw   -= e.movementX * 0.002 * settings.sensitivity;
  player.pitch -= e.movementY * 0.002 * settings.sensitivity;
```

Find the existing keydown listener that checks `'KeyE'` (around line 937). Add a new listener after it:
```javascript
document.addEventListener('keydown', e => {
  if (e.code === 'Escape') {
    if (settingsOpen) { closeSettings(); return; }
    if (state.phase === 'playing') openSettings();
  }
});
```

### Step 5: Apply bloom setting on startup

Find the line where `bloomPass` is created (search for `new UnrealBloomPass` — it's around the EffectComposer setup). After `bloomPass` is created add:
```javascript
bloomPass.strength = settings.bloom;
```

### Step 6: Swap movement checks to use `isAction()`

Find `updatePlayer(dt)` (line ~1226). Replace the hard-coded key checks:
```javascript
  player.crouching = !!keys['KeyC'];
```
with:
```javascript
  player.crouching = isAction('crouch');
```

And:
```javascript
  if (keys['KeyW']) move.addScaledVector(forward,  1);
  if (keys['KeyS']) move.addScaledVector(forward, -1);
  if (keys['KeyA']) move.addScaledVector(right,   -1);
  if (keys['KeyD']) move.addScaledVector(right,    1);
```
with:
```javascript
  if (isAction('forward')) move.addScaledVector(forward,  1);
  if (isAction('back'))    move.addScaledVector(forward, -1);
  if (isAction('left'))    move.addScaledVector(right,   -1);
  if (isAction('right'))   move.addScaledVector(right,    1);
```

And:
```javascript
  if (player.onGround && keys['Space']) {
```
with:
```javascript
  if (player.onGround && isAction('jump')) {
```

### Step 7: Verify

Open the game. Press Esc during play — the settings overlay should appear. Moving the sensitivity slider should change turn speed immediately. Click a bind button — it says "…" — press a key — it updates. Close. Movement should use the newly bound keys.

### Step 8: Commit

```bash
cd ~/Desktop/fortnite-game
git add index.html
git commit -m "feat: settings overlay with sensitivity slider, rebindable keys, bloom control"
```

---

## Task 2: Build mode — state, ghost piece, grid snap, and placement

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html`

### Step 1: Add piece geometry helpers and build state after the `envObjects` declaration

Find:
```javascript
const envObjects = [];   // used for collision
const chests = [];
```

Insert immediately after:

```javascript
// ════════════════════════════════════════════════════════════════
//  BUILDING SYSTEM
// ════════════════════════════════════════════════════════════════
const GRID = 4;
const PIECE_TYPES = ['wall', 'floor', 'ramp', 'cone'];

const MATERIALS_DEF = {
  wood:  { color: 0x8B5E3C, hp: 150, roughness: 0.90, metalness: 0.0 },
  stone: { color: 0x888888, hp: 300, roughness: 0.85, metalness: 0.05 },
  metal: { color: 0x7a8a9a, hp: 450, roughness: 0.40, metalness: 0.6  },
};

function snapGrid(v) { return Math.round(v / GRID) * GRID; }

function createRampGeo() {
  const g = new THREE.BufferGeometry();
  g.setAttribute('position', new THREE.BufferAttribute(new Float32Array([
    -2,0, 2,   2,0, 2,   2,0,-2,  -2,0,-2,  // bottom 0-3
    -2,4,-2,   2,4,-2,                        // top-back 4-5
  ]), 3));
  g.setIndex([0,2,1, 0,3,2, 0,1,5, 0,5,4, 3,4,5, 3,5,2, 0,4,3, 1,2,5]);
  g.computeVertexNormals();
  return g;
}

function createConeGeo() {
  const g = new THREE.BufferGeometry();
  g.setAttribute('position', new THREE.BufferAttribute(new Float32Array([
    -2,0, 2,   2,0, 2,   2,0,-2,  -2,0,-2,  // base 0-3
     0,4, 0,                                   // peak 4
  ]), 3));
  g.setIndex([0,2,1, 0,3,2, 0,1,4, 1,2,4, 2,3,4, 3,0,4]);
  g.computeVertexNormals();
  return g;
}

function getPieceGeo(type) {
  switch(type) {
    case 'wall':  return new THREE.BoxGeometry(4, 4, 0.3);
    case 'floor': return new THREE.BoxGeometry(4, 0.3, 4);
    case 'ramp':  return createRampGeo();
    case 'cone':  return createConeGeo();
  }
}

function pieceMat(matName, opacity) {
  const def = MATERIALS_DEF[matName];
  return new THREE.MeshStandardMaterial({
    color: def.color, roughness: def.roughness, metalness: def.metalness,
    transparent: opacity < 1, opacity: opacity ?? 1,
  });
}

// Build mode state
let buildMode    = false;
let buildType    = 'wall';      // 'wall'|'floor'|'ramp'|'cone'
let buildMat     = 'wood';      // 'wood'|'stone'|'metal'
let buildRot     = 0;           // rotation in steps of 90° (0-3)
const placedPieces = [];        // { mesh, hp, maxHp, type, mat, envObjs:[] }

// Ghost mesh (single piece, semi-transparent)
let ghostMesh = null;
function rebuildGhost() {
  if (ghostMesh) scene.remove(ghostMesh);
  ghostMesh = new THREE.Mesh(getPieceGeo(buildType), pieceMat(buildMat, 0.45));
  ghostMesh.visible = false;
  scene.add(ghostMesh);
}
```

### Step 2: Add ghost update and placement functions

Insert immediately after the block from Step 1:

```javascript
function updateGhost() {
  if (!buildMode || !ghostMesh) { if (ghostMesh) ghostMesh.visible = false; return; }

  // Cast ray from camera center
  const ray = new THREE.Raycaster();
  ray.setFromCamera(new THREE.Vector2(0, 0), camera);

  // Collect hittable objects: terrain + placed pieces (but not ghost itself)
  const targets = [terrainMesh, ...placedPieces.map(p => p.mesh)];
  const hits = ray.intersectObjects(targets, true);

  if (!hits.length || hits[0].distance > 10) { ghostMesh.visible = false; return; }

  const hit = hits[0];
  let gx, gy, gz, ry = 0;

  if (buildType === 'wall') {
    // Snap to vertical face: find nearest grid-aligned wall in XZ, height snapped
    const nx = Math.abs(hit.normal.x), nz = Math.abs(hit.normal.z);
    if (nx > nz) {
      // X-facing wall
      gx = snapGrid(hit.point.x + hit.normal.x * 0.15);
      gz = snapGrid(hit.point.z);
      ry = Math.PI / 2;
    } else {
      // Z-facing wall
      gx = snapGrid(hit.point.x);
      gz = snapGrid(hit.point.z + hit.normal.z * 0.15);
      ry = 0;
    }
    gy = snapGrid(hit.point.y - 2) + 2; // snap bottom to grid, center is +2
  } else if (buildType === 'floor') {
    gx = snapGrid(hit.point.x);
    gz = snapGrid(hit.point.z);
    gy = snapGrid(hit.point.y) + 0.15;
    ry = 0;
  } else if (buildType === 'ramp') {
    gx = snapGrid(hit.point.x);
    gz = snapGrid(hit.point.z);
    gy = snapGrid(hit.point.y);
    // Face player's yaw snapped to 90°
    ry = Math.round(player.yaw / (Math.PI/2)) * (Math.PI/2);
  } else { // cone
    gx = snapGrid(hit.point.x);
    gz = snapGrid(hit.point.z);
    gy = snapGrid(hit.point.y);
    ry = Math.round(player.yaw / (Math.PI/2)) * (Math.PI/2);
  }

  // Apply manual rotation override
  ry += buildRot * (Math.PI / 2);

  ghostMesh.position.set(gx, gy, gz);
  ghostMesh.rotation.y = ry;
  ghostMesh.visible = true;
}

function placePiece() {
  if (!buildMode || !ghostMesh || !ghostMesh.visible) return;

  const def  = MATERIALS_DEF[buildMat];
  const mesh = new THREE.Mesh(getPieceGeo(buildType), pieceMat(buildMat, 1));
  mesh.position.copy(ghostMesh.position);
  mesh.rotation.copy(ghostMesh.rotation);
  mesh.castShadow = true;
  mesh.receiveShadow = true;
  scene.add(mesh);

  // Compute AABB for envObjects
  const bx = mesh.position.x, bz = mesh.position.z;
  const envObj = buildType === 'wall'
    ? (Math.abs(mesh.rotation.y % Math.PI) < 0.1
        ? { x: bx, z: bz, w: 2.15, d: 0.25 }   // Z-facing wall: wide in X, thin in Z
        : { x: bx, z: bz, w: 0.25, d: 2.15 })   // X-facing wall: thin in X, wide in Z
    : { x: bx, z: bz, w: 2.1, d: 2.1 };         // floor/ramp/cone: square footprint

  envObjects.push(envObj);

  placedPieces.push({
    mesh, hp: def.hp, maxHp: def.hp,
    type: buildType, mat: buildMat,
    envObjs: [envObj],
    rotation: mesh.rotation.y,
  });
}
```

### Step 3: Wire build mode inputs

Find the INPUT section (around line 933) and add new listeners after the existing ones:

```javascript
// Build mode toggle (Q)
document.addEventListener('keydown', e => {
  if (state.phase !== 'playing' || settingsOpen) return;
  if (e.code === settings.binds.build) {
    buildMode = !buildMode;
    if (buildMode) { rebuildGhost(); }
    else if (ghostMesh) { ghostMesh.visible = false; }
  }
  // Direct piece selection
  if (buildMode) {
    if (e.code === settings.binds.pieceWall)  { buildType = 'wall';  rebuildGhost(); }
    if (e.code === settings.binds.pieceFloor) { buildType = 'floor'; rebuildGhost(); }
    if (e.code === settings.binds.pieceRamp)  { buildType = 'ramp';  rebuildGhost(); }
    if (e.code === settings.binds.pieceCone)  { buildType = 'cone';  rebuildGhost(); }
    // Material selection
    if (e.code === 'Digit1') { buildMat = 'wood';  rebuildGhost(); }
    if (e.code === 'Digit2') { buildMat = 'stone'; rebuildGhost(); }
    if (e.code === 'Digit3') { buildMat = 'metal'; rebuildGhost(); }
  }
});

// Scroll wheel: cycle piece type in build mode
canvas.addEventListener('wheel', e => {
  if (!buildMode || state.phase !== 'playing') return;
  const dir = e.deltaY > 0 ? 1 : -1;
  const idx = (PIECE_TYPES.indexOf(buildType) + dir + PIECE_TYPES.length) % PIECE_TYPES.length;
  buildType = PIECE_TYPES[idx];
  rebuildGhost();
}, { passive: true });

// Right-click: rotate ghost
document.addEventListener('contextmenu', e => {
  if (buildMode && state.phase === 'playing') {
    e.preventDefault();
    buildRot = (buildRot + 1) % 4;
  }
});

// Left-click: place piece (add to existing mousedown or click handler)
canvas.addEventListener('mousedown', e => {
  if (state.phase !== 'playing' || settingsOpen) return;
  if (buildMode && e.button === 0) { placePiece(); return; }
});
```

### Step 4: Call `updateGhost()` in the game loop

Find inside `loop(t)`:
```javascript
    updateLightning(dt);
```
Add after it:
```javascript
    if (buildMode) updateGhost();
```

### Step 5: Verify

Open the game. Press Q — ghost piece appears in front of you as a semi-transparent wall. Scroll wheel cycles through wall/floor/ramp/cone. Right-click rotates. Left-click places (a solid piece appears). Press 1/2/3 to change material color. Press Q again to exit build mode — ghost disappears.

### Step 6: Commit

```bash
cd ~/Desktop/fortnite-game
git add index.html
git commit -m "feat: build mode with ghost preview, grid snap, four piece types, three materials"
```

---

## Task 3: Piece destruction — HP, color tint, bullet raycast

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html`

### Step 1: Add `damagePiece()` function

Find `function killBot(bot)` (around line 1450) and insert before it:

```javascript
function damagePiece(piece, dmg) {
  piece.hp -= dmg;
  // Darken tint based on remaining HP
  const t = Math.max(0, piece.hp / piece.maxHp);
  const base = MATERIALS_DEF[piece.mat].color;
  const r = ((base >> 16) & 0xff) / 255;
  const g = ((base >>  8) & 0xff) / 255;
  const b = ( base        & 0xff) / 255;
  piece.mesh.traverse(child => {
    if (child.isMesh) child.material.color.setRGB(r * (0.35 + t * 0.65), g * (0.35 + t * 0.65), b * (0.35 + t * 0.65));
  });
  if (piece.hp <= 0) destroyPiece(piece);
}

function destroyPiece(piece) {
  scene.remove(piece.mesh);
  // Remove all envObjects entries belonging to this piece
  piece.envObjs.forEach(obj => {
    const idx = envObjects.indexOf(obj);
    if (idx !== -1) envObjects.splice(idx, 1);
  });
  const idx = placedPieces.indexOf(piece);
  if (idx !== -1) placedPieces.splice(idx, 1);
}
```

### Step 2: Add placed-piece raycast check to the shoot function

Find the existing shoot / fire logic. Search for `ray.intersectObject(bot.mesh, true)` — the shooting raycast (around line 679 area). Read the surrounding context to find the full shoot function. It should look something like:

```javascript
  const ray = new THREE.Raycaster();
  ray.setFromCamera(new THREE.Vector2(...), camera);
  // ... intersect bots ...
```

After the existing bot intersection code but before applying bot damage, add a placed-piece check. Find where bot hit is processed and insert before it:

```javascript
  // Check placed pieces first (closer targets block bot hits)
  if (placedPieces.length > 0) {
    const pieceMeshes = placedPieces.map(p => p.mesh);
    const pieceHits = ray.intersectObjects(pieceMeshes, true);
    if (pieceHits.length > 0) {
      const hitMesh = pieceHits[0].object;
      const hitPiece = placedPieces.find(p => {
        let found = false;
        p.mesh.traverse(c => { if (c === hitMesh) found = true; });
        return found;
      });
      if (hitPiece && (!botHit || pieceHits[0].distance < botHit.distance)) {
        damagePiece(hitPiece, WEAPONS[player.weapon].damage);
        spawnImpact(pieceHits[0].point);
        return; // bullet consumed by piece
      }
    }
  }
```

(The exact insertion point depends on the shape of the existing shoot function — read it carefully and find the right place before damage is applied to a bot.)

### Step 3: Verify

Open the game. Build a wall. Shoot at it — it should darken progressively. After enough shots (150 for wood) it disappears and you can walk through.

### Step 4: Commit

```bash
cd ~/Desktop/fortnite-game
git add index.html
git commit -m "feat: placed pieces take bullet damage, darken, and are destroyed at 0 HP"
```

---

## Task 4: Edit mode — preset cycles, sub-panel visuals, collision update

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html`

### Step 1: Add edit mode state and preset definitions

Find the `// BUILDING SYSTEM` section (added in Task 2) and append after `const placedPieces = [];`:

```javascript
// Edit mode state
let editMode   = false;
let editPiece  = null;   // the placedPiece being edited
let editPreset = 0;      // current preset index
let editPanels = [];     // array of sub-panel meshes shown during editing

// ── Wall edit presets ────────────────────────────────────────────
// 3×3 grid indices: 0=TL 1=TC 2=TR 3=ML 4=MC 5=MR 6=BL 7=BC 8=BR
const WALL_PRESETS = [
  { name:'Door Left',       remove:[6,3] },
  { name:'Door Center',     remove:[7,4] },
  { name:'Door Right',      remove:[8,5] },
  { name:'Window Left',     remove:[3]   },
  { name:'Window Center',   remove:[4]   },
  { name:'Window Right',    remove:[5]   },
  { name:'L-Open Left',     remove:[6,7,3] },
  { name:'L-Open Right',    remove:[7,8,5] },
  { name:'Arch Left',       remove:[6,7,3,4] },
  { name:'Arch Right',      remove:[7,8,4,5] },
];

// Floor 2×2 indices: 0=TL 1=TR 2=BL 3=BR (diagonal quadrants)
const FLOOR_PRESETS = [
  { name:'Remove TL',    remove:[0] },
  { name:'Remove TR',    remove:[1] },
  { name:'Remove BL',    remove:[2] },
  { name:'Remove BR',    remove:[3] },
  { name:'Remove Left',  remove:[0,2] },
  { name:'Remove Right', remove:[1,3] },
  { name:'Remove Top',   remove:[0,1] },
  { name:'Remove Bot',   remove:[2,3] },
];

// Ramp 2×2: 0=TL-ramp 1=TR-ramp 2=BL-floor 3=BR-floor
const RAMP_PRESETS = [
  { name:'Half Left',     remove:[0,2] },
  { name:'Half Right',    remove:[1,3] },
  { name:'Ramp Only',     remove:[2,3] },
  { name:'Floor Only',    remove:[0,1] },
  { name:'Remove TL',     remove:[0]   },
  { name:'Remove TR',     remove:[1]   },
  { name:'Remove BL',     remove:[2]   },
  { name:'Remove BR',     remove:[3]   },
];

// Cone 4 triangles: 0=FL 1=FR 2=BL 3=BR
const CONE_PRESETS = [
  { name:'Remove FL',     remove:[0]   },
  { name:'Remove FR',     remove:[1]   },
  { name:'Remove BL',     remove:[2]   },
  { name:'Remove BR',     remove:[3]   },
  { name:'U-Shape',       remove:[0,1] },
  { name:'L-Shape TL',    remove:[0,2] },
  { name:'L-Shape TR',    remove:[1,3] },
  { name:'Open Back',     remove:[2,3] },
];

function getPresets(type) {
  return { wall:WALL_PRESETS, floor:FLOOR_PRESETS, ramp:RAMP_PRESETS, cone:CONE_PRESETS }[type];
}
```

### Step 2: Add sub-panel mesh builders

Append after the presets block:

```javascript
// Panel dimensions for the 3×3 wall grid (each panel is 4/3 × 4/3 × 0.3)
const WP = 4/3;
// Wall panel centers (TL→BR, row-major): [x, y] offsets from wall center
const WALL_PANEL_OFFSETS = [
  [-WP, WP],[0, WP],[WP, WP],
  [-WP, 0 ],[0, 0 ],[WP, 0 ],
  [-WP,-WP],[0,-WP],[WP,-WP],
];

function buildEditPanels(piece) {
  // Remove old edit panel meshes
  editPanels.forEach(m => scene.remove(m));
  editPanels = [];

  const mat = (removed) => new THREE.MeshStandardMaterial({
    color: removed ? 0x4488ff : MATERIALS_DEF[piece.mat].color,
    transparent: true, opacity: removed ? 0.35 : 0.8,
    roughness: 0.85, metalness: 0.0,
  });

  const presets = getPresets(piece.type);
  const removed = new Set(presets[editPreset].remove);

  if (piece.type === 'wall') {
    WALL_PANEL_OFFSETS.forEach(([ox, oy], i) => {
      const m = new THREE.Mesh(new THREE.BoxGeometry(WP - 0.05, WP - 0.05, 0.32), mat(removed.has(i)));
      // Transform into piece local space → world space
      const local = new THREE.Vector3(ox, oy, 0);
      local.applyQuaternion(piece.mesh.quaternion);
      m.position.copy(piece.mesh.position).add(local);
      m.rotation.copy(piece.mesh.rotation);
      scene.add(m);
      editPanels.push(m);
    });
  } else if (piece.type === 'floor') {
    // 4 quadrant panels (2×2 triangular approximated as boxes for display)
    const offsets = [[-1,0,-1],[1,0,-1],[-1,0,1],[1,0,1]];
    offsets.forEach(([ox,oy,oz], i) => {
      const m = new THREE.Mesh(new THREE.BoxGeometry(1.95, 0.32, 1.95), mat(removed.has(i)));
      const local = new THREE.Vector3(ox, oy, oz);
      local.applyQuaternion(piece.mesh.quaternion);
      m.position.copy(piece.mesh.position).add(local);
      m.rotation.copy(piece.mesh.rotation);
      scene.add(m);
      editPanels.push(m);
    });
  } else if (piece.type === 'ramp') {
    // 4 panels: TL/TR (ramp top), BL/BR (floor bottom)
    const offsets = [[-1,2,-1],[1,2,-1],[-1,0,1],[1,0,1]];
    offsets.forEach(([ox,oy,oz], i) => {
      const m = new THREE.Mesh(new THREE.BoxGeometry(1.95, 0.32, 1.95), mat(removed.has(i)));
      const local = new THREE.Vector3(ox, oy, oz);
      local.applyQuaternion(piece.mesh.quaternion);
      m.position.copy(piece.mesh.position).add(local);
      m.rotation.copy(piece.mesh.rotation);
      scene.add(m);
      editPanels.push(m);
    });
  } else { // cone — 4 triangle panels
    const offsets = [[-1,1, 1],[1,1, 1],[-1,1,-1],[1,1,-1]];
    offsets.forEach(([ox,oy,oz], i) => {
      const m = new THREE.Mesh(new THREE.BoxGeometry(1.95, 1.95, 1.95), mat(removed.has(i)));
      const local = new THREE.Vector3(ox, oy, oz);
      local.applyQuaternion(piece.mesh.quaternion);
      m.position.copy(piece.mesh.position).add(local);
      m.rotation.copy(piece.mesh.rotation);
      scene.add(m);
      editPanels.push(m);
    });
  }
}

function clearEditPanels() {
  editPanels.forEach(m => scene.remove(m));
  editPanels = [];
}
```

### Step 3: Add `applyEdit()` which rebuilds the piece geometry

Append after `clearEditPanels()`:

```javascript
function applyEdit() {
  if (!editPiece) return;
  const presets  = getPresets(editPiece.type);
  const preset   = presets[editPreset];
  const removed  = new Set(preset.remove);

  if (editPiece.type === 'wall') {
    // Replace single mesh with up to 9 sub-panel boxes
    scene.remove(editPiece.mesh);
    const group = new THREE.Group();
    group.position.copy(editPiece.mesh.position);
    group.rotation.copy(editPiece.mesh.rotation);
    const baseMat = pieceMat(editPiece.mat, 1);
    WALL_PANEL_OFFSETS.forEach(([ox, oy], i) => {
      if (removed.has(i)) return;
      const m = new THREE.Mesh(new THREE.BoxGeometry(WP - 0.05, WP - 0.05, 0.3), baseMat.clone());
      m.position.set(ox, oy, 0);
      m.castShadow = true;
      group.add(m);
    });
    scene.add(group);
    editPiece.mesh = group;

    // Update collision: remove old envObj, add per-column entries
    editPiece.envObjs.forEach(o => {
      const idx = envObjects.indexOf(o);
      if (idx !== -1) envObjects.splice(idx, 1);
    });
    editPiece.envObjs = [];

    // Re-add collision only for remaining columns/rows
    // Keep it simple: left column, center column, right column as thin slabs
    const isZFacing = Math.abs(editPiece.rotation % Math.PI) < 0.1;
    const bx = editPiece.mesh.position.x;
    const bz = editPiece.mesh.position.z;
    // Column 0 (left), 1 (center), 2 (right)
    for (let col = 0; col < 3; col++) {
      const rows = [col, col+3, col+6]; // panel indices in this column
      const allRemoved = rows.every(r => removed.has(r));
      if (!allRemoved) {
        const offset = (col - 1) * WP;
        const obj = isZFacing
          ? { x: bx + offset, z: bz, w: WP/2, d: 0.25 }
          : { x: bx, z: bz + offset, w: 0.25, d: WP/2 };
        envObjects.push(obj);
        editPiece.envObjs.push(obj);
      }
    }
  }
  // floor/ramp/cone: geometry stays; just hide removed sub-panels visually
  // (collision footprint stays the same — acceptable for non-wall types)

  clearEditPanels();
  editMode  = false;
  editPiece = null;
}

function cancelEdit() {
  clearEditPanels();
  editMode  = false;
  editPiece = null;
}
```

### Step 4: Wire edit mode key inputs

Add to the build-mode keydown listener (from Task 2, the one checking `settings.binds.build`):

```javascript
  // Edit mode (G)
  if (e.code === settings.binds.edit) {
    if (editMode) {
      applyEdit();
      return;
    }
    if (buildMode) { buildMode = false; if (ghostMesh) ghostMesh.visible = false; }
    // Find aimed piece
    const ray = new THREE.Raycaster();
    ray.setFromCamera(new THREE.Vector2(0, 0), camera);
    const meshes = placedPieces.map(p => p.mesh);
    const hits   = ray.intersectObjects(meshes, true);
    if (hits.length && hits[0].distance < 12) {
      const hitMesh = hits[0].object;
      editPiece = placedPieces.find(p => {
        let f = false;
        p.mesh.traverse(c => { if (c === hitMesh || p.mesh === hitMesh) f = true; });
        return f;
      });
      if (editPiece) {
        editMode   = true;
        editPreset = 0;
        buildEditPanels(editPiece);
      }
    }
  }
  // Cancel edit with Esc (add to Esc handler)
```

Also update the Esc handler (from Task 1) to cancel edit mode first:
```javascript
document.addEventListener('keydown', e => {
  if (e.code === 'Escape') {
    if (editMode) { cancelEdit(); return; }
    if (settingsOpen) { closeSettings(); return; }
    if (state.phase === 'playing') openSettings();
  }
});
```

Add scroll-wheel cycling for edit presets. Find the existing wheel listener and extend it:
```javascript
canvas.addEventListener('wheel', e => {
  if (state.phase !== 'playing') return;
  if (editMode && editPiece) {
    const presets = getPresets(editPiece.type);
    const dir = e.deltaY > 0 ? 1 : -1;
    editPreset = (editPreset + dir + presets.length) % presets.length;
    buildEditPanels(editPiece);
    return;
  }
  if (!buildMode) return;
  // ... existing piece-type cycling ...
}, { passive: true });
```

### Step 5: Verify

Build a wall. Aim at it, press G — colored sub-panels appear highlighting the door preset. Scroll wheel cycles through presets. Press G again — the wall now has a door gap. Player can walk through the gap.

### Step 6: Commit

```bash
cd ~/Desktop/fortnite-game
git add index.html
git commit -m "feat: edit mode with 10 wall presets, floor/ramp/cone panel highlights, collision update"
```

---

## Task 5: Loot drops — spawn, animate, collect, despawn

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html`

### Step 1: Add `worldWeapons[]` array near the top

Find:
```javascript
const envObjects = [];   // used for collision
const chests = [];
```
Add after:
```javascript
const worldWeapons = [];  // dropped loot: { mesh, light, weaponType, x, y, z, bobT, lifetime }
```

### Step 2: Add `spawnLoot()` function

Find `function damagePiece(piece, dmg)` (added in Task 3) and insert before it:

```javascript
const WEAPON_LOOT_COLORS = {
  pistol: 0xaaaaaa, rifle: 0x556655, shotgun: 0x8B4513, sniper: 0x334455, smg: 0x888888,
};

function spawnLoot(x, y, z, weaponType) {
  const color  = WEAPON_LOOT_COLORS[weaponType] || 0xffffff;
  const group  = new THREE.Group();

  // Small box representing the weapon
  const body = new THREE.Mesh(
    new THREE.BoxGeometry(0.5, 0.15, 1.0),
    new THREE.MeshStandardMaterial({ color, emissive: color, emissiveIntensity: 0.4, roughness: 0.6, metalness: 0.4 })
  );
  group.add(body);

  // Glow light (small range to stay performant)
  const light = new THREE.PointLight(color, 1.2, 4);
  light.position.y = 0.4;
  group.add(light);

  group.position.set(x, y + 0.8, z);
  scene.add(group);

  worldWeapons.push({ mesh: group, light, weaponType, x, y: y + 0.8, z, bobT: Math.random() * Math.PI * 2, lifetime: 60 });
}

function updateWorldWeapons(dt) {
  for (let i = worldWeapons.length - 1; i >= 0; i--) {
    const w = worldWeapons[i];
    w.lifetime -= dt;

    // Bob and rotate
    w.bobT += dt * 2;
    w.mesh.position.y = w.y + Math.sin(w.bobT) * 0.18;
    w.mesh.rotation.y += dt * 1.5;

    // Auto-collect if player is close
    const dx = camera.position.x - w.x;
    const dz = camera.position.z - w.z;
    if (Math.sqrt(dx*dx + dz*dz) < 1.5) {
      giveWeapon(w.weaponType);
      scene.remove(w.mesh);
      worldWeapons.splice(i, 1);
      continue;
    }

    // Despawn after 60 seconds
    if (w.lifetime <= 0) {
      scene.remove(w.mesh);
      worldWeapons.splice(i, 1);
    }
  }
}
```

### Step 3: Call `updateWorldWeapons(dt)` in the game loop

Find inside `loop(t)`:
```javascript
    if (buildMode) updateGhost();
```
Add after it:
```javascript
    updateWorldWeapons(dt);
```

### Step 4: Call `spawnLoot()` from `killBot()`

Find `function killBot(bot)` (line ~1450):
```javascript
function killBot(bot) {
  bot.alive = false;
  bot.deathTimer = 2;
```

Add after `bot.deathTimer = 2;`:
```javascript
  // Drop bot's weapon as a world pickup
  spawnLoot(bot.x, bot.y, bot.z, bot.weapon || 'pistol');
```

(Note: `bot.weapon` may not exist — check the bot object structure. If bots don't have a `weapon` property, use `'pistol'` as default or randomize from the WEAPONS keys.)

### Step 5: Verify

Kill a bot. A glowing bobbing pickup should appear at their position. Walk over it — it disappears and you gain (or top up ammo for) their weapon type.

### Step 6: Commit

```bash
cd ~/Desktop/fortnite-game
git add index.html
git commit -m "feat: weapon loot drops from killed bots, bob animation, auto-collect, 60s despawn"
```

---

## Completion Checklist

1. Esc opens settings overlay, Esc again closes it and resumes game
2. Sensitivity slider changes mouse look speed immediately and persists across reload
3. Rebinding a key updates movement/build controls immediately
4. Press Q to enter build mode — ghost piece appears, semi-transparent
5. Scroll wheel cycles wall/floor/ramp/cone; 1/2/3 changes material
6. Right-click rotates ghost 90°; left-click places solid piece
7. Placed pieces block player movement (envObjects)
8. Shooting a placed piece darkens it; enough shots destroys it
9. Aim at placed wall, press G — colored sub-panels show current preset
10. Scroll cycles presets (door left, window center, arch, etc.)
11. Press G to confirm — wall now has the opening; player can walk through door presets
12. Killing a bot drops a glowing bobbing pickup at their position
13. Walking over the pickup collects the weapon / tops up ammo
14. Pickup despawns after 60 seconds if not collected
