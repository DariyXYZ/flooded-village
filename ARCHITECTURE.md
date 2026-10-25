# Flooded Village — Architecture

This document is for anyone (human or AI) about to change the game. Read it before you edit `index.html`.
It explains how the pieces fit, which invariants hold the look together, and which edits break things in non-obvious ways.

---

## 1. Shape of the project

| File | Role |
|---|---|
| `index.html` | **The entire game.** One ES-module `<script>`: config, shaders, world, entities, game loop. ~1.5k lines. |
| `README.md` | Player-facing description and controls. |
| `ARCHITECTURE.md` | This file. |
| `AGENTS.md` / `CLAUDE.md` | Short rules for coding agents, pointing here. |
| `docs/preview.png` | README image. `docs/*.mp4` is git-ignored. |
| `.nojekyll` | Lets GitHub Pages serve the repo as-is. |

- **No build step.** three.js `0.160.0` is loaded through an import map from unpkg (`three`, `three/addons/`). Do not change the version without retesting every `onBeforeCompile` patch — they replace three's internal shader chunks (`#include <opaque_fragment>`, `#include <project_vertex>`, `#include <begin_vertex>`) by name.
- **Deploy:** push to `main` → GitHub Pages (legacy build from `/`). Live at https://dariyxyz.github.io/flooded-village/. Pages takes ~1 min.
- Comments in the code are in Russian. Keep that style when you edit nearby code.

### URL modes (use them to test)

| URL | What it does |
|---|---|
| `/` | normal game: city builds → intro voyage with 4 speech bubbles → play |
| `?test` | skips intro; 96 logs already on the dam, 4 logs floating near it |
| `?ending` | skips intro; dam complete, ending (drain) starts immediately |

---

## 2. Source layout (order inside `<script>`)

Sections are marked `// ---------- name ----------`. Top to bottom:

1. **настройки (config)** — all tunable constants (`MAX`, `HALF_W`, `RX/RZ`, `DAM_X`, `LOGS`, `GOAL`, `CAP`, `BOAT_START`, `SHOP`, `TEST`, `ENDING` …).
2. **форма воды** — JS signed-distance functions for the world (`sdBasin`, `sdChannel`, `shoreWarp`, `sdWater`, `waterGrad`, `inChannel`) and the **same functions as GLSL strings** `GLSL_WORLD`, plus `GLSL_NOISE` (`ring`, `hash`, `vnoise`, `sdSeg`).
3. **рендер** — renderer, off-screen render target `rt`, scene, fixed 45° camera (`CAM_OFF`), OrbitControls (zoom only), sun + shadow map, `shadowCatcher`, shared materials (`blockMat`, `windowMat`, `reflMat`, `reflMaskMat`, `shopMat`) and the `blackShadows` shader patch.
4. **вода** — uniform arrays (`segU`, `infoU`, `wakeU`, `pudU`, `rockU`) and `waterMat`, the big water/ground fragment shader on a 600×600 plane. `rippleMat` — per-log ring decals.
5. **пост** — `postMat`: the one-bit halftone post-process. `resize()`.
6. **геометрия домика** — `roofGeo`, `houseGeo(len)`, `windowGeo(len, type)` with caches.
7. **блоки** — `Block` class (a house), physics helpers (`segSeg`, `impulse`, `magnet`, `field`), `spawn`.
8. **генератор фигур** — `FIG` typologies (`single`, `small`, `shop`, `cross`), `FIG_MIX`, spawn `queue`, `planCity`.
9. **брёвна** — `Log` class, `freeSpot`, `scatterLogs`, rocks (`placeRocks`, `pushOutRocks`).
10. **дамба** — `buildDamRuins`, `addToStack` (the log stack = progress display).
11. **берег** — `noise2` (CPU twin of `vnoise`), `tuftGeo`, `grassMat`, `noisyStone`, `bushGeo`, `plantShore()` (puddles, grass chunks, boulders, sticks/pebbles). Runs **once** at load.
12. **кораблик** — boat mesh, `DECK` slots, intro (`INTRO_LINES`, `startIntro`, `updateIntro`, `drawBubble`), `capsize`/`spillOne`, `updateBoat`, `followCam`.
13. **партия** — `newGame()`, `freezeCity()`, `layBed()`/`revealBed()` (lakebed debris for the ending), `disposeTree`.
14. **ввод** — keyboard (by `e.code`, layout-independent).
15. **цикл** — `frame()`: the game loop.

