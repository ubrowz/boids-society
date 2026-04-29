# Boids Aquarium (Animated) — Design & Implementation

## Overview

An animated browser-based population simulation built on the classic boids algorithm. Coloured boids wander autonomously inside a bounded aquarium. They can bond, breed, age and die. A multi-arm feedback loop attempts to maintain a stable population around a user-defined target.

The visual layer extends the base simulation with:

- **Motion trails** with two-pass glow rendering
- **Birth rings** at spawn events
- **Morphological genetics** — males carry a `beakGene` (horn length), all boids carry an `eyeGene` (eye size), both heritable and mutable via bit-flip
- **Colour mutation** via 24-bit bit-flip
- **Colour preference** (assortative mating) slider
- **Predators** — autonomous hunters with sensing radius, colour preference, starvation
- **Export** button — generates a standalone full-screen HTML with no UI, auto-start, auto-disease

Target framerate is 30 fps (capped; simulation time is dt-based so it runs correctly at any framerate).

All logic runs in a single self-contained HTML file with no external dependencies.

---

## Boid Appearance

### Males

Rendered as a **filled body circle** + a **forward-pointing tapered horn** (unicorn style).

```
horn tip ──► ●  ──► direction of movement
```

Horn length is determined by `beakGene` (see *Morphological Genetics* below). Base length = `RADIUS × 2.2 × beakMultiplier(beakGene)`.

Both sexes have **two eyes** (filled dark circles with a white highlight dot) placed at the front of the body, spread perpendicular to the direction of movement. Eye radius is determined by `eyeGene`.

### Females

Rendered as a **filled body circle** + two eyes. No horn.

### Both sexes

| State                  | Visual                                                        |
|------------------------|---------------------------------------------------------------|
| Fertile                | Full opacity                                                  |
| Infertile              | 55% opacity                                                   |
| Bonded                 | Gold radial glow around body                                  |
| Diseased               | White spot at body centre                                     |
| Fading in/out          | globalAlpha ramps over BIRTH_DURATION / FADE_DURATION         |
| Trail                  | Two-pass glow + core line behind every moving boid            |

---

## Boid Lifecycle

Each boid progresses through the following states:

```
spawn → fade-in → juvenile → fertile adult → post-fertile → dying → removed
```

| Property            | Description                                                |
|---------------------|------------------------------------------------------------|
| `age`               | Seconds lived since birth                                  |
| `maxLifespan`       | Biological ceiling — fixed at birth, drawn from Normal(BASE_LIFESPAN, 0.2 × BASE_LIFESPAN), floored at 30% of base. Never changes. |
| `effectiveLifespan` | Display-only: current implied mean lifespan given population pressure (= BASE_LIFESPAN × pressureScale). Not used as a death trigger. |
| `birthTimer`        | Fade-in duration (1.5 s)                                   |
| `dying`             | True when age ≥ maxLifespan, or when a per-frame Gompertz hazard roll triggers death; boid fades out over 2 s |

---

## Movement

Each boid moves at a constant speed, direction updated every frame by four competing steering forces applied in sequence:

1. **Wall avoidance** — soft repulsion begins 40 px from any wall; hard bounce at the wall itself
2. **Wander** — a random-walk steering target projected ahead of the boid on a circle, jittered each frame
3. **Nearness** — attraction toward the local flock centre (see Feedback Arm 1)
4. **Bond attraction** — bonded partners stay within 45 px of each other

Speed is user-controlled via a slider (1–10, mapped to 0.45–4.5 px/frame).

---

## Singles

Each boid is assigned a "single" status at birth, drawn probabilistically from the **singles** slider (0–80%, default 20%). Singles never bond. They are drawn at 0.7× the normal radius to visually distinguish them.

---

## Sexual Orientation

Each non-single boid is assigned an orientation at birth:

- **Straight** (default) — can only bond with an opposite-sex non-gay boid
- **Gay** — can only bond with a same-sex gay boid

The **gay %** slider (0–50%, default 10%) sets the fraction of non-single boids born gay. Singles take precedence: a boid that is single is never also marked gay. Gay pairs bond and divorce normally but can never breed (no opposite-sex requirement is met). The feedback loop's deadlock detection and immigration system ignore gay boids, since they do not contribute to reproduction.

Immigrants are always straight, as they are spawned specifically to break reproductive deadlocks.

---

## Bonding

Two unbonded, non-single boids bond after accumulating `BOND_THRESHOLD` collisions (overlapping circles). A per-pair cooldown prevents one slow overlap from counting as many hits.

