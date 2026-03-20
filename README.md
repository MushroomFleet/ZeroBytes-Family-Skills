# ZeroBytes Family Skills

> *The universe springs complete from coordinates. Zero bytes store infinity.*

A collection of Claude Skills implementing the **ZeroBytes Family** of procedural generation methodologies — a unified philosophy of deterministic, stateless, coordinate-addressed world generation. Every member of this family shares one principle: **the identifier of a thing is its seed**. Nothing is stored. Everything is computable.

---

## Philosophy

Traditional procedural generation systems accumulate state. They simulate, propagate, store, and cache. The ZeroBytes Family eliminates all of that. Each skill answers a question purely from a coordinate — spatial, temporal, relational, or spectral — using a deterministic hash. The result is identical on every machine, at any moment, in any execution order, with zero bytes of stored world state.

The family grows by extending the coordinate type:

| Skill | Seed Type | Core Question |
|-------|-----------|---------------|
| **ZeroBytes** | position | What is *at* this coordinate? |
| **Zero-Quadratic** | coordinate pair | What exists *between* two positions? |
| **Zero-Cubic** | coordinate triple | What *emerges* from three positions together? |
| **Zero-Temporal** | position + epoch | What is here *at this moment* in world-time? |
| **Zero-Spectral** | region frequency bands | What is the *spectral character* of this region? |
| **Zero-Graph** | region coordinate | What *nodes and edges* exist in this region? |
| **Zero-Causal** | origin + depth | What *event* occurred at step N in this chain? |
| **Zero-Field** | spatial coordinate | What continuous *influence* permeates this point? |

---

## Skills

### 🧱 ZeroBytes
**Position-is-seed procedural generation**

The foundation of the entire family. ZeroBytes replaces sequential state mutation with pure coordinate hashing — the coordinate IS the seed. Using `xxhash` with a deterministic struct-packed input, any property at any position in an infinite world is computable in O(1) without iteration, caching, or stored state.

**Intended applications:** Infinite tilemaps, procedural terrain and biomes, roguelike dungeon rooms, star systems, ore veins, chunk-based world generation, item drop tables. Any system where a spatial position should deterministically define content.

**The Five Laws:** O(1) access · Parallelism · Coherence · Hierarchy · Determinism

---

### 🔗 Zero-Quadratic
**Pair-is-seed relational procedural determinism**

The first descendant. Extends ZeroBytes from point properties into relationship properties. The coordinate *pair* `(A, B)` is hashed to produce a deterministic property of the *edge* between two entities — without storing any graph or adjacency matrix. Introduces symmetric hashing (sorted pair → same hash regardless of query order) and N-declaration as a budget constraint.

**Intended applications:** Faction tension and diplomacy, NPC social graphs, trade route viability, ley line networks, signal propagation between nodes, political pressure maps. Any system where a property lives *between* positions rather than *at* them.

**Key contribution:** Symmetry-as-hash-property · N budget declaration · Threshold sparsity for edge existence

---

### 🔺 Zero-Cubic
**Triple-is-seed triadic procedural determinism**

Extends Zero-Quadratic from pairwise relationships to emergent properties of three-entity configurations. `property(A,B,C)` is not reducible to any sum of pairwise values — it is genuinely triadic, seeded directly from the sorted triple coordinate. Introduces all-pairs proximity guards and full 6-permutation symmetry enforcement.

**Intended applications:** Coalition diplomacy and three-faction stability, triangular trade circuits, sacred geometry activation, NPC group dynamics (who leads, who is scapegoated), three-reagent procedural crafting reactions, tectonic triple-junction geology.

**Key contribution:** Full permutation symmetry · All-pairs proximity guard · Hierarchy from triple seeds

---

### ⏱️ Zero-Temporal
**Coordinate+epoch-is-seed temporal procedural determinism**

Adds world-time as a fourth coordinate dimension. The position+epoch pair IS the seed. Any world state at any moment — past, present, or future — is directly computable without simulation, replay, or stored logs. Introduces the critical distinction between *cyclic* properties (sin/cos of epoch — seasons, tides) and *stochastic* properties (temporal hash — storms, economic shocks). `epoch` must always be a world-internal integer tick, never wall-clock time.

**Intended applications:** Deterministic weather systems, seasonal terrain state (snow cover, flood plains), NPC daily schedules and patrol routes, historical archaeology (query any past epoch directly), economic boom-bust cycles, dynastic rise-and-fall without simulation.

**Key contribution:** Time-as-coordinate · Cyclic vs stochastic split · O(1) historical access

---

### 🌊 Zero-Spectral
**Frequency-domain-is-seed procedural generation**

Operates in the frequency domain rather than the spatial domain. Instead of hashing individual points and smoothing afterward, Zero-Spectral seeds the entire spectral decomposition of a region from a single region coordinate — then reconstructs any point within it analytically at any resolution. The region's frequency profile (its amplitude shape across octave bands) becomes a first-class designable property.

