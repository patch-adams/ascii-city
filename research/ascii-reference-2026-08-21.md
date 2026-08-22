# ASCII / text-mode rendering reference — 2026-08-21

Research pass for the ASCII raycast corridor (first-person `<pre>` Wolfenstein, near-dark amber-on-black, doors opening into other pages). Goal: the most impressive real-time ASCII work on the web, and the concrete techniques worth porting.

Searches run: ~30 (renderers, games, technique articles, input UX). Fetched and read: ~25 sources. Where a source would not load (ASCIICKER HN thread, Neowin ASCII-city article, kiedtl particle post) that is noted; nothing below is invented from a 403.

---

## Part 1 — The finds

### 1. javidx9 / OneLoneCoder — CommandLineFPS
- URL: https://github.com/OneLoneCoder/CommandLineFPS (JS port: https://github.com/timurridjanovic/command-line-fps-javascript)
- What: the canonical text-mode raycast corridor. A console window, 120x40 cells, walls shaded by distance, floor shaded by row, wall edges drawn as silhouettes.
- Techniques (verbatim from source):
  - Wall shade by distance, four block glyphs + space:
    ```c
    if (fDistanceToWall <= fDepth / 4.0f)      nShade = 0x2588;  // █
    else if (fDistanceToWall < fDepth / 3.0f)  nShade = 0x2593;  // ▓
    else if (fDistanceToWall < fDepth / 2.0f)  nShade = 0x2592;  // ▒
    else if (fDistanceToWall < fDepth)         nShade = 0x2591;  // ░
    else                                       nShade = ' ';
    ```
  - Floor shade by screen row (b = 0 at horizon, 1 at bottom): `#` `x` `.` `-` ` ` at b < .25 / .5 / .75 / .9. Ceiling is blank. This is why it reads as a corridor: the floor ramp carries the perspective.
  - Wall-boundary detection: for the tile the ray hit, cast a vector from each of its corners back to the eye, take `dot = (eyeX*vx + eyeY*vy)/d`, sort by distance, and if `acos(dot) < 0.01` for the two nearest corners the column is a boundary -> `nShade = ' '`. Cheap silhouette lines at every wall corner (and every door frame) with zero edge-detection pass.

### 2. Andreas Gysin — play.core / play.ertdfgcvb.xyz
- URL: https://play.ertdfgcvb.xyz , manual https://play.ertdfgcvb.xyz/abc.html , src https://github.com/ertdfgcvb/play.core
- What: the reference ASCII live-coding playground. Shader-model: one `main(coord, context, cursor, buffer)` function runs per cell and returns a char or `{char, color, backgroundColor, fontWeight}`. ~40 demos (SDF circles, plasma, doom flame, camera input).
- Techniques:
  - Two renderers, same buffer: `text` (DOM) and `canvas`. The text renderer run-length merges adjacent cells with equal style into one `<span>`, diffs each row against a back buffer with `isSameCell()` and only rewrites `innerHTML` of rows that changed. His note: innerHTML is "remarkably" faster than createElement. And: "Frequent horizontal changes in weight, color or background will slow down the rendering considerably." So if we stay in DOM: keep colour changes as horizontal runs, not per-cell noise.
  - The canvas renderer is the naive path (fillRect bg + fillText per cell, font set per cell) — it is explicitly unbatched. It is what we should beat, not copy.
  - `fontWeight` (300/400/700) as a third channel next to char and colour. Bold as "near", light as "far" is a free depth cue we are not using.

### 3. Alex Harri — "ASCII characters are not pixels"
- URL: https://alexharri.com/blog/ascii-rendering
- What: the best current deep dive on choosing glyphs by *shape* rather than brightness.
- Techniques:
  - Shape vectors: sample each cell with six circles (left/right x top/middle/bottom), build a 6-D vector per glyph offline and per cell at runtime, pick nearest glyph by Euclidean distance. Characters then follow edges instead of just averaging them.
  - Directional contrast enhancement: 12 extra sample circles reach into neighbouring cells; each inner component is boosted by the max of its affecting outer circles. Silhouettes sharpen without a Sobel pass.
  - Perf: k-d tree for nearest glyph (~100x over brute force); quantise vector components to 5 bits and cache by packed key (`key = (key << 5) | q`); six-pass GPU pipeline for the sampling.
  - Simple ramp he starts from: `" .:-=+*#%@"` with `index = floor(lightness * (CHARS.length-1))`.

