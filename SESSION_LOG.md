# Session Log — 2026-08-11 / 2026-08-12

## What we worked on (pics-anim-pi-pixi.html + the two docs)

3 commits, all pushed. One thread: giving the ripple a light wave, and paying for it.

### 1. The light wave — a verdict that did not survive its constraints
The user remembered deciding, when the ripple was first built, that a *light* ripple would be too
costly. It was worth re-asking, because the reason had changed. What made it expensive was never
the light maths — it is that per-boid geometry means one mesh and one draw call per boid, with no
batching. Under **upright-only and under 40 images that cost is already sunk**: the mesh exists and
already carries its own draw call. Adding shading to it costs no draw calls, no extra program binds
(all meshes share one Program), and a handful of uniform writes.

Three properties are deliberate, and each avoids a trap this project has already been bitten by:
- **Subtractive, never a highlight.** Panel flicker is a PWM-duty problem and a black background
  helps precisely because most LEDs sit at zero duty, so brightening would work against the fix
  already in place. It is also the honest reading — cloth in water is shaded on the far slope.
- **A multiply in the shader, not a tint from JS.** `_drawHex` is where hue already carries
  brightness and where the offset-then-clamp trap lives; a multiply cannot clamp.
- **Driven by the slope, not the height.** The shader takes the `cos` of the same argument the
  geometry takes the `sin` of. That is the difference between a curved surface and a gradient
  sliding across a photo, and it costs a phase shift.

### 2. Two sliders, and one that was declined
Shade started derived from `RIPPLE_AMT` — which kept them automatically in step, but meant the only
way to judge the shading was to change the wave causing it. Decoupled; phase and clock still shared.

Frequency then forced two changes: the constant had to become a **uniform** (baking it would
recompile the program on every drag), and the mesh had to go from 6 quads to 10, because a wave can
only be carried by a grid that samples it.

The user then asked for an **OFF position on the frequency slider** to A/B performance. Declined,
with the reason: `ripple = 0` already does exactly that and does it *better*, because it stops the
meshes being created at all rather than merely stopping the surface moving. A frequency-off would
leave all 40 draw calls in place and report that the wave was nearly free — the wrong conclusion
from a control that looked like the right one.

### 3. The Pi paid for the finer mesh, so the wave moved to the GPU
Raising the segment count multiplied both the per-frame JS trig loop and the buffer upload by 2.5,
and the Pi felt it immediately. The fix was not to trim that cost but to delete it: the
displacement now runs in the **vertex shader**, computed from the rest grid, so the geometry is
write-once — no JS trig, no buffer traffic, seven uniform writes per rippling boid per frame. All
meshes now share one geometry, since nothing writes to it. The Pi should run this better than it
did before the light wave existed.

### 4. Docs
Neither the design doc nor the field guide mentioned the ripple **at all**, so this documented the
whole cloth story rather than amending it. Writing it up turned up three stale claims: the bridge
mirrors status at ~6 Hz (not ~3×, and that rate is the whole argument for the Pi pop-out), Tint L
defaults to 50 (not 85, in three places across both files — and 50 is the *saturated* end, so the
old text had the characterisation backwards), and the mixer has 14 faders (not 12; lead and canon
arrived with the melody and were never written down).

## Key decisions
- **Re-ask a costed-out decision when its constraints have moved.** "Too costly" was correct for the
  general case and wrong for this one; nothing about the maths changed, only what had already been
  paid for.
- **Flag the measurement, not just the request.** The OFF-slider ask had a correct goal and a
  control that would have answered it misleadingly. Saying so was worth more than building it.
- **One commit for the frequency slider and the GPU move**, against the usual one-topic rule: the
  shader hunks interleave, and any split would leave an intermediate that fails at load (the old
  baked literal referencing a deleted constant). Better than fabricating a state that never existed.

## Traps worth remembering
- **`mediump` is genuinely 16-bit on an embedded GPU, and desktop drivers quietly promote it.** An
  unbounded clock is therefore exact on a Mac and degrades on a Pi — the exact shape of "fine here,
  bad there". Four minutes in, the wave's phase was stepping by a quarter radian. Wrap clocks to
  [0,1) on the CPU; whole cycles are a no-op inside sin/cos. Wrap the two clocks **separately** —
  0.8 of a cycle is not a cycle.
