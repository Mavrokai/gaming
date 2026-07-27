# Shade Rift Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construire *Shade Rift*, un jeu d’action ThreeJS top-down PvE (esprit LoL) avec menu, 3 héros, click-to-move, sorts A Z E R, HUD type LoL, 3 niveaux + boss, score/or/pickups.

**Architecture:** Vite + ES modules. `SceneManager` orchestre des scènes ; la scène `Game` possède la boucle ThreeJS et délègue à des systems purs (`ClickMove`, `Combat`, `WaveSpawner`, `AI`, `Collision`). UI menus/HUD en DOM overlay. Données héros/ennemis/niveaux dans `src/data/`.

**Tech Stack:** HTML/CSS/JS, Vite, Three.js, Vitest (logique pure), localStorage (options)

**Spec :** `docs/superpowers/specs/2026-07-27-html5-gaming-design.md`

---

## File map

| Path | Responsibility |
|---|---|
| `package.json` | deps + scripts `dev` / `build` / `test` |
| `index.html` | shell : canvas + root UI |
| `src/main.js` | bootstrap Game |
| `src/core/Game.js` | owns renderer loop, SceneManager, Input, Settings, Audio |
| `src/core/SceneManager.js` | push/pop/replace scenes |
| `src/core/Input.js` | keys, mouse NDC, rebind map |
| `src/core/Settings.js` | load/save options |
| `src/core/Audio.js` | BGM/SFX volumes |
| `src/data/heroes.js` | 3 héros + sorts |
| `src/data/enemies.js` | types ennemis + boss |
| `src/data/levels.js` | 3 niveaux / vagues |
| `src/systems/ClickMove.js` | cible + intégration position |
| `src/systems/Combat.js` | CD, mana, cast, dégâts |
| `src/systems/Collision.js` | sphères / AABB |
| `src/systems/WaveSpawner.js` | état vagues |
| `src/systems/AI.js` | chase/attack |
| `src/systems/CameraFollow.js` | lerp caméra |
| `src/entities/*` | Hero, Enemy, Boss, Projectile, Pickup, Arena |
| `src/scenes/*` | Menu, Rules, Options, HeroSelect, Game, Pause overlays |
| `src/ui/Hud.js` + `src/ui/hud.css` | barre LoL |
| `src/styles/main.css` | thème dark global |
| `tests/**/*.test.js` | Vitest pure logic |
| `assets/hud/*` | portraits + sorts (déjà présents) |
| `README.md` | lancer, contrôles, règles |

---

### Task 1: Scaffold Vite + Three.js + Vitest

**Files:**
- Create: `package.json`, `vite.config.js`, `index.html`, `src/main.js`, `src/styles/main.css`, `README.md`
- Keep: `.gitignore`, `assets/hud/*`

- [ ] **Step 1: Create `package.json`**

```json
{
  "name": "shade-rift",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "dependencies": {
    "three": "^0.170.0"
  },
  "devDependencies": {
    "vite": "^6.0.0",
    "vitest": "^2.1.0"
  }
}
```

- [ ] **Step 2: Create `vite.config.js`**

```js
import { defineConfig } from 'vite';

export default defineConfig({
  root: '.',
  publicDir: 'assets',
  test: {
    environment: 'node',
  },
});
```

- [ ] **Step 3: Create `index.html`**

```html
<!DOCTYPE html>
<html lang="fr">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Shade Rift</title>
    <link rel="stylesheet" href="/src/styles/main.css" />
  </head>
  <body>
    <div id="app">
      <canvas id="game-canvas"></canvas>
      <div id="ui-root"></div>
    </div>
    <script type="module" src="/src/main.js"></script>
  </body>
</html>
```

- [ ] **Step 4: Create minimal `src/styles/main.css` et `src/main.js`**

