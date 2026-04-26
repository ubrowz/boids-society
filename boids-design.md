# Boids Aquarium — Design & Implementation

## Overview

A browser-based population simulation built on the classic boids algorithm. Coloured circles (boids) wander autonomously inside a bounded aquarium. They can bond, breed, age and die. A multi-arm feedback loop attempts to maintain a stable population around a user-defined target.

All logic runs in a single self-contained HTML file with no external dependencies.

-----

## Boid Lifecycle

Each boid progresses through the following states:

```
spawn → fade-in → juvenile → fertile adult → post-fertile → dying → removed
```

|Property           |Description                                               |
|-------------------|----------------------------------------------------------|
|`age`              |Seconds lived since birth                                 |
|`effectiveLifespan`|Actual lifespan — varies with population pressure         |
|`birthTimer`       |Fade-in duration (1.5 s)                                  |
|`dying`            |True when age ≥ effectiveLifespan; boid fades out over 2 s|

-----

## Movement

Each boid moves at a constant speed, direction updated every frame by four competing steering forces applied in sequence:

1. **Wall avoidance** — soft repulsion begins 40 px from any wall; hard bounce at the wall itself
1. **Wander** — a random-walk steering target projected ahead of the boid on a circle, jittered each frame
1. **Nearness** — attraction toward the local flock centre (see Feedback Arm 1)
1. **Bond attraction** — bonded partners stay within 45 px of each other

Speed is user-controlled via a slider (1–10, mapped to 0.45–4.5 px/frame).

-----

## Singles

Each boid is assigned a "single" status at birth, drawn probabilistically from the **singles** slider (0–80%, default 20%). Singles never bond. They are drawn at 0.7× the normal radius to visually distinguish them.

-----

## Bonding

Two unbonded, non-single boids bond after accumulating `BOND_THRESHOLD` collisions (overlapping circles). A per-pair cooldown prevents one slow overlap from counting as many hits.

**Restrictions:**

- A bonded boid cannot bond again until its partner dies or the pair divorces
- A child cannot bond with either parent
- Siblings (same two parents) cannot bond with each other
- Singles never bond

**Bond threshold interaction with nearness:**
Higher `BOND_THRESHOLD` requires more collisions, so boids need sustained proximity. The nearness strength is scaled upward proportionally (`1 + (threshold - 1) × 0.24`) to compensate. The hit cooldown also shrinks with higher thresholds (`COOLDOWN_FRAMES / √threshold`) so hits accumulate faster.

-----

## Divorce

Each frame, each bonded pair has a small chance to split equal to `DIVORCE_RATE × dt`. When a divorce occurs:

- Both partners become unbonded and their hit/children records for that pair are cleared
- Each partner's breed timer is reset with a random offset
- `pairFamilyTarget` is reset to 0 for both

The **divorce rate** slider (0–50%, default 5%) sets the per-second probability.

-----

## Sex and Fertility

Each boid is assigned a sex (`M` or `F`) at birth. Sex is not purely random — it is biased toward whichever sex is currently underrepresented in the living population, keeping the ratio close to 50/50.

Fertility windows are fixed fractions of each boid's effective lifespan:

|Parameter     |Value                    |
|--------------|-------------------------|
|♂ fertile from|20% of lifespan          |
|♂ fertile for |70% of lifespan          |
|♀ fertile from|20% of lifespan          |
|♀ fertile for |50% of lifespan          |

A boid is fertile only when `age` falls within `[lifespan × from, lifespan × (from + for)]`. Fertility windows scale automatically when the lifespan slider is changed.

**Only bonded M+F pairs where both partners are currently fertile can produce children.**

-----

## Breeding

When a bonded fertile M+F pair is active, a per-pair family target determines reproductive capacity. At bond formation the pair's family target is drawn from `Normal(FAMILY_SIZE, 0.8)`, clamped to a minimum of 0 and rounded to the nearest integer. This is stored on both partners as `pairFamilyTarget`.

The effective maximum children is then:

```
t              = (globalPressure + 1) / 2          # 0..1
desiredMax     = ceil(pairFamilyTarget × (1 − max(t − 0.5, 0) × 2))
effectiveMax   = min(desiredMax, bioCeiling)
bioCeiling     = floor(effectiveLifespan × F_FERTILE_FOR / PREG_INTERVAL)
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

-----

## The Feedback Loop

Stability is maintained through three independent feedback arms, all driven by **pressure signals** derived from the current population state. The three pressure components are kept separate to prevent them from interfering with each other.

### Pressure Signals

**Local pressure** (`localPressure`, 0–1)
Measures neighbourhood crowding: the fraction of other living boids within the sensing radius (130 px), normalised against the total living population minus one.

```
localPressure = min(neighbours / max(totalBoids − 1, 1), 1)
```

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

-----

### Arm 1 — Nearness (birth rate, indirect)

**Signals used:** `localPressure` and `popPressure`
**Effect:** Controls how strongly boids steer toward each other.

```
combinedPressure = min(localPressure + max(popPressure, 0), 1)
strength = (MIN_NEARNESS + (BASE_NEARNESS − MIN_NEARNESS) × (1 − combinedPressure))
           × thresholdScale