- **A uniform shared across shader stages must match in precision, not just type.** Raising only the
  vertex stage to `highp` fails at **link** time, with both sources individually valid — and it
  surfaces as the images silently not drawing. Nothing in the console points at precision. Checking
  that every uniform is declared, initialised and written passes this cleanly, because the fault is
  agreement *between* stages, not within one.
- **Pixi's colour is premultiplied.** Scaling all four channels fades an image out; shading means
  multiplying `.rgb` only.
- **A grid carries N/2 crests at N quads across.** Beyond that a wave collapses into facets, and it
  looks poor well before the wall — which is what caps the frequency slider, not taste.
- **Amplitude and frequency fight.** Amplitude is a fraction of image width regardless of crest
  count, so at high frequency the same amplitude folds the mesh through itself. Real waves get
  shallower as they get shorter; this one does not unless you do it.

## Files changed
- `pics-anim-pi-pixi.html` — light wave, shade/waves sliders, GPU deformation, both shader fixes.
- `pics-anim-pi-design.html` — two new appearance sections, four control rows, three optimisation
  rows, the Pi pop-out, plus the three staleness corrections.
- `boid-world-field-guide.html` — two new sections (cloth, panel orientation).
- `SESSION_LOG.md` — this entry.

# Session Log — 2026-08-09 / 2026-08-10

## What we worked on (pics-anim-pi-pixi.html only)

3 commits, **none pushed**. Two threads: hardware sensors for the LED-panel installation, and
finally explaining why the Pi pop-out has always been unusable.

### 1. Two sensors — proximity and tilt
The panel is getting a proximity sensor (digital, 1 = object close) and an inclinometer. The
browser cannot touch GPIO at all — no Web API, and Chromium is sandboxed away from
`/dev/gpiochip*` — so every approach is the same shape: a bridge process on the Pi owns the pin
and the page subscribes over localhost. Settled on a ~40-line Python bridge serving SSE, with the
page-side listener wrapped so it no-ops when nothing is listening (the same HTML is the public
Pages build). Design written up in `sensor-interfacing-notes.md`, which is **git-ignored** — it is
installation hardware, not part of the site.

Neither sensor is wired yet. The tilt one is, however, already simulatable.

### 2. Panel tilt — a slider now, the inclinometer later
The panel rotates **in its own plane**, portrait ↔ landscape. That mattered more than it looked:
only in-plane rotation is corrected by rotating the images. Had the panel tipped like a laptop lid
instead, gravity's in-plane *direction* would never change — only its magnitude — and counter-
rotating would actively make things worse. Worth being sure which mount you have before reusing
this code.

Implementation was almost free, because "upright" was expressed as a literal `0` in exactly three
places and the tilt is that constant becoming a global. 0° is the panel hanging vertical, which is
how it hangs today, so the default changes nothing. Only images rendered upright are corrected — a
boid drawn at its heading is already right at any angle, since it swims in the tank and the tank
turns with the panel. Three things came along for free: the click hit-test (it reads back
`spr.rotation` rather than recomputing), tank sync, and settings save/load — the last two because
both walk range inputs generically.

### 3. The Pi pop-out — a two-year annoyance with a one-line-ish cause
An exported file ftp'd to the Pi runs smooth; the pop-out tank on the same Pi crawls. The two
documents are generated from the same `outerHTML` and differ only in title, run mode and
auto-disease, so it was never the simulation.

**A pop-out shares a renderer process — and therefore a main thread — with the dashboard that
opened it.** `window.open` + `document.write` into `about:blank` keeps the two documents
same-origin and script-connected, so Chromium puts them together. The export gets a process to
itself with nothing in it, which is the entire advantage. Everything the tank sent home came
straight out of the frame budget: a status mirror at 6 Hz driving ~25 DOM writes into the
dashboard's *visible* control panel, plus a `toDataURL` PNG encode of the pyramid on the render
thread.