```css
/* src/styles/main.css */
:root {
  --bg: #0c0a09;
  --gold: #c4a574;
  --text: #e7e5e4;
  --danger: #f87171;
  --hp: #22c55e;
  --mana: #3b82f6;
}
* { box-sizing: border-box; margin: 0; padding: 0; }
html, body, #app { width: 100%; height: 100%; overflow: hidden; background: var(--bg); color: var(--text); font-family: Georgia, 'Times New Roman', serif; }
#game-canvas { display: block; width: 100%; height: 100%; }
#ui-root { position: absolute; inset: 0; pointer-events: none; }
#ui-root .panel { pointer-events: auto; }
```

```js
// src/main.js
console.log('Shade Rift boot');
```

- [ ] **Step 5: Install and verify**

Run: `npm install && npm test`  
Expected: Vitest exits 0 (0 tests) or “No test files found” according to vitest version — acceptable.  
Run: `npm run dev` — page loads, console “Shade Rift boot”.

- [ ] **Step 6: Commit**

```bash
git add package.json package-lock.json vite.config.js index.html src/main.js src/styles/main.css README.md
git commit -m "chore: scaffold Vite Three.js Vitest for Shade Rift"
```

---

### Task 2: Settings + Input (pure)

**Files:**
- Create: `src/core/Settings.js`, `src/core/Input.js`
- Test: `tests/core/Settings.test.js`, `tests/core/Input.test.js`

- [ ] **Step 1: Write failing Settings tests**

```js
// tests/core/Settings.test.js
import { describe, it, expect, beforeEach } from 'vitest';
import { Settings } from '../../src/core/Settings.js';

describe('Settings', () => {
  beforeEach(() => {
    globalThis.localStorage = {
      store: {},
      getItem(k) { return this.store[k] ?? null; },
      setItem(k, v) { this.store[k] = String(v); },
      removeItem(k) { delete this.store[k]; },
    };
  });

  it('loads defaults then persists rebinds', () => {
    const s = new Settings();
    expect(s.spellKeys).toEqual({ A: 'KeyA', Z: 'KeyZ', E: 'KeyE', R: 'KeyR' });
    s.setSpellKey('A', 'KeyQ');
    s.save();
    const s2 = new Settings();
    expect(s2.spellKeys.A).toBe('KeyQ');
  });
});
```

- [ ] **Step 2: Run test — expect FAIL**

Run: `npm test -- tests/core/Settings.test.js`  
Expected: FAIL (module not found)

- [ ] **Step 3: Implement Settings**

```js
// src/core/Settings.js
const STORAGE_KEY = 'shade-rift-settings';

const DEFAULTS = {
  musicVolume: 0.6,
  sfxVolume: 0.8,
  showDamage: true,
  shadows: true,
  spellKeys: { A: 'KeyA', Z: 'KeyZ', E: 'KeyE', R: 'KeyR' },
  utilKeys: { D: 'KeyD', F: 'KeyF' },
};

export class Settings {
  constructor() {
    this._data = structuredClone(DEFAULTS);
    this.load();
  }

  get musicVolume() { return this._data.musicVolume; }
  get sfxVolume() { return this._data.sfxVolume; }
  get showDamage() { return this._data.showDamage; }
  get shadows() { return this._data.shadows; }
  get spellKeys() { return this._data.spellKeys; }
  get utilKeys() { return this._data.utilKeys; }

  setMusicVolume(v) { this._data.musicVolume = clamp01(v); }
  setSfxVolume(v) { this._data.sfxVolume = clamp01(v); }
  setShowDamage(v) { this._data.showDamage = !!v; }
  setShadows(v) { this._data.shadows = !!v; }
  setSpellKey(slot, code) { this._data.spellKeys[slot] = code; }

  load() {
    try {
      const raw = localStorage.getItem(STORAGE_KEY);
      if (!raw) return;
      const parsed = JSON.parse(raw);
      this._data = { ...structuredClone(DEFAULTS), ...parsed, spellKeys: { ...DEFAULTS.spellKeys, ...(parsed.spellKeys || {}) } };
    } catch {
      this._data = structuredClone(DEFAULTS);
    }
  }

  save() {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(this._data));
  }
}

function clamp01(v) {
  return Math.min(1, Math.max(0, Number(v) || 0));
}
```

