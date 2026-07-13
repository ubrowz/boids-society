# Boids Aquarium

**A little society, behind glass.**

Not another flocking demo. These boids pair up and drift apart, raise families, grow old, catch
epidemics, evolve their colours and get hunted — a whole population living and dying while you watch.
The classic boids model gives you graceful motion; Boids Aquarium keeps that underneath and builds an
entire population biology on top.

**▶ Live demo: [ubrowz.github.io/boids-society](https://ubrowz.github.io/boids-society/)**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Pages](https://img.shields.io/badge/hosted%20on-GitHub%20Pages-121013.svg)](https://ubrowz.github.io/boids-society/)

---

## Most boids simulations are a *swarm*. This one is a *society*.

On top of cohesion / alignment / separation, every number you see is a living demographic:

- **Bonds & families** — boids marry, raise offspring, and divorce.
- **Birth, age & death** — populations grow, age on a **Gompertz** survival curve, and die back.
- **Genetics** — colour is the genome; a shared gene pool evolves, with inbreeding depression.
- **Epidemics** — a contagious disease spreads on an **R number** and can wipe out a crowd.
- **Predation** — predators hunt with an agenda (bump or lock strategies).
- **Depth** — an optional 2.5D z-range with continuous depth sorting.

Under the hood the flock uses a **conservative cohesion spring** (centre-of-mass preserving, so clusters
breathe and roam instead of creeping into a corner) plus **Boltzmann rotational diffusion** — a thermal
"temperature" that keeps the swarm mixing.

## Try it

- **[Open the live site](https://ubrowz.github.io/boids-society/)** — landing page, with an embedded
  live simulation ("Run it live").
- **[Open the dashboard](https://ubrowz.github.io/boids-society/pics-anim-pi-pixi.html)** — the full
  control panel: dozens of sliders (speed, sensing, cohesion, temperature, populations, genetics,
  disease, predators, image scale), toggles, **Save/Load presets**, and video recording. Steer the
  aquarium live in its own tab, or export a standalone copy.
- **[Read the field guide](https://ubrowz.github.io/boids-society/boid-world-field-guide.html)** — an
  illustrated guide to the creatures and their world.
- **[About / colophon](https://ubrowz.github.io/boids-society/about.html)**

## Run it locally

No build, no server, no dependencies — it's plain HTML:

```sh
git clone https://github.com/ubrowz/boids-society.git
cd boids-society
open index.html          # or double-click it, or serve the folder statically
```

Everything (including the PixiJS engine and fonts) is embedded, so it runs fully offline.

## How it's built

- **Rendering:** PixiJS v7 (WebGL), embedded inline — no CDN fetch.
- **Effects & overlays:** Canvas 2D (trails, tails, beaks, bond lines, birth/death FX).
- **Language:** vanilla JavaScript — no framework, no build step.
- **Delivery:** each screen is a single self-contained `.html` file.

The dashboard has three run modes: a **dashboard** (controls only), a **live tank** (opened in its own
tab and steered live from the dashboard), and a **standalone export** — a UI-less, auto-starting kiosk
build that runs fullscreen and fully offline in any WebGL browser.

## Repo layout

| File | What it is |
|------|------------|
| `index.html` | Landing page (with embedded live demo) |
| `about.html` | Colophon / credits |
| `boid-world-field-guide.html` | Illustrated field guide to the boid world |
| `pics-anim-pi-pixi.html` | **The main app** — PixiJS dashboard + live aquarium + exporter |
| `boids-society.html` | An exported standalone build (kiosk / embed) |
| `pics-anim-pi-design.html` | Design & implementation notes for the main app |

Other files in the repo are earlier Canvas-2D iterations and design notes kept for reference.

## License

[MIT](LICENSE) © 2026 Joep Rous.

Questions or ideas? Open an [issue](https://github.com/ubrowz/boids-society/issues) or find me at
[@ubrowz](https://github.com/ubrowz).