### 4. Acerola — AcerolaFX ASCII (+ humanbydefinition's p5.js port)
- URL: https://github.com/GarrettGunnell/AcerolaFX/blob/main/Shaders/AcerolaFX_ASCII.fx ; https://github.com/humanbydefinition/p5js-edge-detection-ascii-renderer ; canvas port https://github.com/mokshitagupta/ascii-js
- What: the "I Tried Turning Games Into Text" pipeline; the look everyone now copies.
- Techniques:
  - Fill pass: luminance quantised to 10 levels, `floor(lum*10)-1)/10`, looked up in an 80x8 fill atlas (10 glyphs x 8px). Exposure and attenuation as `pow(lum * _Exposure, _Attenuation)` before quantising.
  - Edge pass: Difference of Gaussians (sigma 2.0, scale 1.6, kernel 2, threshold 0.005, `D = blur1 - tau*blur2; D = D >= thr ? 1 : 0`) then Sobel for direction; direction binned into 4 classes (vertical `absTheta<0.05||>0.9`, horizontal `0.45..0.55`, two diagonals) -> glyphs `| - / \`.
  - Per 8x8 tile histogram of edge directions; only draw an edge glyph if the winning direction count > `_EdgeThreshold` (default 8 of 64). Kills speckle, keeps lines coherent.
  - Depth fog: `fog = exp2(-(k * max(0, z - offset))^2)` with `k = _DepthFalloff*0.005/sqrt(ln2)`; blend toward background. Exponential-squared, not linear — this is what makes the far end of a corridor sink into black convincingly.

### 5. Brogue CE — Light.c
- URL: https://github.com/tmewett/BrogueCE/blob/master/src/brogue/Light.c
- What: the most-admired coloured lighting in any ASCII game.
- Techniques (verbatim):
  - Per-light colour with randomness = flicker, computed per paint:
    `colorComponents[0] = randComponent + light->red + rand_range(0, light->redRand);` (same for g, b; `randComponent` is shared across channels so the light brightens/dims as a whole plus a small hue wobble).
  - Radial falloff to a floor, not to zero: `lightMultiplier = 100 - (100 - fadeToPercent) * sqrt(dx*dx+dy*dy) / radius;`
  - Additive accumulation per cell per channel: `tmap[i][j].light[k] += colorComponents[k] * lightMultiplier / 100;` Several lights simply sum; the cell's glyph colour is then (base colour * light) per channel.
  - The player's own "miner's light" radius shrinks under darkness by a cubic: `fraction = f*f*f` (and halves in water). Cheap "your eyes adjust / the hall gets darker" control.
  - `VISIBILITY_THRESHOLD = 50`: cells below that summed light are not drawn at all. A hard cutoff below which we print nothing reads as true dark.

### 6. Cogmind — FOV aesthetics + ASCII particle engine
- URL: https://www.gridsagegames.com/blog/2016/04/fov-aesthetics/ ; https://www.gridsagegames.com/blog/2014/03/particle-effects/ ; https://www.gridsagegames.com/blog/2014/04/making-particles/
- What: the richest ASCII FX vocabulary in a shipped game.
- Techniques:
  - Out-of-view cells: desaturate with `r=g=b = 0.3r + 0.59g + 0.11b`, then tint toward the theme colour. Newly revealed cells fade in over **400 ms on a sine curve from RGB(0,48,0)** (a dark version of the theme colour) to their true colour. Direct port: when a door opens, the cells behind it sine-fade from dark amber to lit over ~400 ms.
  - Particles are defined in external text files, hot-reloadable, with a template system that generates recoloured variants from one definition ("a recolored variant of any given style can be created by adding a single line of script"). 7 ballistic, 12 laser, 4 plasma, 5 rocket, 24 explosion styles.
  - Style-by-type rule: thermal effects use **only punctuation**, full alphanumerics are **reserved for EM weapons**, power level maps to colour (gray/brown -> orange/red) and to amount of smoke/flash. A glyph-vocabulary-per-category rule is exactly how to make doors distinct.

### 7. Stone Story RPG — ASCII animation tutorial
- URL: https://stonestoryrpg.com/ascii_tutorial.html
- What: the best hand-animated ASCII in games; art made frame by frame as text, rendered with custom shaders in Unity.
- Techniques:
  - "Subtractive animation": draw the full keyframe, then produce in-betweens by *removing* characters rather than redrawing. A door opening can be 4–6 frames of the same frame art with glyphs stripped away.
  - Character vocabularies: block `█` for solids; `~ ! ^ ( ) ´ ‾ ¡ ·` for outlines; `o O v V T L 7 U c C x X` for weight variation; `| \ .` spaced unevenly as anti-aliasing; `_` and `\` rows below a form as its shadow.
  - Hard rule: monospace at a 1:2 cell aspect or the art distorts.

### 8. ASCIICKER (Gumix)
- URL: http://asciicker.com/y0/ (Hackaday: https://hackaday.com/2020/01/01/explore-this-3d-world-rendered-in-ascii-art/)
- What: a full 3D ASCII MMO in the browser, colours, water reflections, thin fences, rotating camera.
- Techniques: C++ rasteriser compiled to WASM writes a colour-cell buffer; a tiny pure-JS WebGL renderer "plops colored ascii" from that buffer. The split (CPU/WASM decides char+fg+bg per cell; GPU only blits glyphs) is the architecture for high-res ASCII at 60 fps. The HN thread (details on shading) returned 429 and could not be read.

### 9. textmode.js + textmode.filters.js (humanbydefinition)
- URL: https://github.com/humanbydefinition/textmode.js , docs https://code.textmode.art , editor https://editor.textmode.art
- What: the modern "ASCII engine" library: WebGL2, instanced glyph rendering, dynamic font atlas from TTF/OTF/WOFF, layers with blend modes and opacity, custom GLSL ES 3.00 materials and filter shaders, offscreen framebuffers; can textmode-convert any existing `<canvas>`/`<video>`.
- Techniques: one instanced draw for the whole grid (per-instance char index + fg + bg); glyph atlas rebuilt once per font; post filters (glow/blur/CRT) as GLSL over the composited grid. If we ever leave the `<pre>`, this is the shortest path to a real renderer.

### 10. three.js AsciiEffect and DeoVolenteGames/ascii-renderer
- URL: https://threejs.org/examples/webgl_effects_ascii.html ; https://github.com/DeoVolenteGames/ascii-renderer
- What: AsciiEffect reads back the WebGL frame and writes a DOM table every frame with charset `' .:!~*=%@#'` — known slow. The DeoVolente alternative overlays a *static* semi-transparent SVG glyph mask over the live WebGL canvas: "frame rate is drastically improved" because the text never changes; only what shows through it does.
- Technique to steal: the static-mask trick inverted — keep a fixed glyph layer and animate light *under* it (a colour canvas under a `mix-blend-mode` text layer) for door glow without re-laying-out text.

### 11. Shadertoy "Ascii Art" (movAX13h) + Codrops OGL ASCII shader
- URL: https://www.shadertoy.com/view/lssGDj ; https://tympanus.net/codrops/2024/11/13/creating-an-ascii-shader-using-ogl/
- What: fontless ASCII in a fragment shader.
- Techniques: each glyph is a 25-bit int read as a 5x5 bitmap:
  ```glsl
  float character(int n, vec2 p) {
    p = floor(p*vec2(-4.0,4.0) + 2.5);
    if (clamp(p.x,0.,4.)==p.x && clamp(p.y,0.,4.)==p.y) {
      int a = int(round(p.x) + 5.0*round(p.y));
      if (((n >> a) & 1) == 1) return 1.0; }
    return 0.0; }
  float gray = 0.3*col.r + 0.59*col.g + 0.11*col.b;
  int n = 4096;                 // .
  if (gray > 0.2) n = 65600;    // :
  if (gray > 0.3) n = 163153;   // *
  if (gray > 0.4) n = 15255086; // o
  if (gray > 0.5) n = 13121101; // &
  if (gray > 0.6) n = 15252014; // 8
  if (gray > 0.7) n = 13195790; // @
  if (gray > 0.8) n = 11512810; // #
  col = col * character(n, p);  // keep hue, glyph is the mask
  ```
  Sample every 16 px, render glyph into an 8x8 sub-grid. Useful as a fallback glow layer or a door "screen" effect.

### 12. Codrops — Efecto (real-time ASCII + dithering, 2026-01)
- URL: https://tympanus.net/codrops/2026/01/04/efecto-building-real-time-ascii-and-dithering-effects-with-webgl-shaders/
- What: 8 ASCII styles (standard, dense, minimal, blocks, braille, technical, matrix, hatching), CRT stack, dithering.
- Techniques: procedural 5x7 glyphs; `brightness = dot(rgb, vec3(0.299,0.587,0.114))`; Floyd-Steinberg on CPU (weights 7/16, 3/16, 5/16, 1/16) because error diffusion is sequential, ASCII on GPU; CRT curvature
  ```glsl
  vec2 c = uv*2.0-1.0; float d = dot(c,c); c *= 1.0 + curvature*d; uv = c*0.5+0.5;
  ```
  and bloom described as "bright pixels glow into dark ones, softening harsh edges while keeping the dithered texture" — bloom *after* quantisation so the grain survives.

### 13. Adam Sawicki — ASCII art in a pixel shader (volume texture)
- URL: https://asawicki.info/news_1277_ascii_art_in_pixel_shader
- Technique: a 3D texture whose XY is the 8x8 glyph cell and whose Z is luminance; slices go from blank black, through dark-gray `.` `,` `:`, to white `#` on light gray, to blank white. The shader does one `tex3D(uv.xy, luminance)` lookup with point filtering and a -0.5 texel bias. In 2D-canvas terms: a glyph atlas indexed by (char, shade) so tint is baked, not computed per cell.