- [ ] **Step 4: Implement Input + test**

```js
// tests/core/Input.test.js
import { describe, it, expect } from 'vitest';
import { Input } from '../../src/core/Input.js';

describe('Input', () => {
  it('tracks key down/up and consumes just-pressed', () => {
    const input = new Input();
    input.onKeyDown({ code: 'KeyA' });
    expect(input.isDown('KeyA')).toBe(true);
    expect(input.consumePressed('KeyA')).toBe(true);
    expect(input.consumePressed('KeyA')).toBe(false);
    input.onKeyUp({ code: 'KeyA' });
    expect(input.isDown('KeyA')).toBe(false);
  });
});
```

```js
// src/core/Input.js
export class Input {
  constructor() {
    this._down = new Set();
    this._pressed = new Set();
    this.mouse = { x: 0, y: 0, ndcX: 0, ndcY: 0, leftDown: false, leftClicked: false };
  }

  attach(dom, canvas) {
    this._canvas = canvas;
    dom.addEventListener('keydown', (e) => this.onKeyDown(e));
    dom.addEventListener('keyup', (e) => this.onKeyUp(e));
    canvas.addEventListener('mousemove', (e) => this.onMouseMove(e));
    canvas.addEventListener('mousedown', (e) => this.onMouseDown(e));
    canvas.addEventListener('mouseup', (e) => this.onMouseUp(e));
  }

  onKeyDown(e) {
    if (!this._down.has(e.code)) this._pressed.add(e.code);
    this._down.add(e.code);
  }

  onKeyUp(e) {
    this._down.delete(e.code);
  }

  isDown(code) { return this._down.has(code); }

  consumePressed(code) {
    if (!this._pressed.has(code)) return false;
    this._pressed.delete(code);
    return true;
  }

  onMouseMove(e) {
    const rect = this._canvas.getBoundingClientRect();
    this.mouse.x = e.clientX - rect.left;
    this.mouse.y = e.clientY - rect.top;
    this.mouse.ndcX = (this.mouse.x / rect.width) * 2 - 1;
    this.mouse.ndcY = -(this.mouse.y / rect.height) * 2 + 1;
  }

  onMouseDown(e) {
    if (e.button === 0) {
      this.mouse.leftDown = true;
      this.mouse.leftClicked = true;
    }
  }

  onMouseUp(e) {
    if (e.button === 0) this.mouse.leftDown = false;
  }

  endFrame() {
    this._pressed.clear();
    this.mouse.leftClicked = false;
  }
}
```

- [ ] **Step 5: Run tests — expect PASS**

Run: `npm test`  
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add src/core/Settings.js src/core/Input.js tests/core/
git commit -m "feat: add Settings persistence and Input tracking"
```

---

### Task 3: SceneManager + Game shell

**Files:**
- Create: `src/core/SceneManager.js`, `src/core/Game.js`, `src/scenes/BootScene.js`, `src/scenes/MenuScene.js`
- Modify: `src/main.js`

- [ ] **Step 1: Implement SceneManager**

```js
// src/core/SceneManager.js
export class SceneManager {
  constructor(game) {
    this.game = game;
    this._stack = [];
  }

  get current() { return this._stack[this._stack.length - 1] || null; }

  async replace(scene) {
    while (this._stack.length) await this.pop();
    await this.push(scene);
  }

  async push(scene) {
    scene.game = this.game;
    if (scene.enter) await scene.enter();
    this._stack.push(scene);
  }

  async pop() {
    const scene = this._stack.pop();
    if (scene?.exit) await scene.exit();
    return scene;
  }

  update(dt) { this.current?.update?.(dt); }
  render() { this.current?.render?.(); }
}
```

- [ ] **Step 2: Implement Game shell (Three.js renderer ready, dark clear color)**

```js
// src/core/Game.js
import * as THREE from 'three';
import { SceneManager } from './SceneManager.js';
import { Input } from './Input.js';
import { Settings } from './Settings.js';

