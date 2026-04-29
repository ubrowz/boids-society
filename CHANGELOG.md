# Changelog

## 2026-04-22 / 2026-04-23

### merger.html

**Image morphing for children**
- Children born from bonded pairs are rendered as a true triangle-mesh morph of their parents' images (not just a ghost overlay).
- Bowyer-Watson Delaunay triangulation on named landmarks + 8 fixed corner/edge anchors; per-triangle affine warp via `setTransform` + `clip`; parent A warped to midpoint at full opacity, parent B warped to same midpoint at 50% alpha. Random blend factor 0.4–0.6 per child so siblings differ.
- Fallback: `blendImages` 50/50 composite when parents lack landmarks (e.g., grandchildren whose parents are canvas children with no inherited `imgPts`).

**Named landmark schema**
- Fixed 10-landmark schema: Top, Left eye, Right eye, Nose, Mouth L, Mouth R, Chin, Ear L, Ear R, Neck.
- Landmark editor overlay (morph pts button): single-image view, cycle through all uploaded images and landmarks, click to place, auto-advance to next unplaced landmark. Landmark list on the side shows placed (●) vs unplaced (○) and acts as jump buttons.
- Any two images morph using their shared named landmarks as correspondence (min 3 required; else falls back to blend).
- Persistence: per-filename localStorage write on every click; JSON export/import with filename-based matching. Re-uploading the same filenames auto-restores saved landmarks.

**Generation counter + live display**
- `boid.generation` = 0 for pool/immigrants, `max(parentA.generation, parentB.generation) + 1` for children.
- 84×84 thumbnail right of the slider column shows a random boid from the oldest alive generation; refreshes every 15 s of simulation time. GEN N label below.
- Click the thumbnail to open a zoomed overlay (up to 75% viewport, capped at 640 px); click the overlay anywhere to dismiss.

**Data model changes**
- `maleImgs` / `femaleImgs` changed from arrays of `Image` to arrays of `{img, name, pts}`.
- Added `boid.imgPts` field (landmark map reference for pool boids; null for morphed/blended canvas children).
- Added `boid.generation` field.

**Bug fixes (morphing)**
- Fixed Bowyer-Watson super-triangle orientation: was CW, in-circle test requires CCW — caused triangulation to always return zero triangles, producing blank child images. Swapped two super-triangle vertices.
- Fixed `blendImages` aspect ratio: was hardcoded to 256×256 square, distorted non-square parents. Now computes output dimensions from average parent aspect (matches `morphImages`).
- Fixed cumulative transparency over generations: sub-pixel gaps from triangulation anti-aliasing caused the second warp pass (at α=0.5) to paint transparent-to-mid instead of mid-over-opaque. Now pre-fills output canvas with parent A stretched before the triangulated passes.

**Earlier iteration (superseded)**
- First attempt used a single pair-based point correspondence (one set of corresponding points between the first ♂ image and first ♀ image, placed side-by-side). Didn't scale beyond one image per sex and was replaced with the named landmark schema.

## 2026-04-21 / 2026-04-22

### pics.html

**Visuals**
- Males now render as tadpoles: circle head + tapered bezier tail pointing opposite velocity direction. Females remain circles.
- Bond line flashes bright red (4.5 px) for 2 seconds at the moment a child is spawned (`spawnFlashPairs` map).
- Diseased boids show a black spot (radius × 0.38) at their center.

**Disease system**
- Infect button: picks a random healthy boid and infects it.
- Transmission via physical overlap after `DISEASE_THRESHOLD` cumulative hits; per-pair cooldown 1.5 s.
- Bonded partners transmit at 3× cooldown (no overlap required — slow trickle).
- Recovery after `DISEASE_DURATION` seconds → permanent immunity.
- R-factor: rolling mean of `infectCount` over last 30 recovered/dead infected boids; displayed live.
- Two sliders added: **infect. hits** (1–10, default 3) and **infect. dur.** (5–60 s, default 20 s).

**Sexual orientation**
- `isGay` flag assigned at birth from **gay %** slider (0–50%, default 10%). Singles take precedence (never also gay).
- Gay boids bond only with gay same-sex boids; straight boids only with straight opposite-sex boids.
- `hasValidPairs` and immigration exclude gay boids. Immigrants always spawn straight.

**Genetic diversity readout**
- `genomeDiversity()` computes mean pairwise genetic distance across all living boids.
- Displayed as **div** (0.00–1.00) in status readout.

**UI**
- Two-column layout: sliders on the left, indicators/buttons on the right.
- Slider row gap increased from 2 px to 8 px.

**Simulation**
- 15-second nearness warmup ramp: strength scales from 0 to full over first 15 s so initial clustering is visible. No long-term effect.

### pics-design.md
- Created from boids-design.md, updated to accurately describe pics.html.
- Updated for: tadpole males, sexual orientation system, disease system (transmission, R-factor, bonded-partner trickle), genetic diversity readout, two-column UI, warmup ramp.

## 2026-04-20

### boids.html / pics.html / merger.html

**Bug fixes**
- Fixed `localPressure` normalization: was dividing by `boids.length - 1` (total population), now divides by `TARGET_POP * 0.4` (fixed density threshold). Previous formula caused larger sensing radius to paradoxically weaken attraction.
- Fixed isolated boid attraction: when no neighbours in sensing radius, boid now steers toward global flock centroid at strength 1.0 instead of doing nothing.
- Fixed bond line thickness: `ctx.lineWidth` was being reset to 1.2 after the fertile/non-fertile branch, making the fertile-pair amber line the same thickness as non-fertile lines. Now fertile = 3.5 px, non-fertile = 1.5 px.

**Canvas / sizing**
- Canvas now uses full viewport width; height fills remaining space after UI panel.
- `SENSING_RADIUS` now scales dynamically with canvas size (`Math.min(canvas.width, canvas.height) * 0.3`) and recalculates on window resize.

**Simulation changes**
- `family size` slider changed from integer steps (step=1) to decimal (step=0.1), allowing values like 1.7 or 2.4.

**boids-design.md**
- Fully rewritten to match boids.html (HTML is point of truth). Added: singles, divorce, family size, popPressure signal, immigration toggle. Removed: fertility window sliders (now hardcoded). Updated: slider ranges, LIFESPAN_SCALE_MAX, breeding formula, Arm 1 pressure description, visual key.

### pics.html (new)
- Stripped all per-boid visual indicators: age ring, bond ring, cyan density ring, sex symbols, immigrant dot.
- Removed hit-progress arcs.
- Bond lines kept; fertile M+F pairs draw at 3.5 px, non-fertile at 1.5 px.

### merger.html (new)
- Created by merging pics.html (full simulation logic) with aquarium.html (image rendering).
- Per-sex image pools: upload any number of ♂ or ♀ images; each boid is assigned a random image from its pool at birth and keeps it for life.
- Images preserve natural aspect ratio.
- Wiggle animation per boid (sine-wave rotation, adjustable via slider).
- Falls back to colored circles if no images are loaded.
- Background color picker.
- `RADIUS` increased to 40 (images render at 140 px).
- Glow removed.
- Direction flip removed.
- `BOND_DIST` increased 45 → 120 and `BASE_NEARNESS` reduced 4.0 → 2.0 to spread the flock and prevent image overlap.
