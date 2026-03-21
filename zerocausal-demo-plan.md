# ZeroCausal Demo Plan

## Overview

An interactive browser demo showcasing the Zero-Causal methodology — O(1) deterministic access to any point in a procedural causal chain without sequential computation. Two parallel demonstrations: **Civilisation History** and **Genetic Lineage**.

---

## Core Principle

> Depth is a coordinate. The past is not a log.

`chain_origin_seed(x, y, world_seed, type_salt)` → deterministic root  
`event_at_depth(origin_seed, depth)` → any event, O(1), no replay

---

## Demo A: Civilisation History

### What it Shows
A named civilisation (derived from its seed coordinates) with an infinite political history. The user can jump directly to any historical era — 0, 50, 500, 1000 — without computing preceding eras.

### Event Vocabulary (9 types)
`founding | expansion | civil_war | golden_age | plague | invasion | reform | collapse | revival`

### Transition Biases
- `founding` → expansion likely
- `civil_war` → invasion / collapse likely
- `golden_age` → expansion / reform likely
- `collapse` → revival likely

### Data per era
- Event type (from vocabulary)
- Magnitude (0.0 – 1.0)
- Duration (1 – 20 years)
- Lasting effect (boolean, magnitude > 0.75)
- Technology category (agriculture / metallurgy / navigation / medicine / architecture / warfare / philosophy / engineering)

### UI
- World seed input + city coordinate inputs (X, Y)
- City name generator (derived from seed, e.g. "Veranthos")
- Era range slider: jump directly to any era 0–200
- "Jump to Era N" input field for arbitrary depth
- Timeline strip: render eras N–N+12 as coloured event cards
- Live O(1) badge showing no iteration occurred
- Alternate history fork button: branch at current era, show branch A vs branch B side-by-side

---

## Demo B: Genetic Lineage

### What it Shows
A species lineage where each generation carries a mutation. Jump to generation 0, 100, 10000 instantly.

### Mutation Types (8 types)
`beneficial_metabolism | neutral_coloration | harmful_immunity | beneficial_size | neutral_behaviour | beneficial_cognition | harmful_reproduction | neutral_lifespan`

### Data per generation
- Mutation type
- Fitness delta (–1.0 to +1.0)
- Is dominant (boolean)
- Causes speciation split (abs(fitness_delta) > 0.8)

### UI
- Species seed + origin coordinates
- Species name generator (derived from seed)
- Generation jump input: enter any generation number
- Cumulative fitness chart: render generations N to N+20 as a sparkline
- Speciation split indicator: highlight generations where split occurred
- Compare two lineages side by side (different X,Y coordinates, same world seed)

---

## Hash Engine (JavaScript port of ZeroCausal Python)

```javascript
// xxhash32 inline — no external deps
function xxhash32(data, seed = 0) { /* ... */ }

function positionHash(x, y, z, salt = 0) {
  // Pack as 3× int32, run xxhash32
}

function chainOriginSeed(ox, oy, worldSeed, typeSalt = 0) {
  return positionHash(ox, oy, typeSalt, worldSeed);
}

function eventAtDepth(originSeed, depth, salt = 0) {
  return positionHash(originSeed, depth, 0, salt);
}

function eventFloat(originSeed, depth, channel = 0) {
  return (eventAtDepth(originSeed, depth, channel) >>> 0) / 0x100000000;
}

function eventInt(originSeed, depth, modulus, channel = 0) {
  return Math.floor(eventFloat(originSeed, depth, channel) * modulus);
}

function biasedEventType(originSeed, depth, numTypes, transitionBias) {
  const prevType = depth === 0 ? 0 : eventInt(originSeed, depth - 1, numTypes);
  const weights = transitionBias[prevType] || Array(numTypes).fill(1);
  const total = weights.reduce((a, b) => a + b, 0);
  const raw = eventFloat(originSeed, depth);
  let cumulative = 0;
  for (let i = 0; i < weights.length; i++) {
    cumulative += weights[i] / total;
    if (raw < cumulative) return i;
  }
  return numTypes - 1;
}
```

---

## UI Layout

```
┌─────────────────────────────────────────────────────────────┐
│  ZERO-CAUSAL  ·  Chain-is-Seed Deterministic History Engine │
├──────────────────────────┬──────────────────────────────────┤
│  CIVILISATION HISTORY    │  GENETIC LINEAGE                 │
│  World Seed: [____]      │  World Seed: [____]              │
│  City X: [_] Y: [_]      │  Species X: [_] Y: [_]          │
│  City: "Veranthos"       │  Species: "Lyranth"              │
│                          │                                  │
│  Era: [slider 0-200]     │  Generation: [______] [JUMP]     │
│  [Jump to Era: ___] [GO] │                                  │
│                          │  Gen N mutation card             │
│  Timeline strip          │  Fitness sparkline               │
│  [Era cards N..N+12]     │  [Side-by-side lineage compare]  │
│                          │                                  │
│  [Fork timeline ⑂]       │  Speciation alerts               │
└──────────────────────────┴──────────────────────────────────┘
```

---

## Aesthetic Direction

**Theme**: Dark terminal / ancient stone hybrid — monospace data panels with aged parchment texture overlays.  
**Palette**: Deep obsidian `#0a0a0f`, amber `#c8870a`, pale gold `#e8d5a0`, blood red `#8b1a1a` for collapse events, emerald `#1a6b3a` for golden age.  
**Typography**: Display — `Cinzel` (Roman/monumental). Data — `JetBrains Mono`. Body — `Crimson Pro`.  
**Motion**: Era cards slide in from right on jump; speciation splits animate with a branching fork effect; fitness sparkline draws itself.  
**Key detail**: Each event card has a faint icon glyph (⚔ invasion, 🌾 founding, ☀ golden age, ☠ plague, ⚖ reform, 💀 collapse, 🌱 revival) rendered in SVG inline.

---

## Verification Panel

A small collapsible panel demonstrating O(1) proof:  
- Shows time taken to compute era 0 vs era 10,000 (both < 1ms)  
- Shows that querying era 9999 before era 0 gives same result as sequential order  
- Renders as a micro benchmark table

---

## Files

- `demo.html` — single-file self-contained demo, no build step, no external deps except Google Fonts