export class Game {
  constructor({ canvas, uiRoot }) {
    this.canvas = canvas;
    this.uiRoot = uiRoot;
    this.settings = new Settings();
    this.input = new Input();
    this.scenes = new SceneManager(this);
    this.clock = new THREE.Clock();
    this.renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    this.renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    this.renderer.setClearColor(0x0c0a09);
    this.resize();
    this.input.attach(window, canvas);
    window.addEventListener('resize', () => this.resize());
  }

  resize() {
    const w = window.innerWidth;
    const h = window.innerHeight;
    this.renderer.setSize(w, h, false);
    this.currentThreeCamera?.aspect && (this.currentThreeCamera.aspect = w / h);
    this.currentThreeCamera?.updateProjectionMatrix?.();
  }

  async start(firstScene) {
    await this.scenes.replace(firstScene);
    const loop = () => {
      requestAnimationFrame(loop);
      const dt = Math.min(this.clock.getDelta(), 0.05);
      this.scenes.update(dt);
      this.scenes.render();
      this.input.endFrame();
    };
    loop();
  }
}
```

- [ ] **Step 3: Boot → Menu stub**

```js
// src/scenes/BootScene.js
import { MenuScene } from './MenuScene.js';

export class BootScene {
  async enter() {
    await this.game.scenes.replace(new MenuScene());
  }
}
```

```js
// src/scenes/MenuScene.js
export class MenuScene {
  enter() {
    this.game.uiRoot.innerHTML = `
      <div class="panel menu">
        <h1>Shade Rift</h1>
        <p class="tag">HTML5 · ThreeJS</p>
        <button data-action="play">Jouer</button>
        <button data-action="options">Options</button>
        <button data-action="rules">Règles</button>
      </div>`;
    this.game.uiRoot.querySelector('.menu').addEventListener('click', (e) => {
      const action = e.target.dataset?.action;
      if (action === 'play') this.onPlay?.();
      if (action === 'options') this.onOptions?.();
      if (action === 'rules') this.onRules?.();
    });
  }

  exit() { this.game.uiRoot.innerHTML = ''; }
  update() {}
  render() { this.game.renderer.clear(); }
}
```

Wire `main.js`:

```js
import { Game } from './core/Game.js';
import { BootScene } from './scenes/BootScene.js';

const game = new Game({
  canvas: document.getElementById('game-canvas'),
  uiRoot: document.getElementById('ui-root'),
});
game.start(new BootScene());
```

Add menu CSS in `main.css` (centré, boutons gold/dark).

- [ ] **Step 4: Manual check**

Run: `npm run dev`  
Expected: titre Shade Rift + 3 boutons ; fond sombre.

- [ ] **Step 5: Commit**

```bash
git add src/core/Game.js src/core/SceneManager.js src/scenes/BootScene.js src/scenes/MenuScene.js src/main.js src/styles/main.css
git commit -m "feat: add SceneManager game shell and main menu"
```

---

### Task 4: Data — heroes, enemies, levels

**Files:**
- Create: `src/data/heroes.js`, `src/data/enemies.js`, `src/data/levels.js`
- Test: `tests/data/dataShape.test.js`

- [ ] **Step 1: Write shape test**

```js
// tests/data/dataShape.test.js
import { describe, it, expect } from 'vitest';
import { HEROES } from '../../src/data/heroes.js';
import { ENEMIES, BOSSES } from '../../src/data/enemies.js';
import { LEVELS } from '../../src/data/levels.js';