---

## 3. Game flow / state machine

```
load ──► newGame()
          │  startSplash(): splash.scene (black water, 5 logs surfacing) is rendered
          │  instead of the world; the world keeps simulating behind it
          │  city queue filled (planCity), boat hidden
          ▼
     BUILDING  (frozen = false)
          │  spawn() pops queue; magnet()+field() push houses apart
          │  queue empty + settle 1.5 s
          ▼
     freezeCity()  → frozen = true, houses never move again
          │  scatterLogs(100)  (or ?test / ?ending setup)
          │  layBed()  (hidden lakebed debris)
          │  splash ends when frozen && splash.t > 5.2 → black fade → world → startIntro()
          │  (test modes skip splash and intro)
          ▼
     INTRO  (intro.on)  autopilot from dam → east, 4 bubbles, player input ignored
          ▼
     PLAY   collect logs (max CAP=5, 6th → capsize), unload at dam → delivered++
          │  delivered ≥ GOAL  or  no logs left anywhere
          ▼
     WON  (won = true) → drain 0→1 over 10 s: lake east of the dam dries into puddles,
          houses settle, boat sinks, camera pulls back, outro text fades in
```

Key state variables: `frozen`, `intro.on/step/hold`, `won`, `drain`, `delivered`, `boat.cargo`, `boat.spill`, `boat.tip`.
`newGame()` must reset **every** one of these (plus caches, groups, camera, shadow frustum) — if you add state, reset it there.

---

## 4. Coordinate system & world

- Y up, water plane at **y = 0**. The lake is an ellipse-ish SDF centred at the origin (`RX=38`, `RZ=24`).
- West (−X) is the **channel** narrowing to the dam at `DAM_X = −56`. `CH_END` is where the channel meets the lake.
- **`sdWater(x, z)` < 0 = water, > 0 = land.** It is defined **twice**: in JS (collisions, placement) and in GLSL (`GLSL_WORLD`, drawing). **They must stay identical**, including `shoreWarp`.
- `noise2` (JS) and `vnoise`/`hash` (GLSL) are the same value-noise with a **sin-free hash**. Never reintroduce `fract(sin(x)*43758…)` — GPU `sin` is imprecise at large arguments and CPU/GPU results diverge (e.g. puddles drawn in one place, debris placed in another).

---

## 5. Rendering pipeline — the one-bit look

Everything is rendered in colour/grey to `rt`, then `postMat` turns it into pure black & white.

### 5.1 The alpha channel is a material-class flag (critical)

The post shader reads `rt.a` to decide how to treat each pixel:

| alpha written | used by | post-process treatment |
|---|---|---|
| **1.0** (default) | houses, logs, boat, dam, shop | quantised to **3 tones** (black / 0.4 halftone / white) **+ black edge lines** |
| **0.3** | grass, bushes, ground stones/sticks, river rocks | 3 tones, **no edge lines** (thin geometry would turn black) |
| **0.1** | water reflections | **not quantised**, smooth halftone |
| **0.0** | water plane, log ripple decals, shadow catcher | not quantised; very dark tones (0.4–4.5 %) use a sparse 1-pixel Bayer instead of halftone dots |

Rule: **any new object decides its look by the alpha it writes.** A new material that leaves alpha at 1 gets outlines; to opt out, patch `opaque_fragment` to write `.3` (see `bushMat`).
The shadow catcher's blending keeps destination alpha (`blendSrcAlpha: Zero, blendDstAlpha: One`) so shadows don't create fake edges — keep it that way.

### 5.2 Post shader (`postMat`)

- `lum()` applies gamma 1/2.2 — so a linear value of 0.0014 shows as ~5 % dots. When you tune darkness, **think in screen tone, not linear value**: screen ≈ linear^(1/2.2).
- Halftone: 45° screen, cell `uCell = pixel` physical px (`[` `]` keys, default 4). Dot area = tone.
- Tones below ~4.5 % on alpha-0 surfaces switch to `b8()` Bayer at 1 px — the halftone dot would be smaller than a pixel. **Avoid values that hover near 0.045 or 0.03** on large areas: the switch is a visible seam.
- Edges: luminance jump > .12 or neighbour with alpha < .5 → black. Computed on the **unquantised** `l0`.
- `T` key toggles a pure-Bayer mode (`uMode = 1`).