**Restrictions:**

- A bonded boid cannot bond again until its partner dies or the pair divorces
- A child cannot bond with either parent
- Siblings (same two parents) cannot bond with each other
- Singles never bond
- Gay boids only bond with gay boids of the same sex; straight boids only bond with straight boids of the opposite sex

**Bond threshold interaction with nearness:**
Higher `BOND_THRESHOLD` requires more collisions, so boids need sustained proximity. The nearness strength is scaled upward proportionally (`1 + (threshold - 1) × 0.24`) to compensate. The hit cooldown also shrinks with higher thresholds (`COOLDOWN_FRAMES / √threshold`) so hits accumulate faster.

---

## Divorce

Each frame, each bonded pair has a small chance to split equal to `DIVORCE_RATE × dt`. When a divorce occurs:

- Both partners become unbonded and their hit/children records for that pair are cleared
- Each partner's breed timer is reset with a random offset
- `pairFamilyTarget` is reset to 0 for both

The **divorce rate** slider (0–50%, default 5%) sets the per-second probability.

---

## Sex and Fertility

Each boid is assigned a sex (`M` or `F`) at birth. Sex is not purely random — it is biased toward whichever sex is currently underrepresented in the living population, keeping the ratio close to 50/50.

Fertility windows are fixed fractions of each boid's `maxLifespan` (the biological ceiling fixed at birth):

| Parameter      | Value                |
|----------------|----------------------|
| ♂ fertile from | 20% of maxLifespan   |
| ♂ fertile for  | 70% of maxLifespan   |
| ♀ fertile from | 20% of maxLifespan   |
| ♀ fertile for  | 50% of maxLifespan   |

A boid is fertile only when `age` falls within `[maxLifespan × from, maxLifespan × (from + for)]`. Using `maxLifespan` (not the pressure-varying `effectiveLifespan`) means each boid's reproductive window is a fixed fraction of its own biology and does not shift as population pressure changes.

**Only bonded straight M+F pairs where both partners are currently fertile can produce children.**

---

## Breeding

When a bonded fertile M+F pair is active, a per-pair family target determines reproductive capacity. At bond formation the pair's family target is drawn from `Normal(FAMILY_SIZE, 0.8)`, clamped to a minimum of 0 and rounded to the nearest integer. This is stored on both partners as `pairFamilyTarget`.

The effective maximum children is then:

```
t              = (globalPressure + 1) / 2          # 0..1
desiredMax     = ceil(pairFamilyTarget × (1 − max(t − 0.5, 0) × 2))
effectiveMax   = min(desiredMax, bioCeiling)
bioCeiling     = floor(maxLifespan × F_FERTILE_FOR / PREG_INTERVAL)
```

- At target (t = 0.5): `desiredMax = pairFamilyTarget` (no reduction)
- Overpopulated (t = 1): `desiredMax = 0` (breeding stops)
- At crisis (t = 0): `desiredMax = pairFamilyTarget` (no bonus above the family target)

The **family size** slider (0–6, default 4) sets `FAMILY_SIZE`, the mean of the per-pair target distribution.

The **pregnancy interval** slider (1–20 s, default 1 s) sets the minimum time between births. The actual interval is modulated by pressure:

```
intervalScale  = 3^globalPressure
actualInterval = PREG_INTERVAL × intervalScale × random(0.8, 1.6)
```

At target (`globalPressure = 0`): interval ≈ `PREG_INTERVAL`
At crisis (`globalPressure = −1`): interval ≈ `PREG_INTERVAL × 0.33` (3× faster)
Overpopulated (`globalPressure = +1`): interval ≈ `PREG_INTERVAL × 3.0` (3× slower)

Child counts are tracked **per pair** (not per boid), so surviving partners start fresh with a new partner.

---

## The Feedback Loop

Stability is maintained through three independent feedback arms, all driven by **pressure signals** derived from the current population state. The three pressure components are kept separate to prevent them from interfering with each other.

### Pressure Signals

**Local pressure** (`localPressure`, 0–1)
Measures neighbourhood crowding: the number of other living boids within the sensing radius, normalised against 40% of the target population.

```
localPressure = min(neighbours / max(TARGET_POP × 0.4, 1), 1)
```

The sensing radius is dynamic: 30% of the smaller canvas dimension, so it scales correctly at any window size.

**Global pressure** (`globalPressure`, −1 to +1)
Measures population-level error: the ratio of currently fertile pairs to the expected number of fertile pairs at target population. Negative = underpopulated, zero = at target, positive = overpopulated. Used by Arms 2 and 3.

