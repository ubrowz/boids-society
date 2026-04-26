# Pics Aquarium — Design & Implementation

## Overview

A browser-based population simulation built on the classic boids algorithm. Coloured boids wander autonomously inside a bounded aquarium. They can bond, breed, age and die. A multi-arm feedback loop attempts to maintain a stable population around a user-defined target.

Males are rendered as tadpoles (head + trailing tail); females are rendered as circles.

All logic runs in a single self-contained HTML file with no external dependencies.

---

## Boid Lifecycle

Each boid progresses through the following states:

```
spawn → fade-in → juvenile → fertile adult → post-fertile → dying → removed
```

| Property            | Description                                                |
|---------------------|------------------------------------------------------------|
| `age`               | Seconds lived since birth                                                          |
| `maxLifespan`       | Biological ceiling — fixed at birth, drawn from Normal(BASE_LIFESPAN, 0.2 × BASE_LIFESPAN), floored at 30% of base. Never changes. |
| `effectiveLifespan` | Display-only: current implied mean lifespan given population pressure (= BASE_LIFESPAN × pressureScale). Not used as a death trigger. |
| `birthTimer`        | Fade-in duration (1.5 s)                                                           |
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

## Sexual orientation

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

The **family size** slider (0–6, default 2) sets `FAMILY_SIZE`, the mean of the per-pair target distribution.

The **pregnancy interval** slider (1–20 s, default 6 s) sets the minimum time between births. The actual interval is modulated by pressure:

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

| globalPressure   | Interval multiplier | Effective max children (family size 2) |
|------------------|---------------------|----------------------------------------|
| −1 (crisis)      | ~0.33× (3× faster)  | 2 (no bonus above family target)       |
| 0 (at target)    | 1.0×                | 2                                      |
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

- ~5 fertile pairs, interval 6 s → ~0.83 births/s
- 20 boids, 80 s lifespan → 0.25 deaths/s

The system has a birth surplus at target, which gives the feedback loop room to self-correct downward through the interval and cap mechanisms when population exceeds target.

---

## Genetic System

### Color as genome

Each boid carries a `genome` — normalized RGB values `[r, g, b]` derived from its color. Children receive a blended genome from both parents plus a small random mutation (±0.025 per channel).

### Genetic distance preference

When two unbonded boids collide, the cooldown before the next registered hit is modulated by their genetic distance:

```
geneticDistance = euclidean(genome_A, genome_B) / √3    # 0..1
cooldownScale   = 2.0 − 1.5 × geneticDistance
```

Genetically similar boids (distance ≈ 0) have 2× longer cooldowns — they bond slowly. Genetically distant boids (distance ≈ 1) have 0.5× cooldowns — they bond up to 4× faster. This naturally preserves genetic diversity by favouring outbreeding.

### Genetic drift and color convergence

Without intervention, color (and genome) converges to a single value over many generations through **allele fixation by genetic drift** — whichever lineage happens to breed successfully first gradually displaces all others. This is visible as all boids converging to the same color.

### Inbreeding depression

As diversity falls, two independent fitness penalties activate. Both use the same diversity thresholds:

```
t = 1 − clamp((diversity − DIVERSITY_LOW) / (DIVERSITY_HIGH − DIVERSITY_LOW), 0, 1)

DIVERSITY_HIGH = 0.3   # above this: no penalty
DIVERSITY_LOW  = 0.05  # below this: full penalty
```

**Mortality penalty** (`inbreedingHazardMultiplier = 1 + t × 0.5`, up to 1.5×)
Models general fitness reduction — homozygous expression of deleterious recessive alleles shortens lives. Applied as a multiplier to the Gompertz hazard every frame. Shifting the hazard by multiplier M shifts the modal age down by `ln(M) / beta` seconds.

**Disease susceptibility penalty** (`diversityFatalityMultiplier = 1 + t × 2.0`, up to 3×)
Models HLA (immune receptor) diversity collapse — a genetically uniform population cannot mount varied immune responses. Applied to `DISEASE_FATALITY` before computing the disease hazard rate. With 20% base fatality, full convergence pushes effective fatality to 60%. The UI shows `→ N%` next to the fatality slider when the effective rate diverges from the base.

The two penalties compound: a diseased boid in a low-diversity population faces both elevated disease kill probability and elevated background mortality.

### 50/500 rule

**Effective breeders** = fertile bonded M+F pairs × 2. This is computed each frame and displayed in the `eff.br.` readout.

The **50/500 rule** from conservation biology states:
- **< 50 effective breeders**: short-term inbreeding depression becomes severe — immediate viability risk
- **< 500 effective breeders**: long-term genetic drift erodes adaptive potential

A third hazard multiplier captures the trajectory risk of small populations, independent of currently observed diversity (a population may be genetically diverse now but will converge rapidly if it stays small):

```
smallPopHazardMultiplier = 1 + (1 − min(effectiveBreeders, 50) / 50) × 0.2
```

This reaches 1.2× at zero effective breeders. The `50-rule` readout shows **critical** (< 50), **at risk** (< 500), or **ok** (≥ 500).

At typical simulation scales (TARGET_POP = 20), effective breeders are almost always below 50. This is intentional — it reflects that the simulation inherently runs at population sizes the 50/500 rule classifies as critical, illustrating why immigration and genetic diversity matter. Raise TARGET_POP toward 100+ to push into the "at risk" range.

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

- Arrive with a fresh random color and genome
- Have no family key and no parent IDs (can bond with anyone)
- Are spawned at mid-fertile-age so they can breed immediately
- Are assigned the sex most needed for balance
- Are never singles, and are always straight

---

## Disease