### 5.3 Hard black shadows

- `THREE.BasicShadowMap` (no filtering), 4096², ortho frustum ±26 around the camera target (±58 during the ending).
- `followCam()` **snaps the sun target to the shadow texel grid** in the light's basis (`LIGHT_R`, `LIGHT_U`) and keeps the component along `SUN_DIR`. Without snapping, shadow edges crawl as the camera moves and rings on the water flicker in and out of shadow.
- `blackShadows` patch: a face that faces the sun but receives no direct light becomes **pure black** (cast shadow). Faces turned away from the sun keep ambient → halftone grey. Apply it to any new lit material.
- `shadowCatcher` (y = .003) draws cast shadows onto the water/ground. It uses `polygonOffset` + `renderOrder 1` to avoid z-fighting with the water plane.

### 5.4 Water/ground shader (`waterMat`)

One fragment shader decides per pixel, in this order:
1. **Land** (`sdWater > 0`): sparse noise-patched dither (`silt`-like), black puddles from `uPud`, white specks, thin shore line.
2. **Water**: distance to every house segment (`uSeg`, `uInfo`) → spawn waves and breathing contour rings; boat wake (`uWake`); shore surf; channel streamlines bending around rocks (`uRock`, stream function ψ = y(1 − R²/r²)), fading out per-lane east of the dam.
3. **Drain** (`uDrain > 0`, ending): east of the dam water fades into the lakebed texture; noise field `pn > t` = remaining puddles (black), with thin broken lines on their right side.

`v < .012 → 0` cuts gamma-inflated tails. It must run **before** the drain block, otherwise the dark lakebed is zeroed.

### 5.5 Uniform budget (do not exceed)

The water fragment shader holds `uSeg[MAX] + uInfo[MAX] + uWake[24] + uPud[64] + uRock[RK]` ≈ **188 vec4**.
WebGL2 only guarantees **224** fragment uniform vectors; many phones give 256. Going over = **black screen on mobile**, no error on desktop.
→ If you need more houses/puddles/rocks, pack data or move it to a texture; don't just raise `MAX`, `PUD` or `RK`.

### 5.6 Reflections

Each house has `refl` (EdgesGeometry line segments, mirrored `scale.y = −1`) and `reflMask` (same mesh, depth-only, polygon-offset back). The mask hides back edges, so only visible outlines remain.
`reflMat` discards fragments whose ray hits land (`sdWater > 0` at the y=0 intersection). The water material has `depthWrite = false` so the mask below the surface works.

### 5.7 Render order

`water (−2) → reflMask (−1.5) → refl (−1) → everything else → shadowCatcher (1)`. Changing these breaks reflections or shadows.

---

## 6. Entities

### Houses (`Block`)
- Built by `spawn(spec)` during BUILDING; `magnet()` impulse on spawn, `field()` keeps a `GAP` between non-crossing houses.
- `cross` figures are two houses at exactly 90°; crossing members share a **pin** and move as one rigid body (`rootBlock`, `setPin`).
- After `freezeCity()` nothing moves (`frozen` short-circuits `impulse`). The shop (`shop: true`) never moves.
- Geometry: `houseGeo(len)` = walls + Japanese roof (`roofGeo`) + ridge + end tiles; one storey (`TIER2 = 1e9`). `windowGeo(len, type)` — 5 window types (`WIN_TYPES`). Both cached by length and cleared in `newGame()`.
- `MAX = 48` caps houses at 47 (last uniform slot is the boat).

### Logs (`Log`)
- Exactly `LOGS = 100`, `GOAL = 100`. Static in the lake; in the channel a current pushes them **away** from the dam (`damCurrent`).
- Each log has its own `rippleMat` clone (ring decal, max-blended). Spilled logs keep their length (`userData.len`).
- Win also triggers when no logs remain anywhere (prevents a soft-lock if one gets stuck).