```
targetPairs    = (TARGET_POP / 2) × min(M_FERTILE_FOR, F_FERTILE_FOR)
fertilePairs   = min(fertile males, fertile females)
globalPressure = clamp(fertilePairs / targetPairs − 1, −1, +1)
```

An **extinction floor** overrides `globalPressure` to −1 when total population drops below 4, ensuring maximum recovery regardless of fertile pair maths.

**Population pressure** (`popPressure`, −1 to +1)
Measures raw headcount error relative to target. Used by Arm 1 only to suppress clustering when the population overshoots the target.

```
popPressure = clamp(boids.length / TARGET_POP − 1, −1, +1)
```

---

### Arm 1 — Nearness (birth rate, indirect)

**Signals used:** `localPressure` and `popPressure`
**Effect:** Controls how strongly boids steer toward each other.

```
combinedPressure = min(localPressure + max(popPressure, 0), 1)
strength = (MIN_NEARNESS + (BASE_NEARNESS − MIN_NEARNESS) × (1 − combinedPressure))
           × thresholdScale
```

The `popPressure` term is only added when positive (overshoot), so when the raw headcount exceeds target it boosts combined pressure and reduces/reverses attraction — dispersing boids even if local density happens to be low.

| combinedPressure | Nearness strength | Effect                                           |
|------------------|-------------------|--------------------------------------------------|
| 0 (sparse)       | High (up to 4.0×) | Boids cluster → more collisions → faster bonding |
| 1 (dense)        | Low (0.05)        | Boids spread out → fewer collisions              |

When `strength` goes negative (repulsion zone), boids steer away from the flock centre.

This arm controls collision frequency, which is the prerequisite for bonding and therefore the entire breeding pipeline. It does **not** directly control lifespan or birth rate — it acts on the *opportunity* to bond.

---

### Arm 2 — Mortality (death rate)

**Signal used:** `globalPressure`
**Effect:** Modulates the per-frame probability of death via a Gompertz hazard rate, further multiplied by two inbreeding penalties.

Each frame, every living boid rolls against:

```
P(die this frame) = hazardRate(boid, scale) × dt
```

where the full hazard is:

```
scale       = LIFESPAN_SCALE_MAX + (LIFESPAN_SCALE_MIN − LIFESPAN_SCALE_MAX) × (globalPressure + 1) / 2
modalAge    = BASE_LIFESPAN × scale
beta        = 3 / modalAge
alpha       = beta × exp(−beta × modalAge)
gompertz    = alpha × exp(beta × boid.age)

hazard      = gompertz × inbreedingHazardMultiplier × smallPopHazardMultiplier
```

Constants: `LIFESPAN_SCALE_MAX = 1.25`, `LIFESPAN_SCALE_MIN = 0.4`

The Gompertz hazard grows exponentially with age (`exp(beta × age)`), producing realistic senescence: young boids have a very low per-frame death probability; old boids have a much higher one. The mode of the resulting age-at-death distribution is pinned to `modalAge = BASE_LIFESPAN × scale`.

| globalPressure   | scale | Modal age at death (base 80 s) | SD ≈      |
|------------------|-------|--------------------------------|-----------|
| −1 (crisis)      | 1.25× | 100 s                          | ~43 s     |
| 0 (at target)    | 1.0×  | 80 s                           | ~34 s     |
| +1 (over target) | 0.4×  | 32 s                           | ~14 s     |

SD ≈ 1.28 / beta. The `beta = 3 / modalAge` constant controls senescence steepness — increase it to tighten the distribution.

**`inbreedingHazardMultiplier`** (1× – 1.5×) — rises as genetic diversity falls below 0.3, reaching 1.5× at near-zero diversity. Models general fitness reduction from inbreeding depression. At the maximum multiplier, the modal age shifts down by `ln(1.5) / beta` seconds (≈ 11 s at base 80 s). See *Inbreeding Depression* below.

**`smallPopHazardMultiplier`** (1× – 1.2×) — rises as effective breeders fall below 50, reaching 1.2× at zero. Captures the trajectory risk of genetic drift in small populations, independent of currently observed diversity. See *50/500 Rule* below.

At worst case (zero diversity, zero effective breeders): total hazard multiplier = 1.5 × 1.2 = **1.8×** baseline — roughly cutting the modal age by a third at default settings.

In addition to the hazard roll, every boid has a hard biological ceiling at `maxLifespan` (fixed at birth from a Normal distribution). Death is certain by that age regardless of the hazard roll.

