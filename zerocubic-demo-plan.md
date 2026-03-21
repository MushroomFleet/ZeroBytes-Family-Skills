# ZeroCubic Demo Plan: NPC Group Dynamics

## Concept

Demonstrate the Zero-Cubic methodology applied to **NPC group dynamics** — specifically who leads, who is scapegoated, and the emergent shifting-alliance zone that exists between those extremes. The key insight: a trio's social structure **cannot be reduced** to summing three pairwise relationships. The triangle has a soul the edges alone cannot explain.

---

## Zero-Cubic Core: Group Dynamic Engine

### The Three NPC Properties (ZeroBytes Layer — O(1))

Each NPC has intrinsic properties derived from their ID alone:

```js
function npcTraits(id, worldSeed) {
  const h = xxh32(id, worldSeed);
  return {
    dominance:   float(h)          // 0–1: assertiveness
    resilience:  float(h >>> 8)    // 0–1: resistance to scapegoating
    charisma:    float(h >>> 16)   // 0–1: pull on others in group
    volatility:  float(h >>> 24)   // 0–1: how much they destabilize triads
  };
}
```

### The Pairwise Bond (Zero-Quadratic Layer — O(N²))

Each pair has a bond independent of who the third person is:

```js
function pairBond(idA, idB, worldSeed) {
  const sorted = [idA, idB].sort((a, b) => a - b);
  const h = xxh32_pair(sorted[0], sorted[1], worldSeed + 100);
  return {
    affinity:  float(h),        // base like/dislike
    rivalry:   float(h >>> 16)  // competition intensity
  };
}
```

### The Triadic Emergence (Zero-Cubic Layer — O(N³))

The group dynamic **only exists when all three are present**:

```js
function tripleHash(idA, idB, idC, salt) {
  // Symmetric: sort all three IDs
  const [a, b, c] = [idA, idB, idC].sort((x, y) => x - y);
  return xxh32_triple(a, b, c, salt);
}

function groupDynamic(idA, idB, idC, worldSeed) {
  const stability = float(tripleHash(idA, idB, idC, worldSeed));

  // Leader/scapegoat index uses ASYMMETRIC hash — order encodes directionality
  const dominantIdx = Math.floor(
    float(asymTripleHash(idA, idB, idC, worldSeed + 1)) * 3
  );

  if (stability > 0.75) {
    return { type: "cohesive",  leaderIndex:    dominantIdx, stability };
  }
  if (stability < 0.25) {
    return { type: "volatile",  scapegoatIndex: dominantIdx, stability };
  }
  return   { type: "shifting", pivotIndex:     dominantIdx, stability };
}
```

**Why this can't be pair-sums:**
- A highly charismatic NPC paired with two rivals might *stabilize* all three by redirecting their rivalry outward — this only appears at the triadic level.
- Scapegoating requires a majority coalition; two people can't scapegoat themselves.

---

## Demo Structure

### Scene Layout

- **Canvas**: Top-down social graph view, 600×600px
- **NPCs**: 6–8 labeled circles scattered across the canvas
- **Active Trio**: Highlighted triangle connecting 3 selected NPCs
- **Dynamic Readout**: Live sidebar showing the Zero-Cubic result

### Interaction Model

1. **Click NPCs to select/deselect** — up to 3 at a time
2. When exactly 3 are selected, the triadic analysis fires automatically
3. The triangle animates: green pulse = cohesive, red throb = volatile, amber flicker = shifting
4. **"Randomize World Seed"** button regenerates all NPC traits and bonds
5. **"Show All Triads"** heatmap overlay: color-code every possible trio by stability

### Visual Encoding

| Dynamic Type | Triangle Color | Animation | Leader/Scapegoat Indicator |
|---|---|---|---|
| Cohesive (>0.75) | Emerald glow | Slow steady pulse | Crown icon on leader |
| Volatile (<0.25) | Crimson throb | Rapid irregular flicker | Target reticle on scapegoat |
| Shifting (0.25–0.75) | Amber shimmer | Slow drift between nodes | Rotating arrow on pivot |

### Information Panel

Shows for the active trio:
- **Stability score** (0.00–1.00) with colored bar
- **Dynamic type** label
- **Role assignment**: who leads / who is blamed / who pivots
- **Pairwise bonds** for comparison: "A↔B: 0.82 | B↔C: 0.31 | A↔C: 0.67" to illustrate that pair sums ≠ triadic result
- **Proof of emergence**: display what pair-sum would predict vs what Zero-Cubic actually gives

### Heatmap Mode

Toggle a "Show All Triads" overlay that:
- Enumerates C(N,3) triangles (max 56 for N=8)
- Colors each triangle edge by stability tier
- Identifies the single most stable and most volatile trio
- Shows the distribution histogram: % cohesive / shifting / volatile

---

## NPC Roster (N=8, fixed names, IDs 0–7)

| ID | Name | Archetype Flavor |
|---|---|---|
| 0 | Mira | Quiet schemer |
| 1 | Dax | Loud braggart |
| 2 | Senna | Peacemaker |
| 3 | Kolt | Contrarian |
| 4 | Yael | Charming opportunist |
| 5 | Fenn | Anxious follower |
| 6 | Roja | Old grudge-holder |
| 7 | Tabs | Wild card |

NPC positions are fixed in the canvas layout (not physics-simulated) for clarity. Trait values are derived deterministically from ID + worldSeed.

---

## Technical Notes

- **No libraries required** — pure vanilla JS, CSS, Canvas API
- **xxHash** implemented as a fast 32-bit Murmur-style mix (portable, no WASM needed)
- **Seed range**: 0–9999, controlled by slider or random button
- **Determinism guarantee**: same seed → same result across all browsers
- **N cap**: Hard-coded N=8; C(8,3)=56 triples, well within interactive budget
- **Anti-pattern avoidance**: No `Math.random()` used anywhere in the dynamic calculations; no pair-sum approximations

---

## Aesthetic Direction (for Frontend Build)

**Theme**: Dark intelligence / social graph noir  
**Palette**: Deep navy `#0a0e1a`, node glow in faction color, amber/crimson/emerald for dynamics  
**Typography**: Monospace data readout + condensed sans for labels  
**Mood**: Like watching surveillance footage of a social experiment — clinical but tense  
**Animation**: Subtle, purposeful — the triangle breathes with the dynamic type  