### Boat
- `updateBoat`: throttle/steer/brake, load slows it (`-6 %` speed per log), sway grows from 3 logs, `CAP + 1` → `capsize()` (logs slide off one by one over `TIP_T = 5 s`).
- Collides with land (`sdWater`), houses (segment distance), rocks, the dam stack (`wall` accounts for the bow length).
- Near the dam the current is stronger than the boat — that is the "soft wall". Unloading zone: `boat.x < DAM_WALL + 4.5`.

### Shore (`plantShore`, once at load)
- Puddles (`pudU`, 64) → boulder spots (no grass on them, ring of tall grass around) → grass clumps in 10×10 InstancedMesh chunks (view-culled) → sticks/pebbles.
- **Not re-run on R** (shore stays the same across games; only city, logs, rocks, dam stack, lakebed are rebuilt).

### Intro & speech bubble
- `drawBubble(text)` draws into a tiny canvas (`#bubble`) at 1 px = 2 physical px, superellipse "pillow" with a random tail, Times-like serif text, then **hard-thresholds every pixel to black/white** (no grey). White threshold 225 keeps thin serifs.
- `INTRO_X` are the four x-positions where lines appear.

---

## 7. Invariants checklist — read before committing

1. **JS ↔ GLSL twins stay identical:** `sdWater`/`shoreWarp`, `chanHW`, `noise2`↔`vnoise`, `hash` (sin-free), `bedDry`↔`pn` in the drain block.
2. **Alpha = material class** (table 5.1). New objects pick the right alpha.
3. **Uniform budget ≤ 224 vec4** in `waterMat`.
4. **Everything created per game is disposed in `newGame()`** (`disposeTree`, cache clears). Don't `group.clear()` without disposing geometry. Don't dispose shared geometries (`stackGeo`, `logGeo`, `houseGeo` cache is cleared deliberately).
5. **Shadow snapping in `followCam()`** stays in place.
6. **Dark tones:** avoid large areas sitting on the 0.03 / 0.045 thresholds; remember gamma.
7. **Minimal HUD text:** intro bubbles, outro line, the `n / 100` tag over the dam, the coin wallet over the boat (E) and the `+N` coin pop-ups. All are pixel canvases thresholded to pure black/white (`crisp`) — no grey, no DOM text in the world view.
8. **Camera is fixed at 45°** (`enableRotate = false`); the world is only built for that view.
9. Keyboard uses `e.code`; ignore `e.repeat` for one-shot actions; clear keys on `blur`.
10. Keep it a single file with no build step.

---

## 8. How to test a change

1. Serve the folder (`npx serve .`) or open `index.html` directly.
2. Check the console for shader compile errors (a broken `onBeforeCompile` string → black screen).
3. Run all three modes: `/` (intro + play), `?test` (deliver 4 logs → ending), `?ending`.
4. Press **R** a few times (resets, no leaks, no duplicated objects).
5. Move the boat around the dam, along the shore and through the city; look for seams in the dither, flickering rings, shadow crawl.
6. Headless/occluded Chrome throttles `requestAnimationFrame` to ~1 fps; for automated checks keep the tab visible or run a performance trace.

## 9. Known open items

- Water fragment shader is heavy (~1–1.5k ALU/px at DPR 2). Option: render `rt` at DPR 1.
- ~300 draw calls for logs, their ripple decals and the dam stack — could be instanced.
- Shop: flag (`addShopFlag`, animated in `waveFlag`), `S` badge when the boat is within 3.4 of the shop, menu `#shop` (S/E/Enter/click near the shop opens; A/D choose, W/Enter/Space/click buy, S/E/Esc close; elsewhere S is the brake and E shows the wallet; boat frozen while open).
- Economy: `coins` (+100–150 per unloaded log, `spawnCoin` pop-up), `upg` = `{deck, engine, straps}`; `cap()` = `CAP` + deck; engine ×1.2 speed/accel; straps skip pickup when the deck is full. Both reset in `newGame()` via `resetShop()` / `resetTags()`. `DECK` has a 7th slot for the deck-upgrade capsize.
- Tickets (`SHOP_ITEMS`, `drawBody`, `drawStub`): body canvas (art drawn in grey → `dither()` 4×4 Bayer, then text → `crisp(…, 200)`) + stub canvas below the perforation. Purchase → stub gets `.torn` (CSS `tear` keyframes), then `.gone`; body redraws with a SOLD stamp. Ticket pixel scale `shopK` = integer physical px, fitted to the viewport.