`effectiveLifespan` (= `BASE_LIFESPAN × scale`) is still computed each frame and displayed in the UI as the current implied mean, but it is not used as a death trigger.

---

### Arm 3 — Breeding rate (birth rate, direct)

**Signal used:** `globalPressure`
**Effect:** Scales both the pregnancy interval and the maximum children per pair.

| globalPressure   | Interval multiplier | Effective max children (family size 4) |
|------------------|---------------------|----------------------------------------|
| −1 (crisis)      | ~0.33× (3× faster)  | 4 (no bonus above family target)       |
| 0 (at target)    | 1.0×                | 4                                      |
| +1 (over target) | ~3.0× (3× slower)   | 0 (breeding stops)                     |

---

### Why three separate signals?

Early versions used a single blended pressure signal for all arms. This caused a critical failure: when population was low and nearness caused boids to cluster, local density increased, which raised pressure, which suppressed recovery — the system sabotaged itself. Separating the signals prevents this:

- Arm 1 responds to combined local density + raw headcount overshoot (spacing behaviour)
- Arms 2 and 3 respond only to the global population-vs-target error
- `popPressure` only suppresses clustering on the upside; it does not accelerate clustering when under target

---

### Equilibrium condition

The system is stable when births equal deaths:

```
births/s = activeFertilePairs × (1 / actualInterval)
deaths/s = populationSize / effectiveLifespan
```

At target population with default settings:

- ~7 fertile pairs, interval 1 s → ~7 births/s
- 30 boids, 80 s lifespan → 0.375 deaths/s

The system has a birth surplus at target, which gives the feedback loop room to self-correct downward through the interval and cap mechanisms when population exceeds target.

---

## Genetic System

### Colour as genome

Each boid carries a 24-bit RGB colour value that functions as its genome. Children receive a blended colour from both parents:

```
childColor = blendColors(parentA.color, parentB.color)
```

After blending, bit-flip mutation is applied independently to colour, beakGene and eyeGene (see *Colour Mutation* and *Morphological Genetics* below).

### Genetic distance preference

When two unbonded boids collide, the cooldown before the next registered hit is modulated by their genetic distance:

```
geneticDistance = euclidean(genome_A, genome_B) / √3    # 0..1
cooldownScale   = 2.0 − 1.5 × geneticDistance
```

Genetically similar boids (distance ≈ 0) have 2× longer cooldowns — they bond slowly. Genetically distant boids (distance ≈ 1) have 0.5× cooldowns — they bond up to 4× faster. This naturally preserves genetic diversity by favouring outbreeding.

### Genetic drift and colour convergence

Without intervention, colour (and genome) converges to a single value over many generations through **allele fixation by genetic drift** — whichever lineage happens to breed successfully first gradually displaces all others. This is visible as all boids converging to the same colour. The bit-flip mutation system introduces occasional dramatic colour jumps that can restart divergence.

### Inbreeding depression

As diversity falls, two independent fitness penalties activate. Both use the same diversity thresholds:

```
t = 1 − clamp((diversity − DIVERSITY_LOW) / (DIVERSITY_HIGH − DIVERSITY_LOW), 0, 1)

DIVERSITY_HIGH = 0.3   # above this: no penalty
DIVERSITY_LOW  = 0.05  # below this: full penalty
```

**Mortality penalty** (`inbreedingHazardMultiplier = 1 + t × 0.5`, up to 1.5×)
Models general fitness reduction — homozygous expression of deleterious recessive alleles shortens lives. Applied as a multiplier to the Gompertz hazard every frame.

**Disease susceptibility penalty** (`diversityFatalityMultiplier = 1 + t × 2.0`, up to 3×)
Models HLA (immune receptor) diversity collapse — a genetically uniform population cannot mount varied immune responses. Applied to `DISEASE_FATALITY` before computing the disease hazard rate. With 10% base fatality, full convergence pushes effective fatality to 30%. The UI shows `→ N%` next to the fatality slider when the effective rate diverges from the base.

The two penalties compound: a diseased boid in a low-diversity population faces both elevated disease kill probability and elevated background mortality.

### 50/500 rule

**Effective breeders** = fertile bonded M+F pairs × 2. This is computed each frame and displayed in the `eff.br.` readout.

The **50/500 rule** from conservation biology states:
- **< 50 effective breeders**: short-term inbreeding depression becomes severe — immediate viability risk
- **< 500 effective breeders**: long-term genetic drift erodes adaptive potential

