# ZeroGraph Demo Plan

## Overview

An interactive single-page HTML demo showcasing four Zero-Graph topology-is-seed graph types. All graphs are generated **deterministically** from region coordinates — no adjacency lists, no stored structures. Every graph re-generates identically from its seed.

---

## Demo Sections (Tabs)

### 1 · Dungeon Wing — Room-Connection Layouts

**Purpose:** Show how a dungeon wing's rooms and corridors emerge from `(wing_x, wing_y, floor, seed)`.

**Visual:**
- Nodes: rooms drawn as rounded rectangles with icon labels
  - `empty` → grey  |  `combat` → red  |  `treasure` → gold  |  `boss` → purple  |  `shop` → teal  |  `puzzle` → blue
- Edges: corridor lines; **locked doors** drawn as dashed with a padlock icon
- Entrance node: highlighted with a glow ring
- Controls: **Wing X / Wing Y / Floor** sliders + **Seed** input — any change regenerates instantly

**Algorithm (JS port):**
```js
function positionHash(x, y, z, salt = 0) { /* xxhash64-inspired 32-bit mix */ }
function pairHash(ax, ay, bx, by, salt = 0) { /* sorted pair → 32-bit mix */ }
function generateGraph(rx, ry, seed, maxNodes, size, threshold) { ... }
function ensureConnectivity(graph) { /* spanning-tree bridge join */ }
function dungeonWing(wx, wy, floor, seed) { ... }
```

---

### 2 · Settlement Road Network

**Purpose:** Show how a map region's towns and roads emerge from `(region_x, region_y, world_seed)`.

**Visual:**
- Nodes: settlement circles, sized by `population`; type badge: `hamlet / village / town / city / fortress`
- Fortified settlements get a double-ring outline
- Edges: road lines coloured by `road_quality` (green → yellow → red) with `bandit_risk` opacity
- Controls: **Region X / Region Y** sliders + **World Seed** input

---

### 3 · Quest Dependency Graph (Directed DAG)

**Purpose:** Show how quest objectives and their prerequisites are determined by `(origin_x, origin_y, world_seed)`.

**Visual:**
- Nodes arranged in **topological layers** (prerequisite at top, final objectives at bottom)
- Node colour by type: `collect` → amber | `kill` → crimson | `escort` → cyan | `investigate` → violet | `deliver` → lime | `protect` → orange
- Optional objectives: dashed outline
- Directed edges: arrows pointing from prerequisite → unlocked; thickness = `strength`
- Controls: **Origin X / Origin Y** sliders + **Seed** input

---

### 4 · Cave System Topology

**Purpose:** Show organic cave chamber networks with high connectivity, using a lower `edge_threshold`.

**Visual:**
- Nodes: cave chambers as irregular blob shapes (SVG path, Perlin-distorted circle)
- Chamber types: `main passage` | `side chamber` | `underground lake` | `crystal pocket` | `dead end`
- Edges: tunnel lines with width proportional to `strength`
- Forced connectivity edges shown as dotted red bridges
- Controls: **Cave X / Cave Y** sliders + **Depth** + **Seed** input

---

## UI Design Direction

**Aesthetic:** Dark sci-fi dungeon map / cartography terminal. Think old-parchment meets CRT scanlines.

**Palette:**
```
--bg:       #0d0f14
--surface:  #141820
--border:   #2a3040
--accent:   #c8a96e   /* aged gold */
--accent2:  #5ba4cf   /* cold blue */
--text:     #d4cfc4
--muted:    #6b7280
```

**Typography:**
- Display: `Cinzel` (Google Fonts) — ancient/carved stone feel
- Body/Labels: `JetBrains Mono` — technical readability
- Tab headers: Cinzel with letter-spacing

**Layout:**
- Full-viewport dark canvas
- Top tab bar with four sections
- Each tab: left panel (controls) + main canvas (SVG graph render)
- Animated edge draw-in on load/regenerate (stroke-dashoffset animation)
- Node pop-in with staggered CSS delay
- Seed changes trigger a fast fade-dissolve transition

**Canvas rendering:** Pure SVG (no canvas API needed at this scale). Viewbox auto-fitted to node positions.

---

## Core JS Architecture

```
ZeroHash          — positionHash(x,y,z,salt), pairHash(ax,ay,bx,by,salt)
ZeroGraph         — generateGraph(), ensureConnectivity()
DungeonWing       — enriches nodes/edges with room metadata
SettlementNetwork — enriches with town/road metadata
QuestDAG          — directed dependency generation + topological layout
CaveSystem        — enriches with chamber types, low threshold = dense graph

GraphRenderer     — SVG rendering engine shared across all four graphs
  renderNodes(nodes, config)
  renderEdges(edges, nodes, config)
  animateIn()      — staggered stroke + node pop
  fitViewBox(nodes, padding)

UI                — tab switching, slider binding, seed input debounce
```

---

## Controls Panel (per tab)

| Control | Type | Range |
|---|---|---|
| Region / Wing X | Slider | 0–9 |
| Region / Wing Y | Slider | 0–9 |
| Floor / Depth / Origin | Slider | 0–5 |
| Seed | Number input | any int |
| Max Nodes | Slider | 4–16 |

**"Regenerate" button** not needed — sliders are live-bound.

---

## Hash Implementation (JS, no dependencies)

Uses a deterministic 32-bit mix derived from the xxhash spirit — implemented inline without any npm package:

```js
function mix32(h) {
  h ^= h >>> 15; h = Math.imul(h, 0x85ebca6b);
  h ^= h >>> 13; h = Math.imul(h, 0xc2b2ae35);
  h ^= h >>> 16;
  return h >>> 0;
}
function positionHash(x, y, z, salt = 0) {
  let h = salt >>> 0;
  h = mix32(h ^ (x * 2654435761 >>> 0));
  h = mix32(h ^ (y * 2246822519 >>> 0));
  h = mix32(h ^ (z * 3266489917 >>> 0));
  return h;
}
function pairHash(ax, ay, bx, by, salt = 0) {
  let [p1, p2] = [[ax, ay], [bx, by]].sort((a, b) => a[0] - b[0] || a[1] - b[1]);
  return positionHash(p1[0] ^ p2[0], p1[1] ^ p2[1], p1[0] + p2[0], salt);
}
function h2f(h) { return (h >>> 0) / 0x100000000; }
```

---

## File Output

Single self-contained `demo.html` — no build step, no external JS bundles beyond Google Fonts CDN link.