describe('data', () => {
  it('has 3 heroes with AZER spells', () => {
    expect(HEROES).toHaveLength(3);
    for (const h of HEROES) {
      expect(h.spells.A).toBeTruthy();
      expect(h.spells.Z).toBeTruthy();
      expect(h.spells.E).toBeTruthy();
      expect(h.spells.R).toBeTruthy();
      expect(h.portrait).toMatch(/^\/hud\//);
    }
  });

  it('has 3 levels with waves and boss', () => {
    expect(LEVELS).toHaveLength(3);
    for (const l of LEVELS) {
      expect(l.waves.length).toBeGreaterThan(0);
      expect(l.bossId).toBeTruthy();
      expect(BOSSES[l.bossId]).toBeTruthy();
    }
  });

  it('defines enemy archetypes', () => {
    expect(ENEMIES.rusher).toBeTruthy();
    expect(ENEMIES.ranged).toBeTruthy();
    expect(ENEMIES.tank).toBeTruthy();
  });
});
```

- [ ] **Step 2: Implement data files**

`heroes.js` — 3 entries: `shade-bruiser`, `shade-mage`, `shade-assassin` with stats `{ maxHp, maxMana, moveSpeed, ad, ap, armor, msDisplay }` and spells `{ type, damage, manaCost, cooldown, range, radius?, duration? }` + icon paths under `/hud/spell-*.png` (reuse mockup icons for v1; portraits `/hud/hud-portrait.png` for all until unique art).

`enemies.js` — `ENEMIES` map + `BOSSES` with `phases: [{ hpRatio, pattern }]`.

`levels.js` — arena size, fog color, waves arrays of `{ enemyId, count, delay }`, `bossId`.

Spell type enums used later by Combat: `projectile`, `aoe`, `dash`, `shield`, `root`, `slow`.

- [ ] **Step 3: `npm test` — PASS**

- [ ] **Step 4: Commit**

```bash
git add src/data/ tests/data/
git commit -m "feat: add heroes enemies levels data definitions"
```

---

### Task 5: ClickMove + Collision (pure logic)

**Files:**
- Create: `src/systems/ClickMove.js`, `src/systems/Collision.js`
- Test: `tests/systems/ClickMove.test.js`, `tests/systems/Collision.test.js`

- [ ] **Step 1: Failing tests**

```js
// tests/systems/ClickMove.test.js
import { describe, it, expect } from 'vitest';
import { ClickMove } from '../../src/systems/ClickMove.js';

describe('ClickMove', () => {
  it('moves toward target and stops within epsilon', () => {
    const cm = new ClickMove({ speed: 5, stopDistance: 0.3 });
    const pos = { x: 0, z: 0 };
    cm.setTarget(10, 0);
    cm.update(pos, 0.2); // move 1 unit
    expect(pos.x).toBeCloseTo(1, 5);
    // drain remaining
    for (let i = 0; i < 100; i++) cm.update(pos, 0.2);
    expect(Math.hypot(pos.x - 10, pos.z - 0)).toBeLessThanOrEqual(0.3);
    expect(cm.hasTarget).toBe(false);
  });
});
```

```js
// tests/systems/Collision.test.js
import { describe, it, expect } from 'vitest';
import { sphereOverlap, pointInAabb2D } from '../../src/systems/Collision.js';

describe('Collision', () => {
  it('detects sphere overlap', () => {
    expect(sphereOverlap({ x: 0, y: 0, z: 0, r: 1 }, { x: 1.5, y: 0, z: 0, r: 1 })).toBe(true);
    expect(sphereOverlap({ x: 0, y: 0, z: 0, r: 1 }, { x: 3, y: 0, z: 0, r: 1 })).toBe(false);
  });

  it('clamps point inside arena aabb', () => {
    expect(pointInAabb2D(0, 0, { minX: -5, maxX: 5, minZ: -5, maxZ: 5 })).toBe(true);
    expect(pointInAabb2D(9, 0, { minX: -5, maxX: 5, minZ: -5, maxZ: 5 })).toBe(false);
  });
});
```

- [ ] **Step 2: Implement**

```js
// src/systems/ClickMove.js
export class ClickMove {
  constructor({ speed = 6, stopDistance = 0.3 } = {}) {
    this.speed = speed;
    this.stopDistance = stopDistance;
    this.target = null;
  }

  get hasTarget() { return this.target != null; }

  setTarget(x, z) { this.target = { x, z }; }
  clearTarget() { this.target = null; }

  update(pos, dt) {
    if (!this.target) return;
    const dx = this.target.x - pos.x;
    const dz = this.target.z - pos.z;
    const dist = Math.hypot(dx, dz);
    if (dist <= this.stopDistance) {
      this.target = null;
      return;
    }
    const step = Math.min(this.speed * dt, dist);
    pos.x += (dx / dist) * step;
    pos.z += (dz / dist) * step;
  }
}
```

```js
// src/systems/Collision.js
export function sphereOverlap(a, b) {
  const dx = a.x - b.x;
  const dy = (a.y ?? 0) - (b.y ?? 0);
  const dz = a.z - b.z;
  const r = a.r + b.r;
  return dx * dx + dy * dy + dz * dz <= r * r;
}

export function pointInAabb2D(x, z, box) {
  return x >= box.minX && x <= box.maxX && z >= box.minZ && z <= box.maxZ;
}

export function clampPointToAabb2D(pos, box) {
  pos.x = Math.min(box.maxX, Math.max(box.minX, pos.x));
  pos.z = Math.min(box.maxZ, Math.max(box.minZ, pos.z));
}
```

- [ ] **Step 3: `npm test` — PASS · Commit**

```bash
git add src/systems/ClickMove.js src/systems/Collision.js tests/systems/
git commit -m "feat: add ClickMove and collision helpers"
```

---

### Task 6: Combat system (cooldowns, mana, damage)

**Files:**
- Create: `src/systems/Combat.js`
- Test: `tests/systems/Combat.test.js`

- [ ] **Step 1: Tests**

```js
import { describe, it, expect } from 'vitest';
import { CombatState } from '../../src/systems/Combat.js';

const spellA = { manaCost: 30, cooldown: 1, damage: 20, type: 'projectile', range: 12 };

describe('CombatState', () => {
  it('rejects cast without mana or during cooldown', () => {
    const c = new CombatState({ maxMana: 50, spells: { A: spellA, Z: spellA, E: spellA, R: spellA } });
    expect(c.tryCast('A', 0).ok).toBe(true);
    expect(c.mana).toBe(20);
    expect(c.tryCast('A', 0.1).ok).toBe(false); // CD
    c.update(1);
    expect(c.tryCast('A', 1.1).ok).toBe(false); // mana
  });

  it('applies damage with simple armor mitigation', () => {
    expect(CombatState.applyDamage(100, 20)).toBe(80);
    expect(CombatState.applyDamage(10, 50)).toBe(1); // min 1 if hit
  });
});
```

- [ ] **Step 2: Implement `CombatState`**

- Track `mana`, `maxMana`, `cooldowns` map slots → remaining time  
- `tryCast(slot, now)` returns `{ ok, spell }` and spends mana / starts CD  
- `update(dt)` ticks CDs and optional mana regen (e.g. +5/s)  
- static `applyDamage(raw, armor)` → `Math.max(1, raw - armor * 0.5)` (document formula in comment)

- [ ] **Step 3: Tests PASS · Commit**

```bash
git commit -am "feat: add CombatState for mana cooldowns and damage"
```

---

### Task 7: WaveSpawner + AI (pure)

**Files:**
- Create: `src/systems/WaveSpawner.js`, `src/systems/AI.js`
- Test: `tests/systems/WaveSpawner.test.js`, `tests/systems/AI.test.js`

- [ ] **Step 1: WaveSpawner**

API:
- `constructor(waves)` where waves = `[{ enemyId, count, delay }]`
- `update(dt, aliveCount)` → emits spawn events `{ enemyId }[]` when delay elapsed and previous wave cleared (`aliveCount === 0` after first spawn started)
- states: `idle | spawning | waitingClear | boss | done`
- `startBoss()` when all waves cleared

Tests: after delays and clears, reaches `done` only after boss flag set by caller.

- [ ] **Step 2: AI**

```js
// Desired API
export function updateChaser(self, target, dt, { speed, range, attackCooldown }) {
  // if dist > range: move toward target
  // else: if cooldown ready return { attack: true }
  // else return { attack: false }
}
```

Test: moves closer when far; returns attack when in range and CD ready.

- [ ] **Step 3: Commit**

```bash
git commit -am "feat: add WaveSpawner and chaser AI helpers"
```

---

### Task 8: Game scene — arena, hero, click-to-move wired

**Files:**
- Create: `src/entities/Arena.js`, `src/entities/Hero.js`, `src/systems/CameraFollow.js`, `src/scenes/GameScene.js`, `src/scenes/HeroSelectScene.js`
- Modify: `MenuScene.js` to navigate HeroSelect → Game

- [ ] **Step 1: Arena** — `THREE.Group` with ground plane (dark), wall boxes, fog on scene, bounds AABB exported.

- [ ] **Step 2: Hero** — mesh capsule/box color by role, holds `ClickMove`, `CombatState`, `hp`, position on XZ.

- [ ] **Step 3: GameScene.enter(levelId, heroId)`**
  - Build THREE.Scene + PerspectiveCamera (top-down tilt ~55°, y=18)
  - Raycaster from mouse NDC onto ground plane y=0 on left click → `hero.clickMove.setTarget`
  - Update ClickMove + clamp to arena
  - CameraFollow lerp
  - `game.currentThreeCamera = camera` for resize
  - `renderer.render(scene, camera)` in `render()`

- [ ] **Step 4: HeroSelectScene** — 3 cards from `HEROES`, on pick → `GameScene`.

- [ ] **Step 5: Manual check** — clic sol déplace le héros ; caméra suit.

- [ ] **Step 6: Commit**

```bash
git commit -am "feat: wire arena hero click-to-move in GameScene"
```

---

### Task 9: Projectiles, spells cast, enemies, collisions

**Files:**
- Create: `src/entities/Projectile.js`, `src/entities/Enemy.js`, `src/entities/Pickup.js`
- Modify: `GameScene.js`, spell cast from Input mapped via Settings.spellKeys

- [ ] **Step 1: On spell key pressed** — compute ground aim point via raycast ; `combat.tryCast` ; spawn projectile / aoe / dash / root according to `spell.type`.

- [ ] **Step 2: Projectile update** — move along direction, `sphereOverlap` vs enemies, apply damage, despawn on hit or maxRange.

- [ ] **Step 3: Enemy** — mesh, hp, `updateChaser`, attack damages hero ; on death add gold/score and maybe spawn pickup.

- [ ] **Step 4: WaveSpawner integration** — spawn enemies at arena edges.

- [ ] **Step 5: Manual** — A tire, ennemis meurent, HP baisse si touché.

- [ ] **Step 6: Commit**

```bash
git commit -am "feat: add spells projectiles enemies and wave combat"
```

---

### Task 10: Boss phases + 3 levels progression

**Files:**
- Create: `src/entities/Boss.js`
- Modify: `GameScene.js`, `levels.js` tuning

- [ ] **Step 1: Boss** — larger mesh, phase index from `hp/maxHp` thresholds ; patterns: `slam` (telegraph circle then AoE), `volley` (3 projectiles), `summon` (2 rushers).

- [ ] **Step 2: Telegraph** — temporary ground decal (semi-transparent mesh) for 0.7s before damage.

- [ ] **Step 3: Level clear** — when boss dead → if next level exists, short LevelClear UI then new GameScene ; else VictoryScene.

- [ ] **Step 4: Hero death** → GameOverScene (score, recommencer, menu).

- [ ] **Step 5: Manual** — finir niveau 1 boss → niveau 2.

- [ ] **Step 6: Commit**

```bash
git commit -am "feat: add boss phases and three-level progression"
```

---

### Task 11: HUD LoL (DOM)

**Files:**
- Create: `src/ui/Hud.js`, `src/ui/hud.css`
- Modify: `GameScene.js` to mount/update HUD each frame (throttled ok)

- [ ] **Step 1: Hud.mount(uiRoot)** builds structure matching validated mockup:
  - stats AD/AP/Armor/MS
  - portrait + level badge
  - spell buttons A Z E R with `<img>`, mana cost, CD overlay
  - HP / Mana bars with text
  - 6 inventory slots + gold
  - top-right: score + wave label

- [ ] **Step 2: Hud.update(snapshot)** where snapshot = `{ hero, combat, gold, score, wave, level, inventory }`

- [ ] **Step 3: Style** — gold borders, dark gradient, Georgia font ; pointer-events none except if needed.

- [ ] **Step 4: Manual** — CD darkens Z, numbers match.

- [ ] **Step 5: Commit**

```bash
git commit -am "feat: add League-like bottom HUD with spell icons"
```

---

### Task 12: Options, Rules, Pause, Audio stub

**Files:**
- Create: `src/scenes/OptionsScene.js`, `src/scenes/RulesScene.js`, `src/scenes/PauseOverlay.js`, `src/scenes/GameOverScene.js`, `src/scenes/VictoryScene.js`, `src/core/Audio.js`
- Modify: MenuScene, GameScene (Échap → pause)

- [ ] **Step 1: RulesScene** — texte FR : objectif (3 niveaux), contrôles clic + A Z E R, HP/Mana, boss.

- [ ] **Step 2: OptionsScene** — sliders volumes, checkboxes, rebind: clic sur sort puis touche ; `settings.save()`.

- [ ] **Step 3: PauseOverlay** — push overlay scene that freezes GameScene updates when `paused` flag set (GameScene skips gameplay update if `this.paused`).

- [ ] **Step 4: Audio** — HTMLAudioElement wrappers ; play SFX no-op if file missing ; volumes from Settings. Place optional files in `assets/audio/` later.

- [ ] **Step 5: Manual** — options persistent after reload ; pause works.

- [ ] **Step 6: Commit**

```bash
git commit -am "feat: add rules options pause and audio stub"
```

---

### Task 13: Polish visuals + README + balance pass

**Files:**
- Modify: arena materials, lights, fog per level, enemy colors, VFX simple (flash on hit)
- Modify: `README.md`
- Optional: duplicate portrait icons per hero with CSS tint if no unique art yet

- [ ] **Step 1: Visual polish** — hemisphere light + dim point light on hero ; different fog/ground tint per level ; boss emissive pulse in phase 2+.

- [ ] **Step 2: Balance** — ensure level 1 clearable in <5 min for a new player ; mana regen enough for A spam lightly.

- [ ] **Step 3: README**

```markdown
# Shade Rift

Jeu d'action 3D HTML5 (Three.js) — projet W-WEB gaming.

## Lancer

```bash
npm install
npm run dev
```

## Contrôles

- Clic gauche : déplacer
- A Z E R : sorts (direction curseur)
- Échap : pause

## Règles

Survivez aux vagues, battez le boss, enchaînez 3 niveaux.
```

- [ ] **Step 4: Full manual checklist** (from spec §13)

- [ ] **Step 5: Final commit**

```bash
git commit -am "docs: polish Shade Rift visuals balance and README"
```

---

## Spec coverage check

| Spec item | Task(s) |
|---|---|
| ThreeJS top-down | 3, 8 |
| Menu / Règles / Options / rebind | 3, 12 |
| Click-to-move | 5, 8 |
| Sorts A Z E R + mana/CD | 4, 6, 9 |
| 3 héros select | 4, 8 |
| HUD LoL + images | 11 |
| 3 niveaux + vagues | 4, 9, 10 |
| Boss phases + telegraph | 10 |
| Pickups / or / score | 9, 11 |
| Pause / GameOver / Victory | 10, 12 |
| Dark horror look | 8, 13 |
| README | 1, 13 |
| Settings localStorage | 2, 12 |

## Notes for implementers

- Assets Vite : fichiers dans `assets/` sont servis en `/...` via `publicDir: 'assets'` → portrait = `/hud/hud-portrait.png`.
- Préférer géométrie Three.js native (Box/Capsule/Cylinder) plutôt que GLTF au début.
- Ne pas introduire Cannon/Rapier.
- Chaque task se termine par tests verts (si applicable) + commit.