A contagious disease can be introduced at any time by pressing the **infect** button, which randomly selects one healthy boid and infects it.

### Transmission

Healthy boids accumulate exposure hits whenever they physically overlap with an infected boid. A per-pair cooldown (`DISEASE_COOLDOWN = 1.5 s`) prevents rapid accumulation from a single sustained contact. Exposure hits are cumulative across all infected boids the healthy boid encounters. Once hits reach `DISEASE_THRESHOLD`, the healthy boid becomes infected.

```
exposureHits += 1 per qualifying collision with any infected boid
if exposureHits >= DISEASE_THRESHOLD → infected
```

The infected boid that delivers the threshold hit is credited with one entry in its `infectCount`.

### Fatality

Each frame while a boid is diseased it faces an additional death roll independent of the Gompertz hazard:

```
effFatality = min(0.9999, DISEASE_FATALITY × diversityFatalityMultiplier)
h_disease   = −ln(1 − effFatality) / DISEASE_DURATION
P(die from disease this frame) = h_disease × dt
```

This derived from requiring that survival probability over the full disease duration equals `1 − effFatality`. Deaths are therefore spread naturally across the illness period — some boids die early, some late, survivors recover. Setting fatality to 0% disables this mechanism entirely.

When a boid dies from disease its `dying` flag is set immediately (2 s visual fade); `killBoid` records its `infectCount` to the R-factor history since `boid.diseased` remains true during the fade. Disease fatalities are also tracked separately in the `dis.deaths` rolling-30 s readout.

**Diversity link:** `effFatality` is the base fatality scaled by `diversityFatalityMultiplier` (up to 3×). See *Inbreeding depression* above. The UI shows `→ N%` next to the fatality slider when the effective rate diverges from the slider value.

### Recovery and immunity

Infected boids that survive the full `DISEASE_DURATION` recover and become permanently immune. Recovery only fires when `boid.dying` is false — boids already dying from the fatality roll do not also trigger recovery.

### R-factor

The R-factor is the rolling mean of `infectCount` over the last 30 recovered or deceased infected boids. It represents the average number of secondary infections caused by one infected individual and is displayed live in the status readout. Before any boid has recovered the display shows `—`.

### Parameters

| Slider        | Range    | Default | Effect                                                              |
|---------------|----------|---------|---------------------------------------------------------------------|
| Infect. hits  | 1–10     | 3       | Exposure hits required to catch the disease                         |
| Infect. dur.  | 5–60 s   | 20 s    | How long a boid stays infectious                                    |
| Fatality %    | 0–80%    | 20%     | Base case fatality rate — scaled up by diversity loss (up to 3×)   |

---

## User Controls

| Slider             | Range     | Default | Effect                                                    |
|--------------------|-----------|---------|-----------------------------------------------------------|
| Target population  | 2–100     | 20      | Sets the equilibrium point for all feedback arms          |
| Bond after         | 1–20 hits | 3       | Collisions required to form a bond                        |
| Base lifespan      | 5–80 s    | 80 s    | Lifespan at target population (Arm 2 scales around this)  |
| Pregnancy interval | 1–20 s    | 6 s     | Minimum time between births (Arm 3 scales this)           |
| Family size        | 0–6       | 2       | Mean children per pair (drawn per-pair from Normal dist.) |
| Divorce rate       | 0–50%     | 5%      | Per-second probability a bonded pair splits               |
| Speed              | 1–10      | 4       | Wander speed — affects collision frequency                |
| Singles            | 0–80%     | 20%     | Fraction of boids born unable to bond                     |
| Gay %              | 0–50%     | 10%     | Fraction of non-single boids that are gay                 |
| Infect. hits       | 1–10      | 3       | Exposure hits required to catch the disease                              |
| Infect. dur.       | 5–60 s    | 20 s    | How long a boid stays infectious                                         |
| Fatality %         | 0–80%     | 20%     | Base case fatality rate — effective rate rises with diversity loss        |

Buttons: **immigration toggle**, **infect** (introduces disease to one random healthy boid).

---

## Visual Key

| Visual element            | Meaning                                                       |
|---------------------------|---------------------------------------------------------------|
| Circle body (female)      | Female boid; color encodes genetic lineage                    |
| Tadpole body (male)       | Male boid; circle head with tapered tail pointing backward    |
| Small body (0.7× size)    | Single boid — will never bond                                 |
| Gold glow around body     | Bonded boid                                                   |
| Dimmed opacity (55%)      | Outside fertile window (juvenile or post-reproductive)        |
| Black spot (center)       | Diseased boid — currently infectious                          |
| Gold arc (partial)        | Hit progress toward bonding threshold                         |
| Gradient line (amber)     | Bond between fertile straight M+F partners                    |
| Gradient line (color)     | Bond between partners not currently forming a fertile pair    |
| Bright red line (2 s)     | Bond line flash at the moment a child is spawned              |
| Density bar               | Real-time combined pressure (Arm 1)                           |
| Lifespan bar              | Real-time effective lifespan (Arm 2)                          |
| pairs X:Y readout         | Total bonded pairs : fertile bonded M+F pairs                 |
| sick / R readout          | Current infected count and rolling R-factor                   |
| dis.deaths readout        | Disease fatalities in the last 30 s of simulation time        |
| eff.br. readout           | Effective breeders (fertile bonded M+F pairs × 2); red < 50   |
| 50-rule readout           | critical (< 50) / at risk (< 500) / ok (≥ 500)               |
| inbreeding warning        | Appears below lifespan bar when diversity < 0.3; shows hazard multiplier and modal age reduction |
| fatality → N% label       | Appears next to fatality slider when effective fatality diverges from base due to low diversity |
