# Sandboxels: audit and plan for units, combat, weapons, NPCs and graphics

*Fork:* https://github.com/arbiter1170/sandboxels (upstream: R74nCom/sandboxels, audited at `e0e986b`, game v1.13.2)
*Date:* 2026-11-10 (ET)

---

## 0. License: read this first

The repo uses the **R74n Content License v1.1** (`license.txt`). It is **not** an open-source license. In practice:

| Topic | What it says | What it means for us |
|---|---|---|
| Ownership | "Copyright (C) 2024 R74n, All Rights Reserved" | We get a permission to use the code, not ownership. |
| Takedown | R74n can ask us to remove their content "at any time, for any reason" and can take legal action (§1, §1a) | Anything we build on their code can be pulled on request. Don't put real investment into an engine fork without keeping that risk in mind. |
| Mods | "You are typically free to create and distribute scripts that modify our content, such as mods or plugins" (§1b), still subject to takedown | The mod route is the explicitly sanctioned path. That's one more reason to build our work as mods. |
| Commercial use | **No commercial use** without explicit permission: no ads, no selling, no "other means of generating income" (§2) | Hobby or personal only, unless we get written permission from R74n. |
| Attribution | Credit R74n clearly. Do not remove or alter existing credits, links or popups (§3, §3a) | Keep the footer "© R74n", the links and the credits exactly as they are. |
| Grant-back | R74n "are free to use any content you create using our content for any commercial or other purpose" (§4) | Our mods and derivative work can be reused by R74n, commercially too. |
| Warranty | None (§5) | — |
| Private copy | Not addressed explicitly. Copying and modifying for personal use is consistent with §1b/§3 | A **GitHub fork of a public repo is always public** (GitHub can't make it private). If Vincent wants a private copy, it has to be a separate private repo (mirror push). That is allowed by the license as far as we can tell, but it's a non-public derivative and the same takedown and non-commercial terms still apply. |
| Mod repo | Upstream README: mods are no longer accepted here. New mods go to **R74nCom/Sandboxels-Mods** | Check that repo's own license before copying from community mods. Each mod has its own author. |

**Bottom line:** fine for personal or hobby play and for mods. Not OK to monetise. Keep all credits. Build as a mod, so we never ship modified core files.

---

## 1. What's built: architecture

### 1.1 Shape of the codebase
- **No build system, no framework, no tests.** It's a static site. Everything runs from **one file, `index.html` (21,523 lines)**, with inline `<script>`s, plus `style.css` (1,104 lines).
  - Element data: lines ~866–14,140, about 13.3k lines and **509 top-level element literals**. At runtime there are **550 elements, 232 of them hidden**, because more get auto-generated (molten or frozen variants and so on).
  - Engine, renderer, UI, saves and the mod loader take about 7k lines after that. There are 196 named functions.
- Other parts:
  - `mods/` holds 544 legacy community mods (~658k LOC, 20 MB). This folder is frozen; new mods go to the separate Sandboxels-Mods repo.
  - `lang/` has 49 translations.
  - `lite.html` (1.6k lines) is a small legacy version.
  - `scripts/perlin.js` is used for terrain generation.
  - `service-worker.js` is network-first with a cache fallback, and `manifest.json` makes it a PWA.
  - `archive/` holds old versions. `backgrounds/`, `icons/` and `fonts/` hold assets.
- **Repo size:** 66 MB, mostly mods, archive and media.
- Everything is **global variables** (`elements`, `pixelMap`, `currentPixels`, `pixelTicks`, `width`, `height`, `pixelSize`, `settings`…). That's what makes modding easy, but there's no encapsulation at all.

### 1.2 The element definition format (the core data model)
Every material, creature, tool and weapon is one entry in `elements` (`index.html:866`). A mod adds one with `elements.my_thing = {...}`; after load you'd use `addElement(key, obj)`. Common keys:

- **Look:**
  - `color` (string or array, randomised per pixel).
  - `colorPattern` plus `colorKey` (`textures.BRICK`, `textures.GLASS`…).
  - `glow`, `singleColor`.
  - `renderer(pixel, ctx)` for custom per-pixel drawing.
- **Physics:**
  - `state` (solid / liquid / gas) and `density`.
  - `behavior`, which is either a **3×3 behavior grid** (see below) or a function such as `behaviors.POWDER`, `LIQUID`, `GAS`, `WALL` or `FLY`.
  - `viscosity`, `hardness` (0–1, resists explosions and `DL`), `movable`.
- **Heat:** `temp`, `tempHigh` → `stateHigh`, `tempLow` → `stateLow`, `conduct`, `insulate`.
- **Fire:** `burn`, `burnTime`, `burnInto`, `fireColor`.
- **Breaking:** `breakInto` (an element or a list), `breakIntoColor`, `onBreak(pixel)`.
- **Electricity:** `conduct`, `behaviorOn`, `colorOn`, `charge`.
- **Per-pixel state:** `properties: {...}` is copied onto each pixel at creation (deep-copied if it's an object). **This is how units get HP, team and so on.**
- **Hooks:**
  - `tick(pixel)` runs every tick for each pixel of this element.
  - `onPlace(pixel)`, `onDelete`, `onBreak`.
  - `tool(pixel)` for brush-style elements.
  - `perTick()` runs while the element is selected.
  - `hoverRender(ctx)`, `renderTool(ctx)`.
- **UI:** `category` (each new category gets a new tab automatically), `hidden`, `excludeRandom`, `cooldown`, `related`, `pickElement`, `name`.

### 1.3 Behavior grids
A behavior grid is a 3×3 array of `"CODE[:arg][%chance]"` strings, centred on the pixel, combined with `AND`. It's parsed every tick by `pixelTick()` (`index.html:15747`). There are 29 codes:

> `M1 M2` (move, first or second priority) · `SP SA` (support) · `DL DB` (delete or break) · `CH` (change) · `SW` (swap) · `CR` (create) · `CL CF` (clone) · `SH` (shock) · `ST` (stick) · `LB L1 L2` (leave behind) · `CC` (change colour) · `HT CO` (heat or cool) · `FX FY RT` (flip or rotate) · `BO` (bounce) · `C2` · `EX:r>elem` (explode) · `SM` · `MX` (mix) · `RL` · `XX` (nothing)

Example (`bomb`):
```js
behavior: ["XX|EX:10|XX", "XX|XX|XX", "M2|M1 AND EX:10|M2"]
```
The grids can be rotated (`pixel.r`) and flipped (`flipX`/`flipY`).

### 1.4 Reactions
`elements.A.reactions = { B: {...} }` runs when a pixel of A touches B, through `reactPixels(p1, p2)` at `index.html:14811`. The fields:

- **Turn into something else:** `elem1`/`elem2` (an element, a random pick from a list, or `null` to delete).
- **Conditions:** `chance`, `tempMin`/`tempMax`, `burning1`/`burning2`, `charged`, `y:[min,max]`, `setting`.
- **Side effects:** `temp1`/`temp2`, `charge1`/`charge2`, `color1`/`color2`, `stain1`/`stain2`, `attr1`/`attr2` (set pixel properties), `oneway`.
- **Custom code:** `func(p1, p2)`, an escape hatch for any logic.

`attr1` and `func` are the hooks that let world hazards (fire, acid, lasers) **damage units** without touching core code.

### 1.5 Tick and update loop
- `setInterval(tick, 1000/tps)`. The default is **30 TPS**, and the user can change it.
- Rendering is separate: `requestAnimationFrame(animation)` calls `drawLayers()` / `drawPixels()`.
- `tick()` (`index.html:17405`) runs mouse actions, then random events, then `tickPixels()`, then the `runEveryTick` mod hooks.
- `tickPixels()` (`index.html:17076`):
  1. Copies and **shuffles** `currentPixels` every tick, with an optional `updateOrder` sort.
  2. For each pixel: handles border void and loop, then runs `info.tick(pixel)`, then the `runPerPixel` mod hooks, then `pixelTick()` if the element has a `behavior`.
  3. Moves **relations** (rigid multi-pixel groups, see 1.9).
  4. `pixelTicks++`.
- Heat, burning, electricity and staining run per pixel (`doHeat`, `doBurning`, `doElectricity`, `doStaining`, `doAirDensity` inside `pixelTick`/`doDefaults`).
- Storage: `pixelMap[x][y]` is a 2D array of Pixel objects (or `undefined`), and `currentPixels` is a flat array. **Each pixel is a JS object**, not a typed array. That's simple and very flexible, but it's slow at scale (see §1.11).

### 1.6 Renderer
- **Canvas 2D only. No WebGL anywhere in core.**
  - Layers are offscreen canvases (`canvasLayers.pixels`, `canvasLayers.gui`, plus `canvasLayersPre`/`Post`), composited onto `#game`.
  - Each pixel is drawn with **one `fillRect` per pixel** (`drawSquare`) or a "plus" shape for glowing gases (`drawPlus`).
  - Colours are CSS strings, cached rgb→hex.
- Views (keys 1–5): Normal, Thermal, Basic, Streak (fade trail), Outline.
- **Mod render hooks:** `renderPrePixel(ctx)`, `renderEachPixel(pixel, ctx)`, `renderPostPixel(ctx)`, `addCanvasLayer(name)`, a per-element `renderer`, and new views through `viewInfo`.
- Community graphics mods already exist, all Canvas 2D:
  - `glow.js` (blurred emit layer; it checks for Chromium and refuses to run elsewhere).
  - `lightmap.js` / `fast_lightmap.js`, `light.js`, `real_light.js`, `heatglow.js`, `shader_by_jayd.js`.

### 1.7 UI
- Plain DOM: a top toolbar (Pause, Step, ±, Reset, Replace, Elem, Edit, Info, Saves, Mods, Settings), tool buttons (Heat, Cool, Erase, Drag, Pick, Mix, Lookup, Shock, Paint, Smash), category tabs and element buttons.
- A pixel font, a log line and a stats line (`x,y Pxls:N 30tps`).
- Modal "menus" for Info, Saves, Mods and Settings. Keyboard shortcuts live in a `keybinds` map that mods can extend.
- i18n through `lang/*.json` and `langKey()`.

### 1.8 Mod loader
- `enabledMods` (an array of URLs) is stored in `localStorage`. At startup each entry is injected as a `<script src>`. A bare name like `foo.js` resolves to `mods/foo.js`, and full `http(s)` URLs are allowed. The Mod Manager UI adds and removes entries.
- Mods run **before** element finalization, so `elements.x = {...}` is all you need.
- Dependencies: `dependOn(modName, cb, forceLoad)`, `modIsEnabled(name)`.
- Lifecycle hooks:
  - `runAfterLoad`, `runAfterAutogen`.
  - `runEveryTick`, `runPerPixel`.
  - `runAfterReset`, `onAddElement`.
  - `validateMoves` (veto or redirect any `tryMove`).
  - The render hooks above.
- No sandboxing. Mods have full page access, and a broken mod is only caught with `try/catch` on insert.
- **Verified:** I loaded our example mod (§4) through `enabledMods` locally. A "Skirmish" tab appeared, units fought, HP bars drew, and there were zero page errors.

### 1.9 Multi-pixel bodies ("relations")
- Pixels sharing `pixel._r` form a rigid group. `tryMoveRelation(relation, dx, dy)` moves the whole group if every target cell is free, or holds lighter liquids or gases (which it displaces).
- Groups fall as one unit each tick, and the **Group** tool (`"group"` element, `index.html:2758`) builds them by hand. Saves keep relation IDs.
- **This is the existing primitive for vehicles, tanks and large units.**

### 1.10 Save format
- `generateSave()` produces JSON with `meta` (gameVersion, mods list…), a size, a palette of `codes` (element → one character from 'A' upward), a per-pixel grid with non-default properties (x, y, temp, start and colour are stripped unless needed), settings and `relations`.
- Saves go to slots in localStorage, to files, to a URL, and there's a Discord featured-saves browser.
- **Custom pixel properties (`hp`, `team`…) are saved automatically.** Elements from a mod require that mod at load time (unknown elements become `unknown`).

### 1.11 Creatures, humans, NPC-like behaviour today
- **`human`** (`index.html:4026`) is a placement helper. `onPlace` turns it into a **2-pixel stack: `head` above `body`**, and only `body` and `head` exist in the world.
  - `body.tick`:
    - gravity, which drags the head along;
    - death and rot timers (`dead` = tick of death, rotting to `rotten_meat`);
    - decapitation, which makes blood and a chance of death;
    - **panic**, raised by reactions with fire, acid, plague, TNT and so on through `attr1:{panic:N}`, and by "seeing" scary pixels in a small cone ahead;
    - spreading panic to neighbours;
    - random walking (more when panicking), climbing 1 cell, and a 15% chance per move to turn around (`dir *= -1`).
  - `head.tick` has its own death and panic handling, and the head copies state to the body.
  - **There's no goal, no targeting, no pathfinding, no faction, no HP.** "Health" is just turning into another element (cooked_meat, rotten_meat, blood) through heat or breaking.
- **Animals** (`rat`, `bird`, `fish`, `frog`, `bee`, `ant`, `fly`, `spider`, `worm`…) use behavior presets like `behaviors.FLY`, `CRAWLER` and `ANIMAL`, plus `FEEDPIXEL` reactions (eat a neighbour). It's random-walk AI.
- **Community mods** already go much further: `mobs.js` (5k lines: creepers, zombies, skeletons, with head/body pairs, spawning and settings), `humans.js`, `fantastic_creatures.js` and `war.js`. They show the appetite, and they also show that every mod reinvents the same plumbing.

### 1.12 Weapons and explosives today
- The core `weapons` category: `bomb`, `tnt`, `c4`, `grenade`, `dynamite`, `firework`, `nuke`, `rocket`, `flash`, `landmine`, `gunpowder`, plus `laser` (energy, bouncy, deletes on hit).
- Everything funnels into **`explodeAt(x, y, radius, fire)`** (`index.html:16804`). Damage falls off with distance and is reduced by `hardness`:
  - damage above 0.9 turns the pixel into fire;
  - above 0.25 breaks it (`breakInto`/`onBreak`);
  - otherwise it adds heat. (There's also a `>0.75 && burn` → `burnPixel` branch, but it can't be reached because of the `continue`s before it.)
- There's no notion of a projectile entity with velocity in core (`laser` is just a bouncy pixel). The `velocity.js` mod adds that. The root also has `weaponsRewrite.js` (a mod-style file: tsar bomba, directional missiles).

### 1.13 Performance and limits
These were measured headless on the box (Chromium, software render, no GPU), so treat them as relative numbers.

| Scenario | Grid | Pixels | `tickPixels` | `drawPixels` |
|---|---|---|---|---|
| Desktop default (pixelSize 6) | 166×82 | 1,051 (scene) | 0.9 ms | — |
| Desktop default, filled | 166×82 | 6,135 | 2.1 ms | 4.5 ms |
| 390 px phone (pixelSize 5) | 78×102 | 3,606 | 1.5 ms | 2.9 ms |
| Desktop, pixelSize 2, cap off | 500×249 | 82,170 | ~10 ms | ~11 ms |

- **The default grid is tiny** (desktop 166×82 cells, phone 78×102). `maxPixelCount` defaults to 13,861. Below 700 px wide, `pixelSize` drops by 1. The canvas is capped at 1000×500 CSS px on desktop.
- Costs that grow with the pixel count:
  - a full array copy and shuffle every tick;
  - per-pixel JS objects and GC pressure;
  - a string-based behavior parse every tick for grid elements;
  - one `fillRect` per pixel per frame.
- A 2-pixel human costs about what a sand grain does, so **hundreds of units is fine and thousands is not**. Per-unit AI must stay cheap: no per-tick BFS per unit.

### 1.14 Mobile UX (390 px screenshots)
- It works with touch (`touchstart`/`touchmove` on the canvas) and is installable as a PWA.
- The canvas takes the top ~60% of the screen. Below it are rows of small pixel-font buttons. The toolbar and the category tabs **scroll sideways and get cut off** ("Re…", "Soli…"), with no visible scroll hint.
- No pinch-zoom or pan; a cell is ~5 CSS px, so placing a single "unit" by finger is imprecise.
- No undo. Brush size is only changed with the ± buttons.
- `mobile_keybinds.js` and `mobile_shift.js` exist as mods, which tells us the mobile gaps are known.

Screenshots were captured during the audit (they aren't committed to the fork):
- `desktop-01-initial.png`, `desktop-02-scene.png`, `desktop-04-stress.png`, `desktop-05-pixelsize2-stress.png`, `desktop-06-example-mod.png`
- `mobile-390-01-initial.png`, `mobile-390-02-scene.png`, `mobile-390-03-fullpage.png`, `mobile-390-04-stress.png`

---

## 2. Proposal: units, combat types, weapons, NPCs

### 2.1 Principle: one mod pack, zero core edits
- Build everything as a mod pack, say `mods/skirmish/` with `skirmish_core.js`, `skirmish_units.js`, `skirmish_weapons.js` and `skirmish_fx.js`, loaded through the normal Mod Manager.
- Why:
  - The license explicitly allows mods (§1b).
  - We stay **mergeable with upstream forever**: just pull `main`.
  - Any edit to `index.html`, a 21k-line file that upstream changes constantly, guarantees merge conflicts.
- Core edits are reserved for Phase 3, and even then they go behind hooks we'd offer upstream as small PRs, and only if Vincent says so.

### 2.2 Data-driven schema (the "core" mod)
`skirmish_core.js` turns compact definitions into regular `elements` entries:

```js
Skirmish.defineUnit("rifleman", {
  team: "red", color: ["#b03a2e","#c0392b"],
  body: "2x1",                // 1x1, 2x1 (head+body like human), or a relation sprite for vehicles
  hp: 20, armor: { kinetic: 0.1, fire: 0, blast: 0.2 },
  speed: 0.15,                // chance/tick to step (Sandboxels units are probabilistic)
  sight: 30, weapon: "rifle",
  ai: "soldier",              // state machine id
  onDeath: ["blood","meat"]
});
Skirmish.defineWeapon("rifle", {
  projectile: "bullet", cooldown: 12, spread: 1, damage: { kinetic: 5 }, range: 40
});
Skirmish.defineProjectile("bullet", { speed: 2, gravity: 0, color: "#ffd84a",
  onHit: { unit: "damage", terrain: { chance: 0.3, action: "break" } } });
```

Under the hood each call becomes an element:
- `properties` hold `hp`, `maxHp`, `team`, `state`, `dir`, `cd`, `target`.
- A generated `tick` dispatches to the AI state machine.
- `reactions` map world elements to damage types.

That way a unit is ~10 lines of data, and modders or designers add content without touching engine code.

### 2.3 Combat resolution
- **Health and armor live on the pixel** (`pixel.hp`, with armor looked up from the unit definition). For multi-pixel units, HP lives on the "core" pixel (the body, or relation pixel 0), and the other pixels forward damage to it.
- **Damage types:** `kinetic` (bullets, shrapnel, melee), `fire`, `blast`, `acid`, `energy`, `cold`.
  - `final = dmg * (1 - armor[type])`.
  - Then a status: `burning` (reuse core `burnPixel`/`pixel.burning`), `stunned` (electric `charge`), `bleeding` (drop `blood`).
- **Bridge to the existing world:** every core hazard becomes a damage source through `reactions`:
  - `fire`/`plasma` → fire,
  - `acid` → acid,
  - `laser`/`electric` → energy,
  - `explosion` → blast.
  - It uses `func:` (example in §4), so the 550 existing elements immediately interact with units.
- **Explosions:** wrap `explodeAt` with a small post-hook that also applies `blast` damage to unit pixels in the radius. That's safer than editing it. Or ship our own `Skirmish.explode(x, y, r, dmg)` that calls the original and then damages units.
- **Melee:** adjacent enemy plus cooldown means direct damage. No projectile needed.
- **Death:** `changePixel(pixel, def.onDeath)` drops blood, meat or bone, so corpses join the normal simulation (rot, burn, attract flies).

### 2.4 Projectiles as elements
- A projectile is a pixel with `properties: {vx, vy, dmg, team, life}`.
  - Its `tick` steps |v| cells along a short Bresenham line (`lineCoords` already exists) and checks each cell.
  - Gravity is just `vy += g`, which gives arcs for grenades and mortars.
  - `life` counts down to despawn.
- Hit rules: a unit takes damage; terrain gets `breakPixel` with a chance or `hardness`; liquids slow it (density); `fire` ignites it if it's an incendiary.
- Variants we get almost free:
  - **rocket**: leaves `smoke` (`CR:smoke` or `createPixel` behind) and calls `explodeAt` on hit;
  - **flamethrower**: emits `fire` pixels with velocity;
  - **tesla**: sets `charge` on the target;
  - **frost**: `temp -= N`, so the core freezes water;
  - **acid shell**: spawns `acid`.

### 2.5 NPC behaviour: factions, targeting, state machines, pathing
- **Factions:** `Skirmish.factions = { red: {hostile:["blue","zombie"]}, blue: {...}, civilian: {...} }`. Core `body`/`head` humans count as "civilian" through a property we add (or `team` undefined, meaning neutral).
- **Small finite state machine per unit:** `idle → patrol → engage → flee → dead`, plus `follow` and `guard` for squads.
  - Transitions come from cheap sensors: hp below a threshold means flee; an enemy in sight means engage; panic from the core human system can feed flee.
  - States are plain functions, `Skirmish.ai.soldier = { patrol(p){}, engage(p){}, flee(p){} }`, selected by `pixel.state`.
- **Targeting (cheap):**
  - Line-of-sight raycast along the facing direction in a 3-row cone (the example mod does this), and sensing only every N ticks (`(pixelTicks + id) % 5`) to spread the cost.
  - Per tick, a `runEveryTick` hook builds a **spatial bucket of units by team** (e.g. 16×16 cells), so "nearest enemy" doesn't scan every pixel.
- **Pathing on the grid:**
  - **MVP:** "walker" movement, like the core human: step forward, climb 1, turn at walls, fall with gravity.
  - **Phase 2:** a **flow field** (Dijkstra map) per faction target, recomputed every ~30 ticks with a BFS over passable cells, so every unit just follows the gradient. That's one BFS per faction rather than per unit, which scales to hundreds of units.
  - Treat liquids as slow, fire as high-cost, and solids as walls or "diggable" for sappers.
- **Squads and vehicles:** use **relations** (`_r`) for rigid multi-pixel units (tank, mech, boat). `tryMoveRelation` already handles movement, falling and liquid displacement. The core pixel holds AI and HP.
- **Spawners and bases:** elements that `createPixel` units on a timer, which also lets "capture the flag" or "defend the base" modes run inside the sandbox.

### 2.6 How it plays with existing reactions (examples)
- Soldiers wading through `water` slow down; `lava` cooks them (`tempHigh` → `cooked_meat`); `plague` infects them (reuse core reactions by copying `elements.body.reactions`).
- A `tnt` wall is a valid defence that blows up when bullets hit it (bullet `onHit` sets `burning`).
- `zombie` faction units can convert `body` (human) pixels through `reactions: { body: { elem2: "zombie_body", chance: 0.1 } }`.
- Core `human` panic already reacts to `tnt`/`gunpowder`. Add our `bullet` and `rocket` to `elements.human.reactions` with `attr1:{panic:5}` so civilians flee battles.

---

## 3. Graphics improvements (with tradeoffs)

| Option | Gain | Cost / risk | Mod-able? |
|---|---|---|---|
| **A. Palette and texture pass** (curated 32–48 colour palette, `colorPattern` textures for unit uniforms, outlines on units) | Instant clarity; units readable at 5–6 px | Nearly free | Yes |
| **B. Canvas 2D overlay layer** (`addCanvasLayer` + `renderPostPixel`): HP bars, team markers, selection rings, muzzle flashes, tracer lines, damage numbers | The combat becomes readable | Cheap; draw only units, not all pixels | Yes (example in §4) |
| **C. Sprite overlays for units** (draw a 8×8/12×12 sprite atlas at unit pixels; vehicles as relation-sized sprites) | Real characters instead of 2 coloured dots | Breaks the "everything is sim pixels" purity; sprites must follow sim pixels; need art (Kenney packs/CC0) | Yes, via `renderer`/`renderPostPixel` |
| **D. Glow and light** (bloom from `emit` pixels, as in `glow.js`; a coarse lightmap at ¼ resolution, blurred, multiplied) | Explosions, fire, lasers look great; day/night | `glow.js` uses `ctx.filter` blur, which is Chromium-only and slow on phones. Downsample to keep it cheap | Yes |
| **E. Particle FX layer** (non-sim particles: sparks, debris, smoke puffs with alpha, screen shake) | Juice without spending sim pixels or pixel cap | Must stay purely visual so saves/sim are unaffected | Yes (`runEveryTick` + `renderPostPixel`) |
| **F. WebGL renderer** (upload a `width×height` RGBA texture each frame from the pixel grid; one quad; fragment shader for palette, glow, lighting, CRT/pixel-perfect scaling, post-processing) | 10–50× cheaper drawing than one `fillRect` per pixel; enables big grids on phones and real post FX | Biggest change: touches `drawPixels`/views. Can be done as a mod that replaces `drawPixels` and adds a GL canvas to `canvasLayersPre`, but it's fragile against upstream changes. Need a Canvas 2D fallback | Partly; cleanest as core change |
| **G. Sim data layout** (typed arrays for element id, temp, colour instead of pixel objects) | Large sim speedup, bigger battles | Rewrite of core; breaks most of the 544 mods | No: a fork, not a mod |

**Mobile performance:** keep the default grid, prefer A, B and E, add a WebGL path (F) before any blur-based glow, and cap AI sensing frequency. On the mobile UX side, a mod can add pinch-zoom and pan on a CSS transform of `#game`, a fixed bottom tool bar with "unit, weapon, erase, pause", and a selected-unit info card. That's all DOM work, with no core edits.

---

## 4. Example mods (grounded in the real API, tested locally)

### 4.1 Two factions with a projectile and HP bars
The full file is the snippets in 4.1 and 4.2 combined. I tested it locally in headless Chromium: in a 1v1, HP went 20→15→10 from bullet hits, and in a 6v6, 7 units were hurt and 1 died to blood within ~6 s. There were no page errors.

```js
function makeSoldier(team, color, dir) {
  return {
    color: color, category: "skirmish", state: "solid", density: 1500,
    temp: 37, tempHigh: 150, stateHigh: "cooked_meat", breakInto: ["blood", "meat"],
    properties: { hp: 20, maxHp: 20, team: team, dir: dir, cd: 0 },   // copied onto each pixel
    tick: function (pixel) {
      if (pixel.hp <= 0) { changePixel(pixel, "blood"); return; }
      if (tryMove(pixel, pixel.x, pixel.y + 1)) return;              // gravity
      if (pixel.cd > 0) pixel.cd--;
      scan: for (let d = 1; d <= 25; d++) {                          // 3-row line-of-sight cone
        const x = pixel.x + d * pixel.dir;
        if (outOfBounds(x, pixel.y)) break;
        for (let dy = -1; dy <= 1; dy++) {
          if (isEmpty(x, pixel.y + dy, true)) continue;
          const p = pixelMap[x][pixel.y + dy];
          if (p.team && p.team !== pixel.team) {
            const bx = pixel.x + pixel.dir;
            if (pixel.cd === 0 && isEmpty(bx, pixel.y)) {
              createPixel("bullet", bx, pixel.y);
              pixelMap[bx][pixel.y].vx = pixel.dir;
              pixelMap[bx][pixel.y].vy = dy;
              pixel.cd = 12;
            }
            break scan;
          }
          if (dy === 0) break scan;
        }
      }
      if (Math.random() < 0.1 && !tryMove(pixel, pixel.x + pixel.dir, pixel.y)) {
        if (!tryMove(pixel, pixel.x + pixel.dir, pixel.y - 1)) pixel.dir *= -1;   // climb or turn
      }
    },
    reactions: {                                   // existing hazards damage units via core reactions
      "fire":  { func: function (u) { u.hp -= 2; } },
      "acid":  { func: function (u) { u.hp -= 4; } },
      "laser": { func: function (u) { u.hp -= 6; }, elem2: null }
    },
    excludeRandom: true
  };
}
elements.soldier_red  = makeSoldier("red",  "#c0392b",  1);
elements.soldier_blue = makeSoldier("blue", "#2e6fd8", -1);

elements.bullet = {
  color: "#ffd84a", category: "skirmish", state: "solid", density: 7800,
  properties: { vx: 1, vy: 0, dmg: 5 },
  tick: function (pixel) {
    const nx = pixel.x + pixel.vx, ny = pixel.y + pixel.vy;
    pixel.vy = 0;
    if (outOfBounds(nx, ny)) { deletePixel(pixel.x, pixel.y); return; }
    if (isEmpty(nx, ny)) { tryMove(pixel, nx, ny); return; }
    const hit = pixelMap[nx][ny];
    if (hit.hp !== undefined) hit.hp -= pixel.dmg;                     // unit damage
    else if (isBreakable(hit) && Math.random() < 0.3) breakPixel(hit); // terrain damage
    deletePixel(pixel.x, pixel.y);
  },
  excludeRandom: true
};
```

### 4.2 Overlay rendering (HP bars) through the render hook
```js
renderPostPixel(function (ctx) {
  for (let i = 0; i < currentPixels.length; i++) {
    const p = currentPixels[i];
    if (!p || p.maxHp === undefined || p.hp >= p.maxHp) continue;
    const w = Math.max(0, p.hp / p.maxHp);
    ctx.globalAlpha = 1;
    ctx.fillStyle = "#300";
    ctx.fillRect(canvasCoord(p.x) - pixelSize, canvasCoord(p.y) - 3, pixelSize * 3, 2);
    ctx.fillStyle = w > 0.5 ? "#3c3" : "#e33";
    ctx.fillRect(canvasCoord(p.x) - pixelSize, canvasCoord(p.y) - 3, pixelSize * 3 * w, 2);
  }
});
```
(For production, keep a `Set` of unit pixels maintained in `tick`/`onDelete` instead of scanning `currentPixels` every frame.)

---

## 5. Phased plan

| Phase | Scope | Where | Done when |
|---|---|---|---|
| **0. Setup** (done) | Fork, audit, example mod verified locally | fork `audit` branch | This doc merged into the fork |
| **1. MVP mod** (~1–2 sessions) | `skirmish_core.js` (defineUnit/defineWeapon/defineProjectile, factions, damage types, HP/armor, death drops), 3 units (rifleman, grenadier, medic), 3 weapons (rifle, grenade arc, flamethrower), reactions bridge to fire/acid/laser/explosions, HP-bar overlay, spawner element | `mods/skirmish/*.js` in our fork (load by URL from GitHub Pages / raw) | Two teams fight on a sandbox map; units die to core hazards; no page errors; Playwright smoke at desktop + 390 px |
| **2. Depth** | FSM states (patrol/engage/flee/guard), flow-field pathing per faction, spatial buckets, relation-based vehicles (tank), zombies vs civilians using core `body`, sprite overlay + palette pass, particle FX layer, mobile control mod (pinch-zoom, bottom unit bar) | still mods | 200 units at 30 TPS on a mid phone; readable sprites |
| **3. Engine** (only if needed, and only with Vincent's OK) | WebGL renderer (texture upload + shader post FX) with 2D fallback; optional hook PRs upstream (e.g. `onExplode`, projectile helper); typed-array sim only if we accept forking off upstream | core | Measured gain against Phase 2 baseline; upstream-merge cost acknowledged |

**Constraints to remember:**
- Non-commercial only.
- Keep the R74n credits.
- R74n can request takedown, and can reuse our mods.
- Fork is public. A private copy needs a separate private repo.
