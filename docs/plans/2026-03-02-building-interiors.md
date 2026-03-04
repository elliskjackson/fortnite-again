# Building Interiors — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace solid one-piece `BoxGeometry` buildings with hollow 5-panel structures (4 walls + floor) that the player can physically enter, with a door gap, interior ambient light, and one chest inside.

**Architecture:** All changes are in `~/Desktop/fortnite-game/index.html`. `addBuilding()` is rewritten to build wall-panel slabs with per-panel `envObjects` AABB entries. `const chests = []` is moved earlier (before `generateEnvironment()` runs) so `addBuilding()` can push interior chests to it.

**Tech Stack:** Three.js r128 (already loaded). No new CDN scripts required.

---

## Critical Ordering Fix (prerequisite for both tasks)

`const chests = []` is currently at line 462 — **after** `generateEnvironment()` runs at line 425. `addBuilding()` is called inside `generateEnvironment()`, so it cannot push to `chests[]` at line 462 without a temporal dead zone / undefined reference error.

**Fix:** Move `const chests = [];` to before `generateEnvironment()` (around line 347, right after `envObjects`). This is handled in Task 1 below.

---

## Task 1: Rewrite `addBuilding()` with hollow walls, door gap, and interior light

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html`

**Step 1: Move `const chests = []` before `generateEnvironment()`**

Find this block (currently around line 459–462):
```javascript
// ════════════════════════════════════════════════════════════════
//  CHESTS
// ════════════════════════════════════════════════════════════════
const chests = [];
```

Delete **only** the `const chests = [];` line from that location (leave the comment and the `spawnChests` IIFE in place).

Then find this line (around line 347):
```javascript
const envObjects = [];   // used for collision
```

Insert `const chests = [];` immediately after it:
```javascript
const envObjects = [];   // used for collision
const chests = [];
```

**Step 2: Replace `addBuilding()` with the hollow version**

Find the entire `addBuilding` function (lines 361–387):
```javascript
function addBuilding(x, z, seed) {
  const w = 6 + seededRand(seed)   * 10;
  const h = 4 + seededRand(seed+1) * 12;
  const d = 6 + seededRand(seed+2) * 10;
  const geo = new THREE.BoxGeometry(w, h, d);
  const mat = new THREE.MeshStandardMaterial({
    color: new THREE.Color().setHSL(0.08 + seededRand(seed+3)*0.05, 0.12, 0.48 + seededRand(seed+4)*0.15),
    map: concreteTex,
    roughness: 0.85,
    metalness: 0.05
  });
  const mesh = new THREE.Mesh(geo, mat);
  const y = placeOnTerrain(x, z);
  mesh.position.set(x, y + h/2, z);
  mesh.castShadow = true;
  mesh.receiveShadow = true;
  scene.add(mesh);
  envObjects.push({ x, z, w: w/2 + 0.5, d: d/2 + 0.5 });

  // Roof detail
  const roofGeo = new THREE.BoxGeometry(w + 0.5, 0.4, d + 0.5);
  const roofMat = new THREE.MeshStandardMaterial({ color: 0x444444, roughness: 0.9, metalness: 0.1 });
  const roof = new THREE.Mesh(roofGeo, roofMat);
  roof.position.set(x, y + h + 0.2, z);
  roof.castShadow = true;
  scene.add(roof);
}
```

Replace it with:
```javascript
function addBuilding(x, z, seed) {
  const w = 6 + seededRand(seed)   * 10;
  const h = 4 + seededRand(seed+1) * 12;
  const d = 6 + seededRand(seed+2) * 10;
  const groundY = placeOnTerrain(x, z);

  // Material — exterior hue
  const hue = 0.08 + seededRand(seed+3) * 0.05;
  const lightness = 0.48 + seededRand(seed+4) * 0.15;
  const exteriorColor = new THREE.Color().setHSL(hue, 0.12, lightness);
  const interiorColor = new THREE.Color().setHSL(hue, 0.12, Math.min(1, lightness + 0.08));
  function wallMat(isInterior) {
    return new THREE.MeshStandardMaterial({
      color: isInterior ? interiorColor : exteriorColor,
      map: concreteTex,
      roughness: 0.85,
      metalness: 0.05
    });
  }

  // Door dimensions and side
  const doorW = 1.4;
  const doorH = 2.4;
  const doorSideW = (w - doorW) / 2;
  // seededRand(seed+7) > 0.5 → door on +Z side, else -Z side
  const doorSign = seededRand(seed+7) > 0.5 ? 1 : -1;
  const frontZ = z + doorSign * (d / 2);
  const backZ  = z - doorSign * (d / 2);

  // Helper: add a wall panel, cast shadow, add to scene
  function addPanel(geo, px, py, pz, isInterior) {
    const m = new THREE.Mesh(geo, wallMat(isInterior));
    m.position.set(px, py, pz);
    m.castShadow = true;
    m.receiveShadow = true;
    scene.add(m);
  }

  const wallThick = 0.3;
  const midY = groundY + h / 2;   // vertical center of walls

  // Left wall
  addPanel(new THREE.BoxGeometry(wallThick, h, d), x - w/2, midY, z, false);
  envObjects.push({ x: x - w/2, z, w: wallThick/2, d: d/2 });

  // Right wall
  addPanel(new THREE.BoxGeometry(wallThick, h, d), x + w/2, midY, z, false);
  envObjects.push({ x: x + w/2, z, w: wallThick/2, d: d/2 });

  // Back wall (full, no door)
  addPanel(new THREE.BoxGeometry(w, h, wallThick), x, midY, backZ, false);
  envObjects.push({ x, z: backZ, w: w/2, d: wallThick/2 });

  // Front wall — top beam (above door gap)
  const beamH = h - doorH;
  addPanel(
    new THREE.BoxGeometry(w, beamH, wallThick),
    x,
    groundY + doorH + beamH / 2,
    frontZ,
    false
  );
  envObjects.push({ x, z: frontZ, w: w/2, d: wallThick/2 });

  // Front wall — left pier
  addPanel(
    new THREE.BoxGeometry(doorSideW, doorH, wallThick),
    x - doorW/2 - doorSideW/2,
    groundY + doorH / 2,
    frontZ,
    false
  );
  envObjects.push({ x: x - doorW/2 - doorSideW/2, z: frontZ, w: doorSideW/2, d: wallThick/2 });

  // Front wall — right pier
  addPanel(
    new THREE.BoxGeometry(doorSideW, doorH, wallThick),
    x + doorW/2 + doorSideW/2,
    groundY + doorH / 2,
    frontZ,
    false
  );
  envObjects.push({ x: x + doorW/2 + doorSideW/2, z: frontZ, w: doorSideW/2, d: wallThick/2 });

  // Floor (slightly above terrain, inset from walls)
  addPanel(new THREE.BoxGeometry(w - 0.6, 0.2, d - 0.6), x, groundY + 0.1, z, true);
  // floor gets no envObjects entry (player already snaps to terrain Y)

  // Roof detail
  const roofGeo = new THREE.BoxGeometry(w + 0.5, 0.4, d + 0.5);
  const roof = new THREE.Mesh(roofGeo, new THREE.MeshStandardMaterial({ color: 0x444444, roughness: 0.9, metalness: 0.1 }));
  roof.position.set(x, groundY + h + 0.2, z);
  roof.castShadow = true;
  scene.add(roof);

  // Interior ambient light — warm dim glow
  const intLight = new THREE.PointLight(0xfff5e0, 0.5, w * 1.2);
  intLight.position.set(x, groundY + h * 0.6, z);
  scene.add(intLight);
}
```

**Step 3: Open the game in Chrome and verify**

- Buildings should now be hollow — walk up to a building, find the door opening (1.4 units wide), and walk inside.
- The interior should have a slight warm glow distinct from outdoor sunlight.
- You should be blocked by wall panels but can walk through the door gap freely.
- No `TypeError` or `ReferenceError` in the DevTools console on page load.

**Step 4: Commit**

```bash
cd ~/Desktop/fortnite-game
git add index.html
git commit -m "feat: hollow buildings with door gap and interior ambient light"
```

---

## Task 2: Add one interior chest per building in `addBuilding()`

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html`