### 14. Paul Bourke — character ramps
- URL: https://paulbourke.net/dataformats/asciiart/
- 70-char: `$@B%8&WM#*oahkbdpqwmZO0QLCJUYXzcvunxrjft/\|()1{}[]?-_+~<>i!lI;:,"^`'. `
- 10-char: ` .:-=+*#%@`
- Notes: apparent density varies by typeface, so measure the actual font; sample half as often vertically (or use the known font aspect) or everything stretches.

### 15. Xor — GM Shaders Mini: CRT
- URL: https://mini.gmshaders.com/p/gm-shaders-mini-crt
- Techniques with values: phosphor mask (`ind = mod(floor(subcoord.x),3)`, mask colour `vec3(ind==0,ind==1,ind==2)*3`, cell border `1 - cell_uv^2 * MASK_BORDER`); pulse `color *= 1 + 0.03*cos(pixel.x/60 + t*20)`; curvature `uv *= 1 + (dot(uv,uv)-1)*CURVATURE` (0–0.1); vignette `pow(edge.x*edge.y, VIGNETTE)` with `edge = max(1-uv*uv, 0)`; aberration by sampling G at an offset.

### 16. Mouse-look / touch UX
- URLs: https://web.dev/articles/disable-mouse-acceleration ; https://castle-engine.io/wp/2026/08/02/pointer-lock-mouse-look-on-the-web-plus-a-big-refactor-and-windows-improvements/ ; https://www.aaronbell.com/mobile-touch-controls-from-scratch/ ; https://web.dev/articles/pointerlock-intro
- Rules:
  - Request lock only inside a transient user activation (click/keydown). Ask for raw input and fall back:
    ```js
    const p = el.requestPointerLock({ unadjustedMovement: true });
    if (p) p.catch(e => { if (e.name === 'NotSupportedError') el.requestPointerLock(); });
    ```
    (Chromium only for unadjusted; Firefox/Safari ignore the option.)
  - On `pointerlockchange` -> null, do **not** immediately re-request; show a "click to look" overlay. Escape is the user's exit.
  - Use `movementX` deltas only. Sensible default for a text grid: yaw = movementX * 0.0025 rad (≈ 0.14°/px), clamp pitch if any, expose a 0.5x–2x multiplier. Ignore the first event after lock (browsers emit a large jump).
  - Mobile: no pointer lock. Split the screen: left half = move (dynamic joystick that appears under the first touch), right half = drag-to-look. Track touches by `identifier`; `touchmove` listener on `document` with `preventDefault` so the drag survives leaving the element; small dead zone in the joystick centre so the thumb never has to lift.