Fixed with a **Pi pop-out** button: a tank that sends nothing back. Control still flows in and the
sliders steer it as before — only the telemetry going back is gone, which is the asymmetry that
matters, since inbound messages arrive only when someone touches a control. Confirmed fixed on the
Pi. Also stopped drawing the population pyramid in any document where nobody can see it, the
standalone export included.

## Key decisions
- **Sensor notes stay local.** `.gitignore`d rather than committed — hardware for the installation,
  with no bearing on the public site.
- **Tilt zero is today's orientation.** Defining 0° as the panel hanging vertical makes the whole
  feature a pure addition: the shipped default is byte-equivalent to the old behaviour.
- **Image scale deliberately not tied to tilt.** Upright images have ~32px of vertical room in
  portrait and ~192px in landscape, so one tuned for an extreme may read poorly at the other —
  left alone until it actually looks wrong.
- **The structural fix for the shared process was declined for now.** Giving the tank its own
  process means a blob URL with `noopener` and `BroadcastChannel` instead of `opener`, which breaks
  the "document.write a live copy of myself" trick the whole bridge rests on. Held in reserve; the
  cheap fix turned out to be enough.
- **Two commits, not one.** The tilt slider is a feature and had no business riding along with a
  performance fix. Split via a patch applied to the index, and the intermediate commit was checked
  to be self-consistent before being made.

## Traps worth remembering
- **`updateStatus()` looks like display code and is not.** It computes `_cachedDiversity`, which
  feeds disease and hazard, so stripping it from a lean tank would quietly change how the society
  behaves rather than just what it shows. `drawPopPyramid` is the exact opposite — no side effects
  outside its own canvas — which is why one could be skipped and the other could not.
- **`toDataURL` is not a cheap read.** It is a synchronous GPU→CPU readback plus a PNG encode plus
  a base64 stringify, on the render thread, mid-loop.
- **An existing slider made the decisive test free.** Dragging pyramid refresh from 3s to 10s and
  seeing it barely help ruled out the PNG encode as the dominant cost before a line was written.
- **A window opened from another window is not an independent process.** This is the general
  version of the finding above, and it applies to anything measured in a pop-out: the dashboard's
  repaints are competing with the tank's frames.

## Files changed
- `pics-anim-pi-pixi.html` — panel tilt, Pi pop-out, pyramid gating (+64 / −10 across 2 commits).
- `.gitignore` — keep the sensor notes local.
- `sensor-interfacing-notes.md` — created; **git-ignored, local only**.
- `SESSION_LOG.md` — this entry.

# Session Log — 2026-07-27 / 2026-08-09

## What we worked on (pics-anim-pi-pixi.html only)

20 commits, all pushed. Roughly three threads: finishing the AI conductor, a Pi smoothness
investigation, and a run of visual work driven by what reads on a 192×32 LED panel.

### 1. AI conductor — it plays the melody, and three real bugs
The conductor drove only 6 of the 14 mixer faders and never touched the melody at all. It now
scores each movement: lead and canon levels, both octave transposes and the tempo multiplier. The
canon octave carries most of the character — `0` is a round at the unison (procession), `−12` a dark
answer below (die-off, nocturne), `+24` the parts two octaves apart (diaspora). Added four movements
(nocturne, invention, procession, ascension), a `Counterpoint` style that holds population steady and
re-scores the two voices instead, per-instance jitter so no two readings are alike, a rubato LFO so a
held scene breathes, and a long-form arc that grows contrast over nine minutes.

Then the user watched it run and found three things wrong:
- **Temp was 20× too wild.** `_BOLTZ_ROT = boltzRotSlider/100` is the per-frame gaussian sigma **in
  radians** on each boid's heading, and the slider spans 0–150. A conductor fraction of 0.25 meant
  slider 37 = 0.37 rad = **21° of random kick every frame**, against a user calibration of "slider 2
  is already very restless". Every style was affected; the rubato LFO was independently worse,
  swinging temp ±0.09 rad no matter what the scene asked for.