A third hazard multiplier captures the trajectory risk of small populations, independent of currently observed diversity:

```
smallPopHazardMultiplier = 1 + (1 − min(effectiveBreeders, 50) / 50) × 0.2
```

This reaches 1.2× at zero effective breeders. The `50-rule` readout shows **critical** (< 50), **at risk** (< 500), or **ok** (≥ 500).

---

## Immigration

When immigration is enabled (toggle in the UI), an immigrant boid may be spawned based on combined deficit pressure:

```
popDeficit     = max(1 − boids.length / TARGET_POP, 0)
fertileDeficit = max(−smoothGlobalPressure, 0)
pressure       = max(popDeficit, fertileDeficit)
```

Immigration also triggers on genetic deadlock: when no valid unrelated straight M+F fertile non-single pairs exist. The spawn interval shortens as pressure increases (minimum ~5 s at full crisis).

Immigration is suppressed entirely when population exceeds 120% of target.

Immigrants:

- Arrive with a fresh random colour and genome
- Have no family key and no parent IDs (can bond with anyone)
- Are spawned at mid-fertile-age so they can breed immediately
- Are assigned the sex most needed for balance
- Are never singles, and are always straight

---

## Disease

A contagious disease can be introduced at any time by pressing the **infect** button, which randomly selects one healthy boid and infects it. In exported files, disease occurs automatically on a random 30–90 s timer (`AUTO_DISEASE`).

### Transmission

Healthy boids accumulate exposure hits whenever they physically overlap with an infected boid. A per-pair cooldown (`DISEASE_COOLDOWN = 1.5 s`) prevents rapid accumulation from a single sustained contact. Once hits reach `DISEASE_THRESHOLD`, the healthy boid becomes infected.

```
exposureHits += 1 per qualifying collision with any infected boid
if exposureHits >= DISEASE_THRESHOLD → infected
```

### Fatality

Each frame while a boid is diseased it faces an additional death roll independent of the Gompertz hazard:

```
effFatality = min(0.9999, DISEASE_FATALITY × diversityFatalityMultiplier)
h_disease   = −ln(1 − effFatality) / DISEASE_DURATION
P(die from disease this frame) = h_disease × dt
```

Deaths are spread naturally across the illness period — some boids die early, some late, survivors recover.

**Diversity link:** `effFatality` is the base fatality scaled by `diversityFatalityMultiplier` (up to 3×). See *Inbreeding Depression* above.

### Recovery and immunity

Infected boids that survive the full `DISEASE_DURATION` recover and become permanently immune.

### R-factor

The R-factor is the rolling mean of `infectCount` over the last 30 recovered or deceased infected boids. It represents the average number of secondary infections caused by one infected individual and is displayed live in the status readout.

---

## Motion Trails

Every boid stores a fixed-length ring buffer of past positions (`trail[]`). Each frame the current position is appended and the oldest position dropped when the buffer exceeds `MAX_TRAIL`.

Trails are rendered in **two passes** before boids are drawn:

| Pass        | Alpha formula              | Width formula        | Purpose                          |
|-------------|----------------------------|----------------------|----------------------------------|
| Outer glow  | `t^0.6 × 0.28 × TRAIL_OPACITY` | `t × 18 + 1 px`  | Soft wide halo                   |
| Core line   | `t^0.5 × 0.85 × TRAIL_OPACITY` | `t × 3.5 + 0.4 px` | Bright narrow centre            |

where `t = i / (n−1)` is the normalised position along the trail (0 = oldest, 1 = newest).

Dying boids shed one trail position per frame (instead of appending) so the trail dissolves during the 2 s fade.

**Predators** use the same two-pass trail render with a white stroke colour.

---

## Birth Rings

When a child is born, a `birthRing` entry is added at the midpoint between the parents. Each ring renders three concentric expanding elements that fade over its duration:

| Element       | Max radius | Colour                        |
|---------------|-----------|-------------------------------|
| Outer bloom   | 150 px    | Boid colour, alpha ∝ (1−t)    |
| Middle ring   | 80 px     | White, stroked                |
| Central flash | 28 px     | White fill, only first 25% of lifetime |

---

## Death Splashes

When a predator eats a boid, a `deathSplash` is spawned at the boid's position. Each splash renders over ~0.85 s:

| Element        | Max radius | Colour                              |
|----------------|-----------|-------------------------------------|
| Outer ring     | 110 px    | Dark red, stroked, fades out         |
| Second ring    | 70 px     | Red-orange, stroked, fades out       |
| Inner flash    | 42 px     | Orange fill, only first 40% of lifetime |
| 20 particles   | 40–140 px travel | Dark red drops flying outward |