### 17. Minor but useful
- ASCII Maze (itch, https://jfkeo1010etc.itch.io/ascii-maze): 80x40 raycaster at 1:1.5 cell aspect with mouselook; uses *different ASCII textures per distance band* for walls (commenters found it busy — a warning about texture noise far away).
- Codrops "Sketching the Impossible" (https://tympanus.net/codrops/2026/06/11/sketching-the-impossible-a-3d-portfolio-built-without-a-single-3d-model/): doors along an infinite corridor; "auto-glance" head-turn toward a door as you approach; click = align camera in front of door, lazy-load the room, handle turns, door swings, camera flies through; reverse on exit; arrow/space/PageUp-Down keyboard nav. The auto-glance is the UX idea to copy.
- Neowin "walkable 3D ASCII city" (https://www.neowin.net/news/developer-builds-a-fully-walkable-3d-city-entirely-out-of-ascii-characters/): article 403'd; search summary says near objects are drawn with larger, brighter glyph groups and far ones smaller and darker — depth-keyed glyph *size* is in the wild.
- Dwarf Fortress / Caves of Qud: nothing technique-level surfaced beyond Qud's weighted tile-colour mixing; skipped.

---

## Part 2 — Ranked: the 8 techniques most worth stealing

Ranked by payoff for this build. Each is tagged (a) richer corridor, (b) distinct doors, (c) performance.

1. **Split the cell into two channels: glyph = surface, colour = light** — (a)(b)
   Glyph comes from a distance/texture ramp (javidx9's `█▓▒░ ` by distance band, floor `#x.- ` by row, Bourke's 10-ramp for fine detail). Colour comes from a Brogue-style light map: every door is a light source, lights sum additively per cell per channel, falloff `100 - (100-fadeTo)*dist/radius`, shared-random flicker `base + rand(0,redRand)`. Amber palette = one hue, light only changes value. This single separation is what makes Brogue look lit and AsciiEffect look flat.

2. **Exponential-squared depth fog + hard visibility floor** — (a)
   `fog = exp2(-(k*max(0,z-offset))^2)` (Acerola), blended toward black, and below Brogue's `VISIBILITY_THRESHOLD` print nothing. The far end of the hallway becomes a real dark, not a gray `░` band.

3. **Wall-boundary rays for silhouettes, then Acerola direction glyphs** — (a)(b)
   Free first step: javidx9's corner-ray `acos(dot) < 0.01` test marks every wall corner and door jamb; draw `|` there (or a bright amber `|` on the lit side). Second step if wanted: Sobel the depth buffer (not the image), bin into `| - / \`, and only place an edge glyph when the 8x8 histogram winner > 8 of 64. Door frames get crisp outlines; the corridor stops looking like a blur of density.

4. **Pre-rendered glyph atlas, drawImage per cell, no fillText in the loop** — (c)
   Render the charset once per (glyph x shade-level) into an offscreen canvas (Sawicki's luminance-as-Z lookup, done in 2D: atlas index = char*LEVELS + shade). Per frame: one `drawImage` per cell, `ctx.font` never touched. Set `canvas.width = cols*cw*dpr` and `ctx.scale(dpr)` once. Keep the `<pre>` only as a hidden/accessibility mirror or for the low-res fallback. If the DOM path must stay: Gysin's rules — run-length merge equal-style spans, diff per row, rewrite only dirty rows via innerHTML. Realistic target: 240x90 cells at 60 fps on canvas.

5. **Ordered dither before quantising the ramp** — (a)
   Add a 4x4 Bayer threshold (scaled to ±half a ramp step) to luminance before `floor(lum * (N-1))`. Kills the contour bands that 5-level ramps produce on a lit wall, and at 1:2 cell aspect reads as texture, not noise. Optional slow time-jitter of the matrix offset (1 cell per ~250 ms) gives a phosphor shimmer without per-frame flicker.

6. **Per-door glyph vocabulary + light profile + particle emitter** — (b)
   Cogmind's style-by-category rule applied to doors: each door gets (i) a character set — punctuation-only, alphanumeric, block/braille, box-drawing; (ii) a light colour/radius/flicker profile (steady, candle, strobe, breathing); (iii) a tiny text-defined particle emitter (glyph sequence over life, spread, speed, lifetime, colour ramp) leaking from the threshold — motes, drips, static. Define them in JSON, hot-reloadable, and generate variants by recolouring (Cogmind's template trick).

7. **Door-open as subtractive ASCII animation + 400 ms sine reveal** — (b)
   Draw each door as hand-made ASCII frame art (Stone Story vocabulary: `█` body, `~!^()` outline, `_\` shadow row). The opening animation is the same frame with characters subtracted over 4–6 frames. As it opens, cells behind it sine-fade from dark amber (Cogmind's RGB(0,48,0) equivalent, e.g. `#2a1600`) to lit over 400 ms, then navigate. Add the Codrops "auto-glance": a small yaw toward the nearest door when within ~1.5 tiles.

8. **Cheap glow/CRT post pass** — (a)(b)
   On canvas: draw the grid to an offscreen canvas, then draw it back once at low opacity with `ctx.filter = 'blur(6px)'` and `globalCompositeOperation = 'lighter'` (bloom after quantisation, per Efecto). Then Xor's vignette `pow(edge.x*edge.y, v)` as a radial gradient overlay, and optional scanline pulse `1 + 0.03*cos(x/60 + t*20)`. On DOM: one `text-shadow: 0 0 8px currentColor` on the `<pre>` — set once, it is GPU-composited and costs nothing. Use the static-mask idea from DeoVolente for door halos: a colour canvas under a fixed text layer.

Bonus (input, not rendering): pointer lock with `unadjustedMovement: true` + fallback, request only on click, never auto-relock, default 0.0025 rad/px with a user multiplier; mobile right-half drag-to-look, left-half dynamic joystick, touches tracked by identifier.

---

## Sources
- https://github.com/OneLoneCoder/CommandLineFPS
- https://github.com/timurridjanovic/command-line-fps-javascript
- https://play.ertdfgcvb.xyz/abc.html
- https://github.com/ertdfgcvb/play.core
- https://alexharri.com/blog/ascii-rendering
- https://github.com/GarrettGunnell/AcerolaFX/blob/main/Shaders/AcerolaFX_ASCII.fx
- https://github.com/humanbydefinition/p5js-edge-detection-ascii-renderer
- https://github.com/mokshitagupta/ascii-js
- https://github.com/tmewett/BrogueCE/blob/master/src/brogue/Light.c
- https://www.gridsagegames.com/blog/2016/04/fov-aesthetics/
- https://www.gridsagegames.com/blog/2014/03/particle-effects/
- https://www.gridsagegames.com/blog/2014/04/making-particles/
- https://stonestoryrpg.com/ascii_tutorial.html
- http://asciicker.com/y0/ ; https://hackaday.com/2020/01/01/explore-this-3d-world-rendered-in-ascii-art/
- https://github.com/humanbydefinition/textmode.js ; https://code.textmode.art
- https://threejs.org/examples/webgl_effects_ascii.html ; https://github.com/DeoVolenteGames/ascii-renderer
- https://www.shadertoy.com/view/lssGDj ; https://tympanus.net/codrops/2024/11/13/creating-an-ascii-shader-using-ogl/
- https://tympanus.net/codrops/2026/01/04/efecto-building-real-time-ascii-and-dithering-effects-with-webgl-shaders/
- https://asawicki.info/news_1277_ascii_art_in_pixel_shader
- https://paulbourke.net/dataformats/asciiart/
- https://mini.gmshaders.com/p/gm-shaders-mini-crt
- https://web.dev/articles/disable-mouse-acceleration ; https://web.dev/articles/pointerlock-intro
- https://castle-engine.io/wp/2026/08/02/pointer-lock-mouse-look-on-the-web-plus-a-big-refactor-and-windows-improvements/
- https://www.aaronbell.com/mobile-touch-controls-from-scratch/
- https://jfkeo1010etc.itch.io/ascii-maze
- https://tympanus.net/codrops/2026/06/11/sketching-the-impossible-a-3d-portfolio-built-without-a-single-3d-model/
- https://www.neowin.net/news/developer-builds-a-fully-walkable-3d-city-entirely-out-of-ascii-characters/ (403, summary only)
