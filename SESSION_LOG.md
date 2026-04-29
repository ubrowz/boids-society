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