---

## Morphological Genetics

Both `beakGene` and `eyeGene` are **8-bit integers** (0–255). They are heritable, blended at birth, and mutated via the same bit-flip mechanism as colour.

### Inheritance

```
childGene = mutateGene( round( (parentA.gene + parentB.gene) / 2 ) )
```

Immigrants and pool boids receive the default gene value.

### Mutation (bit-flip)

With probability `MUTATION_RATE`, one random bit (0–7) is flipped:

```
childGene = parentGene XOR (1 << randomBit)   [masked to 8 bits]
```

Low bits produce tiny phenotypic shifts; flipping bit 7 (MSB) produces a dramatic jump of ±128 gene units. The distribution of effect sizes is thus power-law-like with no explicit parameterisation.

### Beak gene (males only expressed, carried by all)

| Gene | Multiplier formula                   | Range     |
|------|--------------------------------------|-----------|
| 0    | `0.2 + 0/255 × 2.8 = 0.20`           | very short |
| 73   | `0.2 + 73/255 × 2.8 ≈ 1.00`          | default    |
| 255  | `0.2 + 255/255 × 2.8 = 3.00`         | very long  |

Horn length = `RADIUS × 2.2 × beakMultiplier`. Females carry the gene silently.

### Eye gene (expressed by both sexes)

| Gene | Eye radius formula                        | Range      |
|------|-------------------------------------------|------------|
| 0    | `r × (0.1 + 0/255 × 0.55) = r × 0.10`   | tiny dot   |
| 64   | `r × (0.1 + 64/255 × 0.55) ≈ r × 0.24`  | default    |
| 255  | `r × (0.1 + 255/255 × 0.55) = r × 0.65` | large      |

Eye spread (lateral offset from body centre) also scales with eye radius to prevent overlap at large sizes.

---

## Colour Mutation

At every birth, after blending parents' colours, a bit-flip mutation is independently applied to the 24-bit packed RGB value:

```
if random() < MUTATION_RATE:
    packed = parseInt(hex, 16)
    bit    = random integer in [0, 23]
    packed = packed XOR (1 << bit)
    childColor = '#' + packed.toString(16).padStart(6, '0')
```

| Bit flipped | Channel affected | Max Δ channel | Typical perception |
|-------------|-----------------|---------------|--------------------|
| 0–7         | Blue            | ±1 – ±128     | Invisible – large  |
| 8–15        | Green           | ±1 – ±128     | Invisible – large  |
| 16–23       | Red             | ±1 – ±128     | Invisible – large  |

Within each channel, bits 0, 8, 16 (LSBs) produce ±1 shifts (silent); bits 7, 15, 23 (MSBs) produce ±128 shifts (major hue jumps). This naturally replicates the real distribution of mutation effect sizes without separate "small/large" logic.

---

## Colour Preference (Assortative Mating)

When `COLOR_PREF > 0`, each potential bonding hit is gated by a colour-similarity probability:

```
similarity = 1 − euclidean(RGB_A, RGB_B) / 441.67   # 441.67 = √(255²+255²+255²)
hitProb    = 1 − (1 − similarity) × COLOR_PREF
if random() > hitProb: skip this hit
```

| COLOR_PREF | Identical colours | Max-different colours |
|------------|-------------------|-----------------------|
| 0%         | 100% hit chance   | 100% hit chance (no effect) |
| 50%        | 100%              | 50%                   |
| 100%       | 100%              | 0% (never bond)       |

High colour preference causes the population to self-sort into colour lineages over generations. Combined with bit-flip mutation, this produces drifting colour families with occasional dramatic jumps.

---

## Predators

Predators are autonomous agents separate from the boid population. They do not flock, do not age, and do not breed.

### Appearance

Each predator is rendered as:

- **Body** — filled half-circle (flat/open side faces forward, curved side faces backward), white
- **Tail** — same tapered shape as a male boid horn, pointing backward, white
- **Eyes** — two circles at the diameter endpoints (jaw tips); colour transitions from yellow (full) to orange to red (very hungry); pulse with a `shadowBlur` glow when hungry
- **Glow** — soft white radial halo; brighter when hungry
- **Back dot** — small filled circle on the rear of the body, coloured in the current protected family's CSS colour (indicates which family the predator will not eat)

### Movement

Speed = `SPEED × 1.2` (20% faster than boids at the same speed setting).

