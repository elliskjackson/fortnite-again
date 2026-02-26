# Bot Visual Upgrade — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace the flat blue block-figure bots with articulated humanoid characters — cylinder limbs, box torso/head/hips, randomized outfit colors, a visible gun in the right hand, full walk/aim/crouch animations, and head tracking toward the player.

**Architecture:** All changes are in `~/Desktop/fortnite-game/index.html`. The existing `createBotMesh()` function is replaced with a new version that builds a pivot-group hierarchy and stores named references in `group.userData.parts`. `spawnBots()` captures those parts into `bot.parts`. The animation, hit-flash, and death code in `updateBots()` and `killBot()` switch from fragile child-index lookups to `bot.parts`.

**Tech Stack:** Three.js r128 (already loaded). No new CDN scripts required.

---

## Orientation — what exists today

In `~/Desktop/fortnite-game/index.html`:

```
line ~940  //  BOTS  section
line  946  const botBodyMat = new THREE.MeshLambertMaterial({ color: 0x2244aa });
line  947  const botHeadMat = new THREE.MeshLambertMaterial({ color: 0xffdead });
line  949  function createBotMesh() { ... }  // 6 flat children, index-addressed
line 1003  const mesh = createBotMesh();    // inside spawnBots()
line 1011  bots.push({ mesh, alive:true, ... });  // no parts field yet
line 1137  function updateBots(dt) { ... }
line 1144  bot.mesh.children.forEach(...)   // death opacity fade
line 1171  bot.mesh.children.forEach((c,idx)=>...)  // hit flash — uses idx===1 for head
line 1178  bot.mesh.position.set(...)
line 1182  bot.mesh.children[4].rotation.x = ...   // left leg
line 1183  bot.mesh.children[5].rotation.x = ...   // right leg
line 1260  function damageBot(bot, dmg) { ... }
line 1267  function killBot(bot) { bot.mesh.children.forEach(...) }
```

The raycasting call `ray.intersectObject(bot.mesh, true)` (line ~679) is **not** changed — it already uses recursive mode and will work with nested groups.

---

## Task 1: Rewrite `createBotMesh()` with articulated pivot hierarchy

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html`

**Step 1: Remove the two shared material constants and replace `createBotMesh()`**

Find and delete these two lines:
```javascript
const botBodyMat = new THREE.MeshLambertMaterial({ color: 0x2244aa });
const botHeadMat = new THREE.MeshLambertMaterial({ color: 0xffdead });
```

Then find the entire `createBotMesh()` function (from `function createBotMesh() {` through its closing `}`) and replace it with:

```javascript
function createBotMesh(outfitColor) {
  const group = new THREE.Group();

  // Materials — baseColor stored for hit-flash restore
  function mat(color, roughness, metalness) {
    const m = new THREE.MeshStandardMaterial({ color, roughness, metalness });
    m.userData.baseColor = color instanceof THREE.Color ? color.getHex() : color;
    return m;
  }
  const outfitMat = mat(outfitColor, 0.80, 0.10);
  const skinMat   = mat(0xffdead,   0.85, 0.00);
  const bootMat   = mat(0x222222,   0.90, 0.10);
  const gunMat    = mat(0x333333,   0.50, 0.60);

  // Helper: create a mesh, position it, enable shadow
  function mk(geo, material, px, py, pz) {
    const m = new THREE.Mesh(geo, material.clone());
    m.material.userData.baseColor = material.userData.baseColor;
    m.position.set(px, py, pz);
    m.castShadow = true;
    return m;
  }

  // ── Static torso parts ──────────────────────────────────────────
  const hips  = mk(new THREE.BoxGeometry(0.38, 0.22, 0.25), outfitMat, 0, 0.91, 0);
  const torso = mk(new THREE.BoxGeometry(0.55, 0.55, 0.28), outfitMat, 0, 1.30, 0);
  const neck  = mk(new THREE.CylinderGeometry(0.10, 0.10, 0.15, 8), skinMat, 0, 1.65, 0);
  group.add(hips, torso, neck);

  // ── Head pivot (rotates Y for head-turn) ────────────────────────
  const headPivot = new THREE.Group();
  headPivot.position.set(0, 1.90, 0);
  const head = mk(new THREE.BoxGeometry(0.38, 0.38, 0.38), skinMat, 0, 0, 0);
  headPivot.add(head);
  group.add(headPivot);

  // ── Arms ────────────────────────────────────────────────────────
  function makeArm(side) {
    const pivot = new THREE.Group();
    pivot.position.set(side * 0.37, 1.52, 0);
    const upper = mk(new THREE.CylinderGeometry(0.09, 0.09, 0.35, 8), outfitMat, 0, -0.175, 0);
    pivot.add(upper);
    const forearmPivot = new THREE.Group();
    forearmPivot.position.set(0, -0.35, 0);
    const fore = mk(new THREE.CylinderGeometry(0.075, 0.075, 0.30, 8), skinMat, 0, -0.15, 0);
    forearmPivot.add(fore);
    pivot.add(forearmPivot);
    return { pivot, forearmPivot };
  }
  const leftArm  = makeArm(-1);
  const rightArm = makeArm( 1);
  group.add(leftArm.pivot, rightArm.pivot);

  // Gun on right forearm
  const gun = mk(new THREE.BoxGeometry(0.08, 0.08, 0.35), gunMat, 0.05, -0.15, -0.22);
  rightArm.forearmPivot.add(gun);

  // ── Legs ────────────────────────────────────────────────────────
  function makeLeg(side) {
    const pivot = new THREE.Group();
    pivot.position.set(side * 0.12, 0.82, 0);
    const thigh = mk(new THREE.CylinderGeometry(0.11, 0.10, 0.38, 8), outfitMat, 0, -0.19, 0);
    pivot.add(thigh);
    const shinPivot = new THREE.Group();
    shinPivot.position.set(0, -0.38, 0);
    const shin = mk(new THREE.CylinderGeometry(0.09, 0.08, 0.36, 8), bootMat, 0, -0.18, 0);
    shinPivot.add(shin);
    pivot.add(shinPivot);
    return { pivot };
  }
  const leftLeg  = makeLeg(-1);
  const rightLeg = makeLeg( 1);
  group.add(leftLeg.pivot, rightLeg.pivot);

  // ── Collect all meshes (for flash / death) ──────────────────────
  const allMeshes = [];
  group.traverse(child => { if (child.isMesh) allMeshes.push(child); });

  group.userData.parts = {
    headPivot,
    leftArmPivot:  leftArm.pivot,
    rightArmPivot: rightArm.pivot,
    leftLegPivot:  leftLeg.pivot,
    rightLegPivot: rightLeg.pivot,
    allMeshes
  };

  return group;
}
```

**Step 2: Verify the function exists and has no syntax errors**

Open `index.html` in Chrome DevTools. There should be no "Unexpected token" or "SyntaxError" in the console when the page loads (the game won't work yet because `spawnBots` still calls `createBotMesh()` without a color argument — that's fixed in Task 2).

**Step 3: Commit**

```bash
cd ~/Desktop/fortnite-game
git add index.html
git commit -m "feat: rewrite createBotMesh() with articulated cylinder-limb pivot hierarchy"
```

---

## Task 2: Update `spawnBots()` to pass outfit color and store `bot.parts`

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html`