**Step 1: Add chest creation at the end of `addBuilding()`**

Inside the new `addBuilding()` function, find this comment at the very end (just before the closing `}`):
```javascript
  // Interior ambient light — warm dim glow
  const intLight = new THREE.PointLight(0xfff5e0, 0.5, w * 1.2);
  intLight.position.set(x, groundY + h * 0.6, z);
  scene.add(intLight);
}
```

Replace with:
```javascript
  // Interior ambient light — warm dim glow
  const intLight = new THREE.PointLight(0xfff5e0, 0.5, w * 1.2);
  intLight.position.set(x, groundY + h * 0.6, z);
  scene.add(intLight);

  // Interior chest — placed against back wall, centered
  const chestGroup = new THREE.Group();
  const chestGoldMat = new THREE.MeshStandardMaterial({ color: 0xf0c040, emissive: 0xf0c040, emissiveIntensity: 0.6, roughness: 0.4, metalness: 0.8 });
  const chestDarkMat = new THREE.MeshStandardMaterial({ color: 0x8B6914, roughness: 0.7, metalness: 0.3 });

  const cBody = new THREE.Mesh(new THREE.BoxGeometry(1.2, 0.8, 0.7), chestGoldMat);
  cBody.castShadow = true;
  chestGroup.add(cBody);

  const cLid = new THREE.Mesh(new THREE.BoxGeometry(1.2, 0.3, 0.7), chestDarkMat);
  cLid.position.y = 0.55;
  cLid.castShadow = true;
  chestGroup.add(cLid);

  const cLock = new THREE.Mesh(
    new THREE.BoxGeometry(0.15, 0.15, 0.05),
    new THREE.MeshStandardMaterial({ color: 0xffd700, roughness: 0.3, metalness: 0.9 })
  );
  cLock.position.set(0, 0.1, 0.38);
  chestGroup.add(cLock);

  // Glow light on chest
  const cLight = new THREE.PointLight(0xf0c040, 1.5, 8);
  cLight.position.set(0, 1.5, 0);
  chestGroup.add(cLight);

  // Position: back wall interior face, centered on wall width, floorY + 0.4
  const chestX = x;
  const chestZ = backZ + doorSign * 0.8;   // 0.8 units inward from back wall
  const chestY = groundY + 0.4;
  chestGroup.position.set(chestX, chestY, chestZ);
  scene.add(chestGroup);

  chests.push({ mesh: chestGroup, x: chestX, y: chestY, z: chestZ, open: false });
}
```

**Step 2: Open the game in Chrome and verify**

- Walk into a building. You should see a glowing gold chest against the back wall.
- Walk up to the chest and press E — it should open and give you loot (same `openChest()` logic as outdoor chests).
- There should be exactly one chest per building interior.
- No duplicate chests spawning in the same spot across game restarts (seeded placement is deterministic).
- `chests.length` in the DevTools console should equal `CHEST_COUNT` (30 outdoor) + number of buildings spawned.

**Step 3: Commit**

```bash
cd ~/Desktop/fortnite-game
git add index.html
git commit -m "feat: spawn one interior chest per building against back wall"
```

---

## Completion Checklist

1. Player can walk through a door gap (1.4 × 2.4 units) in the front wall of each building
2. Player is blocked by all 6 wall panel AABBs (left, right, back, front-top, front-left-pier, front-right-pier)
3. Interior has a warm dim PointLight (0xfff5e0) distinct from outdoor sunlight
4. Each building has exactly one glowing gold chest inside against the back wall
5. Interior chest can be opened with E and uses the same `openChest()` pickup flow as outdoor chests
6. No console errors on page load (`chests` is defined before `generateEnvironment()` runs)
7. Outdoor chests (30) are unchanged