```

The `popPressure` term is only added when positive (overshoot), so when the raw headcount exceeds target it boosts combined pressure and reduces/reverses attraction — dispersing boids even if local density happens to be low.

|combinedPressure|Nearness strength|Effect                                          |
|----------------|-----------------|------------------------------------------------|
|0 (sparse)      |High (up to 4.0×)|Boids cluster → more collisions → faster bonding|
|1 (dense)       |Low (0.05)       |Boids spread out → fewer collisions             |

When `strength` goes negative (repulsion zone), boids steer away from the flock centre.

This arm controls collision frequency, which is the prerequisite for bonding and therefore the entire breeding pipeline. It does **not** directly control lifespan or birth rate — it acts on the *opportunity* to bond.

-----

### Arm 2 — Lifespan (death rate)

**Signal used:** `globalPressure`
**Effect:** Scales effective lifespan around the base value set by the slider.

```
t              = (globalPressure + 1) / 2          # 0..1
scale          = LIFESPAN_SCALE_MAX + (LIFESPAN_SCALE_MIN − LIFESPAN_SCALE_MAX) × t
effectiveLifespan = min(BASE_LIFESPAN × scale, MAX_LIFESPAN)
```

Constants: `LIFESPAN_SCALE_MAX = 1.25`, `LIFESPAN_SCALE_MIN = 0.4`, `MAX_LIFESPAN = 100 s`

|globalPressure  |Lifespan scale|Effective lifespan (base 80 s)|
|----------------|--------------|------------------------------|
|−1 (crisis)     |1.25×         |100 s (hard cap)              |
|0 (at target)   |1.0×          |80 s                          |
|+1 (over target)|0.4×          |32 s                          |

This arm acts on the **death rate** directly and with no lag — every living boid's lifespan extends or shrinks immediately when global pressure changes.

-----

### Arm 3 — Breeding rate (birth rate, direct)

**Signal used:** `globalPressure`
**Effect:** Scales both the pregnancy interval and the maximum children per pair.

|globalPressure  |Interval multiplier|Effective max children (family size 2)|
|----------------|-------------------|--------------------------------------|
|−1 (crisis)     |~0.33× (3× faster) |2 (no bonus above family target)      |
|0 (at target)   |1.0×               |2                                     |
|+1 (over target)|~3.0× (3× slower)  |0 (breeding stops)                    |

-----

### Why three separate signals?

Early versions used a single blended pressure signal for all arms. This caused a critical failure: when population was low and nearness caused boids to cluster, local density increased, which raised pressure, which suppressed recovery — the system sabotaged itself. Separating the signals prevents this:

- Arm 1 responds to combined local density + raw headcount overshoot (spacing behaviour)
- Arms 2 and 3 respond only to the global population-vs-target error
- `popPressure` only suppresses clustering on the upside; it does not accelerate clustering when under target

-----

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

-----

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

-----

## Immigration

When immigration is enabled (toggle in the UI), an immigrant boid may be spawned based on combined deficit pressure:

```
popDeficit     = max(1 − boids.length / TARGET_POP, 0)
fertileDeficit = max(−smoothGlobalPressure, 0)
pressure       = max(popDeficit, fertileDeficit)
```

Immigration also triggers on genetic deadlock: when no valid unrelated M+F fertile non-single pairs exist. The spawn interval shortens as pressure increases (minimum ~5 s at full crisis).

Immigration is suppressed entirely when population exceeds 120% of target.

Immigrants:

- Arrive with a fresh random color and genome
- Have no family key and no parent IDs (can bond with anyone)
- Are spawned at mid-fertile-age so they can breed immediately
- Are assigned the sex most needed for balance
- Are never singles

### Immigrant lineage tracking

Each boid carries an `immigrantGeneration` field:

- `−1` — no immigrant ancestry
- `0` — the immigrant itself
- `1` — child of immigrant
- `2` — grandchild, etc.

A black dot is drawn inside the boid, fading each generation:

```
dotOpacity = max(0, 1 − immigrantGeneration / 4)
```

The dot disappears completely by the 4th generation, representing full assimilation of the immigrant lineage into the founding population.

-----

## User Controls

|Slider            |Range    |Default|Effect                                                   |
|------------------|---------|-------|---------------------------------------------------------|
|Target population |2–100    |20     |Sets the equilibrium point for all feedback arms         |
|Bond after        |1–20 hits|3      |Collisions required to form a bond                       |
|Base lifespan     |5–80 s   |80 s   |Lifespan at target population (Arm 2 scales around this) |
|Pregnancy interval|1–20 s   |6 s    |Minimum time between births (Arm 3 scales this)          |
|Family size       |0–6      |2      |Mean children per pair (drawn per-pair from Normal dist.) |
|Divorce rate      |0–50%    |5%     |Per-second probability a bonded pair splits              |
|Speed             |1–10     |4      |Wander speed — affects collision frequency               |
|Singles           |0–80%    |20%    |Fraction of boids born unable to bond                    |

An **immigration toggle** button enables or disables immigrant spawning at runtime.

-----

## Visual Key

|Visual element           |Meaning                                                             |
|-------------------------|--------------------------------------------------------------------|
|Circle body              |The boid; color encodes genetic lineage                             |
|Small circle (0.7× size) |Single boid — will never bond                                       |
|♂ / ♀ mark inside        |Sex (drawn as geometric lines in dark blue / dark red)              |
|Yellow-to-red arc outside|Age ring — fills over effective lifespan, turns red near death      |
|Gold ring                |Bonded                                                              |
|Gold arc (partial)       |Hit progress toward bonding threshold                               |
|Cyan ring                |Local density pressure                                              |
|Dimmed opacity (55%)     |Outside fertile window (juvenile or post-reproductive)              |
|Black left dot           |Immigrant or immigrant descendant; fades over 4 generations         |
|Gradient line (amber)    |Bond between fertile M+F partners                                   |
|Gradient line (color)    |Bond between partners where pair is not currently fertile           |
|Density bar              |Real-time combined pressure (Arm 1)                                 |
|Lifespan bar             |Real-time effective lifespan (Arm 2)                                |