**Step 1: Add outfit color generation and pass it to `createBotMesh()`**

Inside `spawnBots()`, find:
```javascript
    const mesh = createBotMesh();
```

Replace with:
```javascript
    const outfitHue   = ((i * 137.508) % 360) / 360;
    const outfitColor = new THREE.Color().setHSL(outfitHue, 0.65, 0.45);
    const mesh = createBotMesh(outfitColor);
```

**Step 2: Add `parts` to the bot object**

Find:
```javascript
    bots.push({
      mesh, alive: true,
```

Replace with:
```javascript
    bots.push({
      mesh, parts: mesh.userData.parts, alive: true,
```

**Step 3: Verify bots appear in game**

Open the game in Chrome. Start a match. You should see articulated humanoid bots with distinct outfit colors — cylinder arms and legs, box torso and head, a small gun stub visible on their right side.

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: seed outfit colors per bot, store bot.parts from mesh hierarchy"
```

---

## Task 3: Rewrite bot animation (walk cycle, arm swing, head turn, crouch)

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html`

**Step 1: Replace the position/animation block inside `updateBots()`**

Find this block (near the bottom of the per-bot loop in `updateBots`):
```javascript
    const groundY = getTerrainY(bot.x, bot.z);
    bot.y = groundY;
    bot.mesh.position.set(bot.x, bot.y, bot.z);
    bot.mesh.rotation.y = bot.yaw;
    if (bot.mesh.children[4]) bot.mesh.children[4].rotation.x =  Math.sin(bot.walkPhase) * 0.4;
    if (bot.mesh.children[5]) bot.mesh.children[5].rotation.x = -Math.sin(bot.walkPhase) * 0.4;
    bot.walkPhase += dt * 4;
```

