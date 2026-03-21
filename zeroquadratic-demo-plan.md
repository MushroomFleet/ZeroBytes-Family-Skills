# Zero-Quadratic Demo Plan

## Overview

A single-page interactive HTML5 demo showcasing the **Zero-Quadratic** methodology — pair-is-seed O(N²) relational procedural determinism. Six layered visualisation panels, all relationships computed on-demand from coordinate pairs via a pure-JS xxhash32 substitute. Zero stored edges. Zero stored graphs. Infinite relational complexity.

---

## Aesthetic Direction

**Theme**: Deep-space tactical cartography — dark void background, neon vector lines, monospace readouts. Think a cold-war era strategic overlay map fused with a sci-fi sensor suite. Palette: near-black `#080c14` bg, electric teal `#00ffe5`, amber `#ffb700`, crimson `#ff2d55`, soft violet `#7c5cbf`. Typography: `Orbitron` for headings (geometric, futuristic), `Fira Code` for data readouts.

---

## Hash Core (JS Implementation)

Replace xxhash64 with a deterministic 32-bit Murmur3-style integer hash safe in JS:

```js
function murmur32(a, b, c, d, seed = 0) {
  // Pack four int16s into two uint32 words, then mix
  let h = seed >>> 0;
  const vals = [a & 0xFFFF, b & 0xFFFF, c & 0xFFFF, d & 0xFFFF];
  for (const v of vals) {
    let k = Math.imul(v, 0xcc9e2d51);
    k = ((k << 15) | (k >>> 17)) >>> 0;
    k = Math.imul(k, 0x1b873593);
    h ^= k;
    h = ((h << 13) | (h >>> 19)) >>> 0;
    h = (Math.imul(h, 5) + 0xe6546b64) >>> 0;
  }
  h ^= 4;
  h ^= h >>> 16; h = Math.imul(h, 0x85ebca6b) >>> 0;
  h ^= h >>> 13; h = Math.imul(h, 0xc2b2ae35) >>> 0;
  h ^= h >>> 16;
  return h >>> 0;
}

function pairHash(ax, ay, bx, by, salt = 0) {
  // Symmetric: sort pair before hashing
  const [p1, p2] = [[ax, ay], [bx, by]].sort((a, b) => a[0] - b[0] || a[1] - b[1]);
  return murmur32(p1[0], p1[1], p2[0], p2[1], salt);
}

function asymPairHash(ax, ay, bx, by, salt = 0) {
  return murmur32(ax, ay, bx, by, salt);
}

function hashToFloat(h) {
  return (h >>> 0) / 0x100000000;
}
```

---

## Six Demo Panels

### Panel 1 — Faction Tension & Diplomacy

**Concept**: 8 city-states on a canvas. Each pair has a deterministic tension score (0–1). Lines drawn between pairs above threshold, coloured by tension (green=alliance, amber=cold war, red=hostility). Hovering a city highlights all its relationships with live tension readouts.

**Algorithm**:
```
tension(A, B) = 0.4 * hashToFloat(pairHash(ax,ay,bx,by, SEED+1))
              + 0.4 * |terrain(A) - terrain(B)|   // cultural gap via O(1)
              + 0.2 * hashToFloat(pairHash(ax,ay,bx,by, SEED+3))
```

**UI**: Canvas overlay on map grid. City nodes as hexagons. Animated pulse on hostile pairs. Sidebar: sorted tension leaderboard.

---

### Panel 2 — NPC Social Graph

**Concept**: 12 NPCs arranged in a circle. Abstract-ID pair hash (no spatial coords). Relationships: allies (bond > 0.7), rivals (rivalry > 0.7), neutral. Line thickness = strength. Click NPC to see their full relationship web + role (inferred from point hash O(1)).

**Algorithm**:
```
bond(A, B)    = hashToFloat(pairHash(idA, 0, idB, 0, SEED))
rivalry(A, B) = hashToFloat(pairHash(idA, 1, idB, 1, SEED))
history(A, B) = hashToFloat(pairHash(idA, 2, idB, 2, SEED))
```

**UI**: Force-directed circle layout. Node icons determined by O(1) point hash (Merchant/Guard/Mage/Thief). Animated relationship arcs. Relationship type badge on hover.

---

### Panel 3 — Trade Route Viability (Directional)

