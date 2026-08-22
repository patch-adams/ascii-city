# The corridor — five passes, 2026-08-21

Snapshot before: `snapshots/corridor-v1-2026-08-21.html`, git tag `corridor-v1`.
After: `index.html` (commit dfbb5ad). Live at https://patch-adams.github.io/ascii-city/

## What changed

**Pass 1 — research.** `research/ascii-reference-2026-08-21.md`. ~30 searches, ~25 sources. The
ones that mattered: javidx9's CommandLineFPS (the canonical text corridor), Brogue's `Light.c`
(additive per-cell light with flicker and a visibility floor), Acerola's ASCII shader (edge glyphs
from a Sobel pass), Andreas Gysin's play.core (per-cell shader model; weight as a third channel),
Cogmind (glyph vocabularies per effect, 400 ms reveal), Stone Story (subtractive animation).

**Pass 2 — mouse.** Drag to look with mouse or finger. Double-click or `L` locks the pointer for
proper FPS look (`unadjustedMovement`, with fallback); `Esc` releases. Scroll wheel walks. A plain
click while alongside a door opens it. Looking up and down shifts the horizon (a raycaster has no
pitch, so this is the honest cheap version). Touch drag multiplies sensitivity ×1.6.

**Pass 3 — the instrument.** One `P` dict is the whole state; the render loop reads it every frame,
so every knob lands on the next frame (the ascii-instrument rule). 27 params in six groups:
resolution (columns, cell width/height, font), the room (fov, wall height, view distance, fog
curve), the light (ramp, brightness, saturation, hue, ceiling-light spacing/power, door bleed),
the glass (glow, glow radius, scanlines, floor grain, dust), the body (walk/turn/mouse speed,
invert, pitch, faces alive, fps readout). Six presets. Persists to localStorage; copy-json to
share a look. `Tab` or the gear.

Under it, the renderer moved from a `<pre>` of text to a canvas glyph atlas: every glyph is
drawn once, white, at the exact cell size; each frame is `drawImage` per non-space cell, then one
`multiply` of a W×H colour map scaled up with smoothing off. That's what makes per-cell colour,
light pools, bloom, and a live resolution slider all cheap. 160×62 runs at ~4.5 ms/frame.

**Pass 4 — doors.** Doors are now two cells wide (a real door's proportions; wide enough on
screen to carry a face). Each of the 18 has its own 34×28 face generator, its own colour that
bleeds into the wall and floor around it and tints the readout, and its own note on approach
(a pentatonic walk up the corridor). Eleven faces move: city windows flicker, breath rings pulse,
depth zooms, film sprockets scroll, the screening screen flickers, presence breathes, the creature
blinks and tracks, the who-door shimmers, ma/ré has waves, the unmarked door is astir.

**Pass 5 — the take.** Below.

## What would make it sing

Ranked by felt difference per hour of work.

1. **Door-open choreography.** Right now opening is a fade to black. It should be: the view eases
   to face the door (auto-glance), the face falls away from the centre outward (subtractive, Stone
   Story), the door's colour floods the corridor, then the cut. Half a day; it's the single biggest
   felt upgrade left.

2. **Sound as space.** Every door already has a pitch. Give each a near-silent drone whose gain is
   your proximity, so walking the corridor plays a scale and you can find a door with your eyes
   shut. Stays under the "only on your own action" rule if it starts with the first step.

3. **Glyph weight as a third channel.** Gysin's move: a second, bolder atlas for near cells. Glyph
   = surface, colour = light, weight = depth. One extra atlas, one `if`.

4. **Real jambs.** Recess each door 0.3 units into the wall so it has edges, then a Sobel on the
   depth buffer to draw `| / \ -` silhouettes at those edges (Acerola). The corridor gets corners
   for the first time.

5. **Dither before quantising.** A 4×4 Bayer on the fog term so far walls dissolve instead of
   banding into rows. Twenty lines.

6. **The corridor breathes.** Light flicker (Brogue's `base + rand`), dust that parallaxes with
   your motion instead of sitting on the screen, a faint hum that changes under each light.

## Toward the playable Sleepy Planet

The corridor is already the right shape: a hub, doors, no labels, a threshold that draws a card.
What turns it from a menu into a world:

- **The map becomes data.** One JSON of cells, doors, faces. Rooms behind doors rendered by the
  same engine, no page load — the hallway becomes one continuous place. The existing pages stay
  as windows (overlays) for the builds that are their own thing.
- **You carry the light.** Replace the ceiling lights with a lantern in your hand. Your pool of
  light is the only light; the corridor beyond it is truly dark. This is the one mechanical change
  that makes it Sleepy Planet rather than a hallway — the lantern is already a motif.
- **The Keeper in the hall.** A glyph body (the Presence creature) that waits at a door, walks
  ahead, turns to look at you. Faces already animate; a body is the same trick at the room scale.
- **The unmarked door runs a passage.** Instead of drawing a random page, it draws a passage from
  the Keeper skill and paints the prompt as a face you read up close. Wooden keys for doors you've
  been through.
- **Witnesses.** A second walker's position over the tailnet (SSE) as another figure in the hall.
  Sleepy Planet is played with others; the corridor should be able to hold two.

## Not done this round, on purpose

Corner silhouettes, Bayer dither, and the open choreography are specced above but not built —
they're pass six. Colour stays restrained by default (bleed 0.8, near-dark ramp): the amber dark
is the identity; the tints are light under the door, not wallpaper.