**Intended applications:** Adaptive level of detail (same seed, variable resolution), biome spectral fingerprinting (smooth plains vs jagged mountains vs fractal coastlines as distinct amplitude profiles), procedural audio character maps (caves sound different from forests), signal interference patterns, region-scale pattern generation where texture character matters.

**Key contribution:** Region-first access · Resolution independence · Designable spectral profiles

---

### 🕸️ Zero-Graph
**Topology-is-seed procedural graph generation**

Generates complete node-edge graph structures deterministically from a region coordinate. Where Zero-Quadratic queries edge properties between *pre-existing* nodes, Zero-Graph generates the node set and graph topology simultaneously from a single seed — no adjacency list, no node registry. Includes `ensure_connectivity` for guaranteed spanning trees and `inter_region_edges` for cross-boundary connections using both region seeds.

**Intended applications:** Dungeon wing room-connection layouts, settlement road networks, quest dependency graphs (directed DAGs), faction alliance and rivalry webs, cave system topologies, river delta branching structures, subway and trade network layouts.

**Key contribution:** Node set + topology from single seed · Connectivity guarantee · Boundary-coherent inter-region edges

---

### 🧬 Zero-Causal
**Chain-is-seed deterministic causal event generation**

Generates discrete narrative event chains where any event at any depth is directly computable in O(1) without replaying preceding events. Unlike a Markov chain, Zero-Causal does not require sequential computation to reach step N — `event_at_depth(origin_seed, N)` jumps there directly. Introduces transition bias tables that create causal plausibility (wars tend to follow border disputes) using only the *type index* of the previous event, not its full value.

**Intended applications:** Procedural civilisation history (jump to any historical era directly), genetic lineage at any generation, criminal causal chain reconstruction for investigation gameplay, technology tree evolution, narrative event chains, branching alternate-history timelines.

**Key contribution:** O(1) depth access · Causal plausibility without sequential compute · Branching via salt · Historical archaeology

---

### 🌌 Zero-Field
**Space-is-seed continuous influence field generation**

The philosophical inversion of entity-first design. Entities do not produce fields — fields produce entities. Mana doesn't radiate from magical creatures; magical creatures exist where mana peaks. Faction territory is not a union of claimed tiles; it is a continuous field whose border is where two fields reach equilibrium. Field strength at any point is computed in O(1) from spatial coordinates alone via layered coherent noise at macro, meso, and micro scales. Field orthogonality is enforced through widely-separated salt constants.

**Intended applications:** Mana and magic intensity maps (entities spawn at field peaks), disease and pollution spread without simulation, faction territory as continuous function (no tile-ownership tables), wind and ocean current vector fields, heat and radiation zones for caves and volcanic terrain, ecosystem pressure fields (predator density, prey abundance, vegetation coverage).

**Key contribution:** Field-first architecture · Entity spawning at peaks · Orthogonal multi-field composition · Vector field support

---

## Repository Contents

```
ZeroBytes-Family-Skills/
├── ZeroByte-Family-grounding.md   # Deep analysis of the family lineage and research directions
├── zerobytes.skill                # The foundational skill
├── zero-quadratic.skill           # Pairwise relationships
├── zero-cubic.skill               # Triadic emergent properties
├── zero-temporal.skill            # Time as a coordinate
├── zero-spectral.skill            # Frequency-domain generation
├── zero-graph.skill               # Procedural graph topology
├── zero-causal.skill              # Causal chain histories
└── zero-field.skill               # Continuous influence fields
```

Each `.skill` file is a zip archive containing a `SKILL.md` — install directly into your Claude Skills directory.

---

## The Family Tree

```
ZeroBytes  (O(1) — position-is-seed)
│
├── Zero-Quadratic  (O(N²) — pair-is-seed)
│
├── Zero-Cubic      (O(N³) — triple-is-seed)
│
├── Zero-Temporal   (O(1) — position+epoch-is-seed)
│
├── Zero-Spectral   (O(band_count) — frequency-is-seed)
│
├── Zero-Graph      (O(N²) per region — topology-is-seed)
│
├── Zero-Causal     (O(1) at any depth — chain-is-seed)
│
└── Zero-Field      (O(1) — space-is-seed)
```

All descendants inherit the Five Laws of ZeroBytes: **O(1) access · Parallelism · Coherence · Hierarchy · Determinism**. Each extends the seed type into a new dimension of generative space.

---

## Usage

Install any `.skill` file by extracting it into your Claude Skills directory. Each skill is self-contained and references its parent skills for context. They are designed to be composed — use ZeroBytes for what exists, Zero-Quadratic for how things relate, Zero-Field for ambient influence, Zero-Temporal for how it all changes over time.

For the theoretical grounding and research directions behind each skill, see **`ZeroByte-Family-grounding.md`**.

---

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{ZeroBytes_Family_Skills,
  title = {ZeroBytes Family Skills: Deterministic Procedural Generation Methodology},
  author = {[Drift Johnson]},
  year = {2025},
  url = {https://github.com/MushroomFleet/ZeroBytes-Family-Skills},
  version = {1.0.0}
}
```

### Donate:

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
