# LED Panel Flickering — Notes & Solutions

## Context

Running `pics-anim-pi-pixi.html` on a Raspberry Pi 4 driving HUB75 RGB LED matrix panels.
The question was whether raising the software frame rate would reduce visible flickering.

## Why a Higher Frame Rate Does NOT Help

HUB75-style LED matrix panels have two layers of refresh that are entirely independent of the
software (animation) frame rate:

1. **Row scanning (multiplexing)** — the panel does not light all rows at once; it cycles through
   them rapidly (typically 1/16 or 1/32 scan). The LED driver library (e.g. `rpi-rgb-led-matrix`)
   handles this continuously via GPIO, even between animation frames.

2. **Per-LED PWM** — brightness is controlled by rapidly switching each LED on/off at a high
   frequency. This is also managed by the library, not the canvas frame rate.

Visible flickering almost always originates from one of these lower-level mechanisms.

> **Raising the animation frame rate increases CPU load, which can worsen GPIO timing jitter
> and therefore increase flicker rather than reduce it.**

---

## Root Causes and Fixes

| Cause | Fix |
|---|---|
| GPIO timing jitter (CPU preempted mid-scan) | `--led-slowdown-gpio=4` (try 3 or 4); isolate a CPU core with `isolcpus=3` in `/boot/cmdline.txt` |
| Too many PWM bits → low panel refresh rate | Reduce `--led-pwm-bits` from 11 to 7–8; loses some colour depth but gains a much higher refresh rate |
| Wrong scan mode for the panel hardware | Try `--led-scan-mode=1` |
| High CPU load interfering with GPIO thread | Keep animation fps moderate (30 fps target is fine); avoid pushing toward 60 fps |

---

## Recommended Approach for RPi 4

1. **Isolate a CPU core** for the LED driver:
   - Add `isolcpus=3` to `/boot/cmdline.txt`, reboot.
   - Run the rpi-rgb-led-matrix process with `taskset -c 3 ...` or the library's built-in
     `--led-gpio-mapping` / real-time priority flags.

2. **Tune `--led-slowdown-gpio`** — start at 4, try 3 if stable. Too low → corrupted pixels.

3. **Reduce PWM bits** if flicker persists:
   ```
   --led-pwm-bits=7
   ```
   This trades 11-bit colour depth (2048 brightness levels) for a faster panel refresh cycle.

4. **Check scan mode** — some panels require `--led-scan-mode=1` (progressive vs. interlaced).

5. **Keep animation at 30 fps** — freeing CPU headroom is more valuable than extra frames.

---

---

## Quick Win: Black Background

Setting the canvas background to black (`#000000`) reduces flicker significantly in practice.

**Why it works:** black pixels mean the LED is off — PWM duty cycle is 0. With most of the panel
dark (as in the boids simulation, where only boids, trails and effects are lit), the driver has
far fewer high-duty-cycle PWM cycles to process. Less switching activity → cleaner GPIO timing →
less visible flicker. No configuration changes needed; works immediately.

Use the **background** colour picker in the controls and set it to `#000000`. The setting is
exported to standalone files.

---

## Summary

The flickering is a **hardware driver tuning problem**, not an animation speed problem.
The quickest fix is a **black background** (reduces active PWM load on the panel).
Further tuning via `rpi-rgb-led-matrix` configuration (GPIO slowdown, PWM bits, CPU isolation)
can reduce it further if needed.