Replace with:
```javascript
    const groundY = getTerrainY(bot.x, bot.z);
    bot.y = groundY;

    // Walk cycle speed
    const isMoving = bot.state === 'chase' || bot.state === 'flee' || bot.state === 'patrol';
    bot.walkPhase += dt * (isMoving ? 4.5 : 1.5);

    const p = bot.parts;

    if (bot.state === 'attack') {
      // Crouch + raise arms to aim
      bot.mesh.position.set(bot.x, bot.y - 0.3, bot.z);
      p.leftArmPivot.rotation.x  = -0.9;
      p.rightArmPivot.rotation.x = -0.9;
      p.leftLegPivot.rotation.x  = 0;
      p.rightLegPivot.rotation.x = 0;
    } else {
      bot.mesh.position.set(bot.x, bot.y, bot.z);
      const legAmp = bot.state === 'flee' ? 0.65 : (isMoving ? 0.55 : 0.10);
      const armAmp = bot.state === 'flee' ? 0.40 : (isMoving ? 0.35 : 0.06);
      p.leftLegPivot.rotation.x  = -Math.sin(bot.walkPhase) * legAmp;
      p.rightLegPivot.rotation.x =  Math.sin(bot.walkPhase) * legAmp;
      p.leftArmPivot.rotation.x  =  Math.sin(bot.walkPhase) * armAmp;
      p.rightArmPivot.rotation.x = -Math.sin(bot.walkPhase) * armAmp;
    }

    // Head turn toward player (smooth lerp, clamped ±60°)
    const pp2 = camera.position;
    const angleToPlayer = Math.atan2(pp2.x - bot.x, pp2.z - bot.z);
    const relAngle = ((angleToPlayer - bot.yaw + Math.PI) % (Math.PI * 2)) - Math.PI;
    const targetHeadY = Math.max(-Math.PI / 3, Math.min(Math.PI / 3, relAngle));
    p.headPivot.rotation.y += (targetHeadY - p.headPivot.rotation.y) * Math.min(1, dt * 5);

    bot.mesh.rotation.y = bot.yaw;
```

**Step 2: Verify animations in-game**

- Walk toward a bot — it should swing arms and legs with opposite phase.
- Let a bot spot you — it should crouch and raise both arms into an aiming stance.
- Move around a bot — its head should smoothly track you.
- Run from a bot in flee state — legs swing wider and faster.

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add bot walk cycle, arm swing, head tracking, and attack crouch animation"
```

---

## Task 4: Fix hit-flash and death to use `bot.parts.allMeshes`

**Files:**
- Modify: `~/Desktop/fortnite-game/index.html`

**Step 1: Fix the hit-flash block in `updateBots()`**

Find:
```javascript
    if (bot.hitFlash > 0) {
      bot.hitFlash -= dt;
      const flashOn = bot.hitFlash > 0;
      bot.mesh.children.forEach((c, idx) => {
        if (c.material) {
          c.material.color.set(flashOn ? 0xff8800 : (idx === 1 ? 0xffdead : 0x2244aa));
        }
      });
    }
```

Replace with:
```javascript
    if (bot.hitFlash > 0) {
      bot.hitFlash -= dt;
      const flashOn = bot.hitFlash > 0;
      bot.parts.allMeshes.forEach(m => {
        m.material.color.set(flashOn ? 0xff8800 : m.material.userData.baseColor);
      });
    }
```

**Step 2: Fix the death opacity fade in `updateBots()`**

Find:
```javascript
      if (bot.deathTimer > 0) {
        bot.deathTimer -= dt;
        bot.mesh.position.y -= dt * 0.5;
        bot.mesh.children.forEach(c => {
          if (c.material) c.material.opacity = Math.max(0, bot.deathTimer / 2);
        });
        if (bot.deathTimer <= 0) bot.mesh.visible = false;
      }
```

Replace with:
```javascript
      if (bot.deathTimer > 0) {
        bot.deathTimer -= dt;
        bot.mesh.position.y -= dt * 0.5;
        bot.parts.allMeshes.forEach(m => {
          m.material.opacity = Math.max(0, bot.deathTimer / 2);
        });
        if (bot.deathTimer <= 0) bot.mesh.visible = false;
      }
```

**Step 3: Fix `killBot()` to use `bot.parts.allMeshes`**

Find:
```javascript
function killBot(bot) {
  bot.alive = false;
  bot.deathTimer = 2;
  bot.mesh.children.forEach(c => {
    if (c.material) {
      c.material = c.material.clone();
      c.material.transparent = true;
    }
  });
  state.kills++;
  checkWinCondition();
}
```

Replace with:
```javascript
function killBot(bot) {
  bot.alive = false;
  bot.deathTimer = 2;
  bot.parts.allMeshes.forEach(m => {
    m.material = m.material.clone();
    m.material.transparent = true;
  });
  state.kills++;
  checkWinCondition();
}
```

**Step 4: Verify hit-flash and death in-game**

- Shoot a bot — all its parts should flash orange briefly.
- Kill a bot — it should sink into the ground and fade out.
- No console errors about `bot.parts` being undefined.

**Step 5: Commit**

```bash
git add index.html
git commit -m "feat: update hit-flash and death animations to use bot.parts.allMeshes"
```

---

## Completion checklist

1. Bots have distinct randomized outfit colors (not all blue)
2. Arms and legs are cylinders; torso, hips, head are boxes
3. A small gun is visible on the right forearm
4. Walking bots swing arms and legs in opposite phase
5. Bots in attack state crouch and raise arms into an aim pose
6. Bot heads smoothly track toward the player
7. Fleeing bots have wider/faster leg swing
8. Hit flash turns all parts orange then restores correct per-part colors
9. Dead bots sink and fade out correctly
