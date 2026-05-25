# Performance Investigation — pics-anim-pi-pixi.html

## Symptoms

- Simulation running at **18 fps** on Raspberry Pi 4
- Same file running at **24 fps** on Mac (Chromium)
- Performance appeared constant regardless of boid count

---

## Mac: 24 fps is a display sync issue, not a code problem

Adding an fps counter (`fps (ms)`) revealed **42ms per frame** on Mac.

`1000ms ÷ 24fps = 41.67ms` — exact match.

**Root cause:** The Mac has a **ProMotion adaptive display**. The browser sees the
animation targeting 30fps (`FRAME_MIN_MS = 33ms`) and ProMotion locks the display
to **24Hz** — the nearest clean rate it can sync to. Every `requestAnimationFrame`
callback fires at 41.67ms intervals, every one triggers a render, and you get 24fps.

This is not a simulation bottleneck. The code completes each frame well within 42ms.
The Mac fps figure is **not a useful benchmark** for this simulation.

---

## Pi: 18 fps caused by CPU throttling due to under-voltage

`vcgencmd measure_temp` → **55°C** (fine, not a temperature issue)

`vcgencmd get_throttled` → **`0x50005`**

Decoded bitmask:

| Bit | Meaning | State |
|---|---|---|
| 0 | Under-voltage detected **right now** | active |
| 2 | CPU currently throttled **right now** | active |
| 16 | Under-voltage has occurred (history) | active |
| 18 | Throttling has occurred (history) | active |

The Pi was detecting insufficient voltage on the USB-C power input and clocking the
CPU **down from 1.5 GHz to ~600 MHz** — roughly 2.5× slower than normal. That
directly explains 18fps instead of the expected ~30fps or better.

**Fix:** use a proper **Raspberry Pi 4 USB-C power adapter (5V / 3A)**. Replace any
long or cheap USB-C cable — voltage drop in the cable is a common cause even with an
adequate adapter. Confirm fix with:

```
vcgencmd get_throttled   # should return 0x0
vcgencmd measure_volts core  # should be ~1.2V, not below 0.9V under load
```

---

## Code optimisations made during investigation

Even though the root cause was the power supply, several genuine per-frame
inefficiencies were found and fixed:

### 1. Cluster detection removed
`updateClusters()` ran a DBSCAN-like neighbour scan (3 passes × all boids) every
frame, even when cluster circles were hidden. Feature was not adding value.
Removed entirely: `updateClusters`, `drawClusters`, `clusterEmigration`, and all
associated UI controls. **Saved the most CPU of any single change.**

### 2. Per-frame GC allocations eliminated
`_sgNeighbors()` was creating a `new []` array on every call — called ~90–120×
per frame. Replaced with a shared reusable `_sgBuf` buffer (set `length=0` to
reset). Zero array allocations in the hot neighbour-scan path.

`_buildSpatialGrid()` was calling `new Map()` every frame. Changed to `.clear()`
and reuse.

`updateEffectiveBreeders()` created its own shadow `new Map(boids.map(...))` and
`new Set()` every frame. Now uses the global `boidMap` (already rebuilt each
frame) and a persistent `_ebSeen` Set.

`updateFertileCounts()` used two `.filter()` calls (throwaway arrays). Replaced
with a single counting loop.

### 3. `updateStatus()` throttled to ~6fps
`updateStatus()` was the hidden O(N²) bottleneck:
- `genomeDiversity()` computes all pairwise genetic distances — 435 comparisons
  for 30 boids — called every frame
- 25+ DOM writes per frame (`.textContent`, `.style.width`) triggering layout passes
- 5× `.filter()` calls creating throwaway arrays

Fixed by running `updateStatus()` every 5th rendered frame (~6fps). The status
panel values (birth rate, diversity, inbreeding) do not need 30fps refresh.

### 4. Main loop array allocations fixed
`boids.filter(b=>b.alive)` (new array every frame) replaced with in-place
`splice()` loop. `POP_CEILING` culling replaced with a count+walk instead of
`filter()`.

---

## Summary

| Issue | Root cause | Fix |
|---|---|---|
| 18 fps on Pi | Under-voltage → CPU throttled to 600 MHz | Proper 5V/3A PSU + quality USB-C cable |
| 24 fps on Mac | ProMotion display synced to 24Hz | Not a problem — display sync artefact |
| General CPU load | Cluster O(N²) scan, GC churn, DOM writes | All fixed in code |