- **Zen was anything but calm.** Rebuilt as floating: temp ~0, speed slider 1–2, 70–110s scenes. It
  needed three new general per-style knobs — `literal` (skip the arc and the jitter, both of which
  were dragging its near-zero temp back up), `ease`, `lfo`.
- **The climax voice could never speak.** `climaxPopSlider` was being driven as a fraction of its own
  2–2000 range, parking it near 800 while 30 boids swam below. The bell evaluated to ~e⁻²⁹, so the
  level fader animated exactly as scored and the voice was silent. Now derived from the movement's
  population goal, clamped to stay within reach of the observed settling average.

### 2. Pi smoothness — and a structural finding
Six per-frame inefficiencies fixed: trails allocated an object per boid per frame (~18,000
short-lived objects a second — GC churn is what surfaces as *stutter* rather than low framerate); the
boid-count overlay re-parsed its innerHTML every frame; `updateFamilyCycle` built two throwaway
arrays per frame; the lone-emigrant pull was O(N²) and fired hardest exactly when dispersed.

The structural finding matters more than any of them: **the simulation is a hybrid.** Biology runs on
real `dt` (lifespan, breeding, disease, predators) but motion runs *per frame* — `boid.x += boid.vx`
with no time term. So a slower machine covers less ground per real second while ageing and breeding
at the same rate: **the Pi runs a measurably different society from identical sliders.** Making it
`dt`-correct is not a one-liner — relaxation terms need `1-(1-k)^dtScale` (alignment reaches exactly
1.0 at slider 100 and would oscillate under a plain multiply) and random walks need `√dtScale`.

We tried lowering `TARGET_FPS` to 24 for even pacing, then reverted it: rAF only fires on refresh
boundaries, so on a 60Hz panel "24" lands on 20, and wander/temp are applied once per *frame*, so
that made the random walk a third coarser. The real fix was **low-passing the rendered position**,
mirroring the `_drawAngle` lerp the heading has always had. That absorbed wander kinks, separation
pushes and frame-time variance together — measured 71% reduction in frame-to-frame acceleration.
Trails had to be recorded from the filtered path too, or every boid trailed 7.5px behind its own
trail head.

### 3. Colour, depth and the LED panel
- **Colour lock**: pin the society to one hue ±spread instead of rolling the family wheel; the hue
  slider stays live so moving it walks the flock to a new colour over 2–3 generations.
- **Depth → lightness and saturation**: the wheel gives no independent light/dark axis, because *hue
  itself carries brightness* (at identical HSL settings yellow emits 10.3× the luminance of blue —
  L* 86 vs 31). Depth light and depth haze add that axis, ~57 L* of swing.
- **Aura** was the one decoration ignoring the size slider — past IMG_SCALE ≈1.4 the body outgrew it
  and the "halo" became a blob painted over the image.
- **Two arrival fades.** Newborns had a fade but it looked abrupt: a *linear* alpha ramp is
  perceptually front-loaded (half apparent brightness a fifth of the way in), while the same ramp run
  downward is back-loaded, which is why dying always looked graceful. Fixed with a gamma-corrected
  ramp. Then the user reported it *still* popping — a second arrival we had created ourselves:
  acquiring an image. In the upright-only view a body enters the tank by picking up a freed unique,
  not by being born, and `birthTimer` is long since 0 by then.

### 4. Image behaviour and interaction
- **Upright images are unique** — one wearer at a time, released when its wearer dies and picked up
  by the next boid to look, so a face travels through the tank instead of being duplicated.
- **The default-shape stand-in is suppressed** once any upright image is loaded, giving an
  upright-only view of just those photographs.
- **Ripple**: a free travelling wave through the whole image, per-boid phase. Initially scoped to
  upright *images*; the user spotted that the Upright *switch* didn't trigger it, and after agreeing
  he would run under 40 image boids we rescoped it to upright *rendering*. That needed unit-square
  geometry so meshes could pool per boid rather than per image.
- **Click a boid to bring it forward.** First version hit-tested a bounding circle, which felt broken
  with cut-out photos — a click on a near boid's transparent corner "hit" it (a no-op) and shadowed
  the visible boid behind. Now each texture carries a 64×64 alpha mask.