**When hungry** (time since last meal > `FULL_DURATION`):
1. Scan all living boids within `PREDATOR_SENSE = 200 px`
2. Target the nearest boid that does **not** belong to the current protected colour family; if none found within range, wander
3. If `veryHungry` (> 2× `FULL_DURATION` without a meal), target any boid regardless of family
4. Steer toward target with a soft turn (`vx += (desired − vx) × 0.08`)

**When full** (recently eaten): wander at 60% speed with slow random heading changes.

Wall behaviour: soft steering force away from walls (begins at 1.5× `WALL_MARGIN`), with a hard velocity bounce at `PREDATOR_RADIUS` from the edge. Predators do **not** wrap around.

### Colour families and eating

Boid hues are partitioned into five named families:

| Family   | Hue range        | CSS colour |
|----------|------------------|------------|
| red      | 330°–30°         | `#ff4444`  |
| yellow   | 30°–90°          | `#ffd32a`  |
| green    | 90°–150°         | `#0be881`  |
| blue     | 150°–270°        | `#54a0ff`  |
| magenta  | 270°–330°        | `#d980fa`  |

A boid counts as belonging to a family only if its saturation ≥ 0.15 (non-grey boids only).

A single global `preferredFamily` is active at any moment — this is the family predators will **not** eat. All predators share the same protected family.

On collision (distance < `PREDATOR_RADIUS + RADIUS`):

| Predator state  | Boid not in preferred family | Boid in preferred family |
|-----------------|------------------------------|--------------------------|
| Hungry          | Eats                         | Ignores (unless `veryHungry`) |
| Very hungry (2×)| Eats                         | Eats                     |
| Full            | Ignores                      | Ignores                  |

Eating kills the boid immediately, spawns a **death splash**, and resets the predator's `lastEaten` timer.

### Family cycle

The protected family advances through the five families in order (`red → yellow → green → blue → magenta → red → …`). The cycle advances when ≥ 90% of living boids belong to the current protected family for `FAMILY_CONVERGE_HOLD = 5 s` continuously.

This creates a drive loop: predators cull non-protected boids → population converges to the protected colour → cycle advances → the formerly safe colour is now prey → population must evolve again. The UI shows the current family name and colour next to the `family` readout.

### Starvation

If there are **no living boids at all**, the predator's `noPreyTimer` increments. After `STARVATION_TIME = 45 s` of zero available prey, the predator dies and is removed. When any boid is alive, `noPreyTimer` resets to 0.

### Count management

The **predators** slider (0–5) is reactive: increasing it spawns new predators immediately; decreasing it removes the excess. On simulation restart, predators are respawned according to the current slider value.

---

## Export

The **⬇ export** button (green, next to the play button) generates a standalone `boids-aquarium.html` file:

1. All current slider positions are synced to their HTML attributes before capture
2. `document.documentElement.outerHTML` is captured (preserving all current values)
3. A `<style>` block is injected at the top of `<head>` to hide `#ui` immediately — no rendering flash
4. The canvas is repositioned as `position:fixed` filling the full window
5. A boot script is injected before `</body>` that:
   - Overrides `resize()` to use `window.innerWidth / innerHeight` (full screen)
   - Sets `AUTO_DISEASE = true` (see below)
   - Sets hardcoded `MAX_TRAIL` and `TRAIL_OPACITY` to current values
   - Calls `resize()` then `startSim()` immediately

The exported file has no controls — it is a self-running visual.

### Auto-disease

In the exported file `AUTO_DISEASE = true` activates an automatic outbreak timer. In the main loop:

```
_autoDisTimer -= dt
if _autoDisTimer <= 0:
    _autoDisTimer = random(30, 90)   # seconds
    infect one random healthy boid
```

This replaces the manual **infect** button with periodic spontaneous outbreaks.

---

## Performance

The frame loop is capped at **30 fps**:

```javascript
const FRAME_MIN_MS = 1000 / 30;
if (lastTime !== null && ts - lastTime < FRAME_MIN_MS) {
    requestAnimationFrame(loop); return;
}
```

`requestAnimationFrame` is still called every vsync — the early-return just skips simulation and rendering. `dt` is computed from actual elapsed time so simulation speed is framerate-independent.

**Primary performance levers** (in order of impact):

| Control          | Effect on cost                                         |
|------------------|--------------------------------------------------------|
| Boid count       | O(N²) collision detection + O(N × MAX_TRAIL) draw calls |
| Trail length     | Directly proportional to draw calls per frame          |
| Trail opacity 0% | Trail draw loop is still executed (no early-out)       |
| Predator count   | Minor; each predator adds one O(N) scan per frame      |