**Concept**: 6 port cities. Asymmetric directional trade flow — A→B ≠ B→A (complementarity-based). Arrow thickness = viability. Colour = profitability tier. Distance penalty applied.

**Algorithm**:
```
resource(A→B) = hashToFloat(asymPairHash(ax,ay,bx,by, SEED+10))
resource(B→A) = hashToFloat(asymPairHash(bx,by,ax,ay, SEED+10))
complementarity = max(0, resource(A→B) - resource(B→A))
distancePenalty = min(1, dist/500)²
viability = complementarity * (1 - distancePenalty)
```

**UI**: Directed arrow graph. Animated cargo-ship icons travelling active routes (viability > 0.3). Hovering route shows resource differential tooltip.

---

### Panel 4 — Ley Line Network

**Concept**: 10 magical nodes on terrain. Sparse threshold graph — only pairs with strength > 0.65 show a ley connection. Connections have frequency (pulse speed) and bidirectionality. Threshold slider to prune/reveal connections live.

**Algorithm**:
```
strength(A,B)  = hashToFloat(pairHash(ax,ay,bx,by, SEED))
freq(A,B)      = hashToFloat(pairHash(ax,ay,bx,by, SEED+99))
bidir(A,B)     = hashToFloat(pairHash(ax,ay,bx,by, SEED+7)) > 0.5
visible = strength > threshold
```

**UI**: Glowing animated lines with pulse animation keyed to `freq`. Bidirectional lines glow brighter. Threshold slider (0.0–0.9) — watch the network topology change without recomputing, just re-querying.

---

### Panel 5 — Signal Propagation Between Nodes

**Concept**: 8 signal tower nodes. Signal strength between pairs decays with distance (quadratic falloff). Visualised as a heat-map overlay: each pixel's received signal = sum of contributions from all towers weighted by pair relationship strength.

**Algorithm**:
```
signalStrength(A, B) = hashToFloat(pairHash(ax,ay,bx,by, SEED+20))
                     * (1 - (dist/MAX_DIST)²)   // quadratic falloff
```

**UI**: Canvas heatmap rendered per-pixel (downsampled 4×4 grid for perf). Tower nodes overlaid. Clicking a tower shows its per-pair signal contribution table. Seed slider regenerates entire field instantly.

---

### Panel 6 — Political Pressure Map

**Concept**: 5 faction capitals on a grid. Each grid cell's "political pressure" = weighted sum of all capital-to-cell pair hashes (treating grid cells as second entity). Cells coloured by dominant faction (highest pressure wins). Contested zones shown as blended colours.

**Algorithm**:
```
pressure(capital_i → cell_j) = hashToFloat(pairHash(ci_x, ci_y, cj_x, cj_y, SEED+i))
                              * (1 - dist(i,j)/MAX_DIST)
dominantFaction = argmax over all capitals
```

**UI**: Full-canvas Voronoi-style pressure map. Faction capitals as large glowing icons. Animates smoothly when world seed changes. Legend shows faction names + territory %. 

---

## Layout & Navigation

- Single HTML file, no build step, no dependencies except Google Fonts CDN
- Fixed left sidebar: title, world seed slider (0–9999), active panel selector (6 buttons)
- Main canvas area: full-height panel display
- Bottom status bar: `PAIRS COMPUTED THIS FRAME: N`, `BYTES STORED: 0`, `SEED: XXXX`
- Panel transitions: CSS clip-path wipe animation

---

## Technical Constraints

- Pure vanilla JS + Canvas2D + CSS animations — no libraries
- All pair relationships computed **on hover/on render** — never cached in arrays
- World seed is a single integer; changing it regenerates all relationships instantly
- Target 60fps for heatmap panel at 4×4 downsample on modern hardware
- Mobile-responsive down to 768px width (panels stack vertically)

---

## Key ZeroQuadratic Principles Demonstrated

| Panel | Principle |
|---|---|
| Faction Tension | Symmetric pair hash + cultural gap from O(1) terrain |
| NPC Social Graph | Abstract-ID pairs (non-spatial), three-salt multi-channel |
| Trade Routes | Asymmetric directional hash, distance penalty |
| Ley Lines | Threshold sparse graph, frequency channel, bidirectionality |
| Signal Propagation | Quadratic distance falloff, field summation |
| Political Pressure | Argmax multi-faction, Voronoi emergence from pair hashes |

**Zero bytes stored. All relationships computed on demand. Same seed = same world, anywhere.**