- **The predator is invisible in a photo tank** — it still hunts and kills, and the Jaws ostinato and
  kill snare still play, so the menace is heard rather than seen.

## Key decisions
- **Per-boid geometry is cheap on uniques, ruinous on the flock.** Meshes never batch, so one per
  boid means one draw call each: fine at 40, 624 draw calls at full population. Count is the whole
  story — rotation is nearly free. A hard cap is not a way out: with 624 boids the 41st-nearest looks
  identical to the 40th, so the seam is visible; the cap that exists (120) is an emergency floor for
  the public page, never the operating point.
- **Filter the render, don't fix the simulation.** The separation solver and depth layering still
  inject genuine per-frame discontinuities; the position low-pass hides them. That was the right
  trade for smoothness, and it left `dt`-correctness optional rather than required.
- **Behaviour over labels, once.** When the ripple/Upright inconsistency came up the user first chose
  "leave the behaviour, fix the naming"; raising it again made clear he wanted the effect, and we
  agreed the constraint (<40 image boids) before implementing.
- **The deployment target is a 192×32 HUB75 LED panel**, not a monitor. At that scale silhouette,
  brightness and colour are what read; tail fins, beaks, eyes and mesh subdivision fall below the
  resolution floor. Future visual effort pays off in motion, brightness and colour — not detail.

## Traps worth remembering
- **Offset-then-clamp silently eats half a range.** The depth-light greyscale path passed a base of
  100, so everything nearer than the midplane clamped to white: 8 L* of swing against 36 intended.
- **Duplicate function declarations hoist and the later one wins.** A new `_hueDelta` would have been
  silently overridden by differentiation mode's existing one, which takes its arguments in the
  opposite order — an exact negation of every colour pull. Renamed to `_lockDelta`/`_lockDist`.
- **`_visR` must not come from a mesh's `scale.x`** — the texture size is folded into it, and that
  value feeds both depth-layering separation and the click hit test.
- **Check which window you are looking at.** A "broken" ripple slider cost a debugging round; there
  can be several tank windows open and each caches its code at open time.
- **When the diff is provably innocent, question the original design.** The click bug was real but
  lived in the hit test's *shape*, not in any recent change.

## Files changed
- `pics-anim-pi-pixi.html` — all of the above (+764 / −126 across 20 commits).

# Session Log — 2026-04-22 / 2026-04-23

## What we worked on (merger.html only)

### 1. Blend first, then real morphing
Started with a simple `blendImages` function (50/50 alpha composite of parent images) as the first attempt at "child looks like a mix of its parents." User correctly identified that this is blending, not morphing — blended images are just ghosts of two parents overlapping, without any spatial understanding of matching features.

### 2. First morph attempt — pair-based correspondence (superseded)
Implemented triangle-mesh morphing using Bowyer-Watson Delaunay + per-triangle affine warp via `setTransform` + `clip`. First version used a single pair of point sets: user placed N corresponding landmarks on the first ♂ image and first ♀ image, side-by-side in an overlay. Worked but didn't scale — multiple images per sex would need N×M pair setups.

### 3. Named landmark schema (current)
Replaced pair-based with a fixed named landmark schema: **Top, Left eye, Right eye, Nose, Mouth L, Mouth R, Chin, Ear L, Ear R, Neck** (10 points). Each uploaded image gets its own set of landmark positions. Any two images can then morph using their shared named landmarks as correspondence — scales as N instead of N×M.

**UI**: single-image overlay editor with image carousel (◀ ▶), landmark list sidebar (● placed, ○ unplaced), active landmark highlighted in yellow, click-to-place with auto-advance to next unplaced landmark. Undo/reset/delete controls.

**Persistence**: per-filename localStorage write on every click (auto-restores when user re-uploads the same file). JSON export/import button for portability between devices — filename-based matching.

