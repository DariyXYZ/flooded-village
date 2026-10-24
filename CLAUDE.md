# Agent instructions

Before editing anything, read **[ARCHITECTURE.md](ARCHITECTURE.md)** — especially section 7 (invariants).

Short version:
- The whole game is `index.html` (no build). Deploy = push to `main` (GitHub Pages).
- `sdWater`, `shoreWarp`, `noise2/vnoise`, `hash` exist in **both JS and GLSL** — change both or neither.
- The alpha written to the render target is a **material-class flag** for the one-bit post-process (1 = outlined, .3 = no outline, .1 = reflection, 0 = water). Pick it deliberately.
- Water shader uniform budget must stay **≤ 224 vec4** (mobile GPUs go black otherwise).
- Anything created per game must be disposed in `newGame()`.
- Don't remove shadow texel snapping in `followCam()`.
- Test `/`, `?test`, `?ending`, and pressing R, with the console open.