---

## User Controls

| Slider             | Range      | Default | Effect                                                          |
|--------------------|------------|---------|------------------------------------------------------------------|
| Target population  | 2–100      | 30      | Equilibrium point for all feedback arms                          |
| Bond after         | 1–20 hits  | 1       | Collisions required to form a bond                               |
| Base lifespan      | 5–80 s     | 80 s    | Lifespan at target population                                    |
| Pregnancy interval | 1–20 s     | 1 s     | Minimum time between births                                      |
| Family size        | 0–6        | 4.0     | Mean children per pair                                           |
| Divorce rate       | 0–50%      | 5%      | Per-second probability a pair splits                             |
| Speed              | 1–10       | 7       | Wander speed (mapped to 0.45–4.5 px/frame)                      |
| Singles            | 0–80%      | 20%     | Fraction of boids born unable to bond                            |
| Gay %              | 0–50%      | 10%     | Fraction of non-single boids that are gay                        |
| Infect. hits       | 1–10       | 1       | Exposure hits to catch disease                                   |
| Infect. dur.       | 5–60 s     | 10 s    | How long a boid stays infectious                                 |
| Fatality %         | 0–80%      | 10%     | Base case fatality rate (rises with diversity loss, up to 3×)    |
| Mutation %         | 0–50%      | 50%     | Per-birth probability of one bit-flip in colour, beak, or eye   |
| Color pref %       | 0–100%     | 0%      | Assortative mating strength by colour similarity                 |
| Predators          | 0–5        | 0       | Number of active predators (reactive — changes take effect live) |
| Full for (s)       | 1–30 s     | 10 s    | Time a predator won't eat after a meal                           |
| Trail opacity      | 0–100%     | 40%     | Scales both glow and core trail alpha                            |
| Trail length       | 5–150      | 30      | Number of past positions retained in the trail buffer            |

Buttons: **immigration toggle**, **infect** (introduces one disease case), **▶ / ▐▐ play/pause**, **⬇ export**.

---

## Visual Key

| Visual element                    | Meaning                                                              |
|-----------------------------------|----------------------------------------------------------------------|
| Circle body (female)              | Female boid; colour encodes genetic lineage                          |
| Circle body + forward horn (male) | Male boid; horn length encodes beakGene                              |
| Two dark eyes (both sexes)        | Eye size encodes eyeGene; same gene inherited by both sexes          |
| Small body (0.7× size)            | Single boid — will never bond                                        |
| Gold glow around body             | Bonded boid                                                          |
| Dimmed opacity (55%)              | Outside fertile window (juvenile or post-reproductive)               |
| White spot (centre)               | Diseased boid — currently infectious                                 |
| Coloured trail behind boid        | Motion history; length = MAX_TRAIL frames; colour matches boid       |
| White trail                       | Predator motion history                                              |
| Expanding rings at birth          | Birth ring event at spawn point                                      |
| Red expanding rings + particles   | Death splash — boid eaten by predator                                |
| Gold arc (partial)                | Hit progress toward bonding threshold                                |
| Gradient line (amber)             | Bond between fertile straight M+F partners                           |
| Gradient line (colour)            | Bond — partners not forming a fertile pair                           |
| Bright red bond line (2 s)        | Flash at moment of child birth                                       |
| White half-circle body + tail     | Predator (full = dark glow; hungry = bright glow + pulsing eyes)     |
| Yellow→red pulsing eyes           | Predator hunger level: yellow = full, orange = hungry, red = very hungry |
| Coloured dot on predator back     | Protected colour family — predators will not eat boids of this colour (unless very hungry) |
| Density bar                       | Real-time combined pressure (Arm 1)                                  |
| Lifespan bar                      | Real-time effective lifespan (Arm 2)                                 |
| pairs X:Y readout                 | Total bonded pairs : fertile bonded M+F pairs                        |
| sick / R readout                  | Current infected count and rolling R-factor                          |
| dis.deaths readout                | Disease fatalities in the last 30 s of simulation time               |
| eff.br. readout                   | Effective breeders (fertile bonded M+F × 2); red when < 50          |
| 50-rule readout                   | critical (< 50) / at risk (< 500) / ok (≥ 500)                      |
| inbreeding warning                | Appears when diversity < 0.3; shows hazard multiplier and modal age reduction |
| fatality → N% label               | Appears when effective fatality diverges from base due to low diversity |