**Data model change**: `maleImgs` / `femaleImgs` switched from `Image[]` to `{img, name, pts}[]`. Boids got a new `imgPts` field (pool boids carry a reference to their image's pts; morphed/blended canvas children have `imgPts = null` and fall back to blend).

### 4. Three morphing bugs fixed
- **Triangulation returned zero triangles**: Bowyer-Watson's in-circle determinant test assumes CCW triangle orientation. Our super-triangle was ordered `[top, bottom-left, bottom-right]` which is CW in screen coords — so every circumcircle test returned the opposite of the truth, no triangles ever split, and the final "remove super-triangle" filter produced zero triangles. Swapped two vertices to `[top, bottom-right, bottom-left]` (CCW) and triangulation immediately worked. This was the root cause of "did not see a morph" AND "bond line dances around / images transparent" — children had blank canvases as images.
- **`blendImages` aspect distortion**: fallback was hardcoded to 256×256 square, squashing portrait/landscape images. Fixed to compute output dimensions from average parent aspect ratio.
- **Cumulative transparency**: sub-pixel anti-aliasing at triangle clip edges created gaps where the first pass (α=1) missed pixels that the second pass (α=0.5) then painted over transparent background → 50% alpha pixels accumulated at edges across generations. Fixed by pre-filling output canvas with parent A stretched before the triangulated passes.

### 5. Generation counter and live display
- `boid.generation`: 0 for pool and immigrants, `max(parentA.generation, parentB.generation) + 1` for children.
- 84×84 canvas thumbnail right of the slider column showing a random boid from the oldest alive generation.
- Refreshes every 15 s of simulation time (pauses with the sim). Initial update at t=0.
- Click the thumbnail → full-viewport overlay with image at ~75% viewport (max 640 px) + GEN label. Click anywhere on overlay to dismiss. `cursor:zoom-in` / `cursor:zoom-out` hint the interaction.

## Key decisions
- Per-image named landmarks (not pair-specific) — scaling, and semantically cleaner (an image's "left eye" means the same regardless of its partner).
- 10 fixed landmark names, geared toward faces/humanoid subjects. If images are very different subjects (fish, abstract shapes), users can still place landmarks where they like — the names are just labels, the geometric correspondence is what matters.
- localStorage as silent auto-save, JSON as explicit portable export — belt-and-suspenders so landmarks never need to be redone within a session and can be moved between machines.
- Morph output size = average of parent aspect ratios × 256 (largest dim). Stable across generations so long as loaded images share aspect.
- Random blend factor 0.4–0.6 per child so siblings visibly differ.
- Generation thumbnail refreshes with `simClock`, not real time, so pausing the sim pauses the display update.

## Files changed
- `merger.html` — all of the above.
- `CHANGELOG.md` — new entry.
- `SESSION_LOG.md` — this entry.

---

# Session Log — 2026-04-21 / 2026-04-22

## What we worked on (pics.html only)

### 1. Spawn flash
Bond line of a fertile pair flashes bright red (4.5 px) for 2 seconds at the moment a child is spawned. Implemented via `spawnFlashPairs` map (pairKey → seconds remaining), ticked in the main loop.

### 2. Male tadpole shape
Males now render as a tadpole: circle head at boid position with a tapered bezier tail (~2.2× radius) trailing opposite velocity direction. Females remain circles.

### 3. pics-design.md created
Reworked from boids-design.md to accurately describe pics.html. Key differences documented: dynamic sensing radius, local pressure normalisation, no per-boid indicators, tadpole males.

### 4. Disease system
- **Infect button** picks a random healthy boid and infects it
- Transmission on physical overlap after `DISEASE_THRESHOLD` hits (per-pair cooldown 1.5s)
- Bonded partners transmit at 3× cooldown (slow trickle, no overlap required)
- Recovery after `DISEASE_DURATION` seconds → permanent immunity
- **R-factor**: rolling mean of `infectCount` over last 30 recovered/dead infected boids
- Visual: black spot (r×0.38) drawn at center of diseased boids
- Two sliders: infect. hits (1–10, default 3), infect. dur. (5–60s, default 20s)

### 5. Sexual orientation (gay %)
- Each non-single boid assigned `isGay` at birth from **gay %** slider (0–50%, default 10%)
- Gay boids bond only with gay same-sex boids; straight only with straight opposite-sex
- `hasValidPairs` and immigration ignore gay boids (they don't contribute to reproduction)
- Immigrants always spawn straight

### 6. Genetic diversity readout
- `genomeDiversity()` computes mean pairwise genetic distance across all living boids (O(n²), fine for typical population sizes)
- Displayed as **div** (0.00–1.00) in the status readout
- Lets you watch genetic drift collapse diversity over time

### 7. Two-column UI layout
`#ui` changed from column to row. Left column: sliders (`#controls`). Right column: bars, readout, toggles, play button (`#rightCol`). Slider gap increased from 2px to 8px.

### 8. Nearness warmup ramp
`WARMUP_DURATION = 15s`. Nearness strength multiplied by `min(simClock/15, 1)`, so boids wander freely at start and clusters form visibly over ~15 seconds. No long-term effect.

### 9. Presentation content
Drafted a 4-slide PowerPoint outline (non-technical audience, 8 people) covering: what it is, real-world analogies (genetic drift, epidemics, social structure), the four-thermostat feedback loop, and why it matters.

## Key decisions
- pics.html now diverges significantly from boids.html — it has features boids.html does not
- pics-design.md is the authoritative doc for pics.html; boids-design.md stays with boids.html
- Disease bonded-partner transmission uses a separate `'b'+pairKey` namespace in `diseaseCooldown` map to avoid collision with collision-based hits
- Fertility window scales with lifespan (noted as important coupling: lifespan feedback does double duty)

## Files changed
- `pics.html` — all of the above
- `pics-design.md` — created; updated for disease, gay orientation, diversity readout

---

# Session Log — 2026-04-20

## What we worked on

### 1. Memory initialisation
Created the memory system for this project. First memory captured the divergence between boids-design.md and boids.html.

### 2. Design doc sync
Rewrote boids-design.md to match boids.html exactly (HTML is point of truth). Key divergences that were documented and corrected: missing sliders (family size, divorce rate, singles), removed sliders (fertility windows, now hardcoded), changed constants (LIFESPAN_SCALE_MAX, slider ranges), changed breeding formula, updated Arm 1 description to include popPressure.

### 3. Canvas sizing fix
- Canvas now fills full viewport width.
- SENSING_RADIUS made dynamic: `Math.min(canvas.width, canvas.height) * 0.3`, recalculated on resize.

### 4. Two sensing bugs fixed
- **localPressure normalization**: dividing by `boids.length - 1` meant a larger sensing radius increased pressure which weakened attraction — the opposite of the intent. Fixed to divide by `TARGET_POP * 0.4`.
- **Isolated boid dead zone**: `if (count === 0) return` meant any boid with no local neighbours got zero attraction. Fixed with global centroid fallback at strength 1.0.

### 5. Family size slider → decimal
Changed step from 1 to 0.1 so values like 1.7 and 2.4 are possible.

### 6. pics.html — clean visual version
Stripped all per-boid indicators (age ring, bond ring, cyan ring, sex symbols, immigrant dot, hit progress arcs). Bond lines kept and thickness bug fixed.

### 7. merger.html — image-based simulation
Combined pics.html simulation logic with aquarium.html image rendering:
- Per-sex image pools (multiple images uploadable, random assigned at birth)
- Aspect ratio preserved in drawImage
- Wiggle animation with slider
- Background color picker
- Fallback to circles when no images loaded
- RADIUS=40, BOND_DIST=120, BASE_NEARNESS=2.0 for good spacing with large images

## Key decisions
- Kept Canvas 2D throughout (not Three.js) — all simulation logic stays 2D
- Image assigned at birth and kept for life (not re-randomised each frame)
- Design doc updated to match code, not the other way around
- SENSING_RADIUS scales with canvas rather than spawning immigrants near flock

## Files changed
- `boids.html` — canvas sizing, sensing radius, localPressure fix, isolated boid fix, family size slider
- `boids-design.md` — full rewrite to match boids.html
- `pics.html` — stripped indicators, bond line fix
- `merger.html` — created from scratch
- `CHANGELOG.md` — created
- `SESSION_LOG.md` — created
