# ZeroByte Family

*A deep analysis of the ZeroBytes methodology lineage, its existing descendants, and new research directions.*

---

## I. The Founding Axiom

ZeroBytes established a philosophical inversion that deserves to be stated plainly before anything else:

> **The universe springs complete from coordinates. Zero bytes store infinity.**

This is not merely a performance optimisation. It is an ontological claim: that a procedural world does not need to be *stored* because it can be *computed*, and that computation is only valid if it is *stateless, deterministic, and coordinate-addressed*. The consequence is that any property that can be derived from position — and in a procedural world, all properties *should* be derivable from position — can be recovered in O(1) time from anywhere, at any time, by any machine.

The Five Laws that enforce this are:

1. **O(1) Access** — no iteration over preceding states
2. **Parallelism** — no sibling dependency during generation
3. **Coherence** — adjacent coordinates produce related values (noise layering)
4. **Hierarchy** — child seeds derive from parent seeds plus local position
5. **Determinism** — same inputs produce same outputs everywhere

These five laws are the constitution of the family. Every descendant must ratify them.

---

## II. The First Descendant: Zero-Quadratic

### The Conceptual Leap

ZeroBytes asks: *What is at this position?*
Zero-Quadratic asks: *What exists between two positions?*

This is the single insight that defines the descendant relationship. ZeroBytes treats entities as atomic — a terrain tile, a dungeon room, a star system. Zero-Quadratic treats *relationships* as first-class generatable entities. The pair `(A, B)` becomes a coordinate in relationship-space, and that coordinate is hashed to produce a deterministic property of the *edge* between A and B, without storing that edge anywhere.

### The Inheritance

Zero-Quadratic inherits all five laws intact:

- **O(N²) Access** is still O(1) per pair — N² is the *budget*, not the per-query cost
- **Parallelism** holds: each pair relationship depends only on its two endpoints, never on other edges during generation
- **Coherence** is achieved by composing with ZeroBytes' `coherent_value` to modulate pair hash output regionally — cultural zones, political climate
- **Hierarchy** is extended: `hierarchical_pair_seed` propagates relational tension downward through scales, so feuding continents statistically produce hostile city-pair and NPC-pair outcomes without any stored record of the original conflict
- **Determinism** is enforced by the critical symmetry constraint: sort `(A, B)` before hashing so `rel(A, B) == rel(B, A)` for symmetric relationships

### The New Contribution

Zero-Quadratic introduces three ideas not present in ZeroBytes:

1. **Symmetry as a hash property** — the sorted pair encoding that guarantees `pair_hash(A,B) == pair_hash(B,A)` is a non-trivial design decision, distinguishing it from asymmetric variants used for directional relationships (rivers, trade winds, aggression vectors)
2. **The N declaration** — the explicit requirement to hard-code the entity count at design time converts what would be an unbounded O(N²) explosion into a known, manageable budget
3. **Threshold sparsity** — using the hash output to decide whether an edge *exists at all* (ley lines, trade connections) rather than always returning a value. This generates sparse graphs deterministically, with no adjacency list required

### The Metaphor

If ZeroBytes is the skeleton of a procedural world — the bone structure of what exists at each point — then Zero-Quadratic is the nervous system: the web of invisible forces connecting every point to every other. Terrain is skeleton. Faction tension, trade pressure, ley line resonance, and NPC social bonds are nervous system. You need both to have a living world.

---

## III. The Complexity Ladder

The family so far occupies the bottom two rungs of a natural complexity ladder:

| Tier | Name | Complexity | Seed | Answers |
|------|------|------------|------|---------|
| 0 | **ZeroBytes** | O(1) per point | coordinate | What is *at* this position? |
| 1 | **Zero-Quadratic** | O(N²) budget, O(1) per pair | coordinate pair | What *connects* two positions? |
| 2 | **Zero-Cubic** *(proposed)* | O(N³) budget | coordinate triple | What *emerges from three* positions? |
| 3 | **Zero-Temporal** *(proposed)* | O(1) per point+time | coordinate + epoch | What is at this position *at this moment*? |
| — | **Zero-Spectral** *(proposed)* | O(N log N) per query | frequency domain | What is the *pattern* across this region? |

The ladder is not a requirement to build all tiers. It is a map of the logical space.

---

## IV. New Research Directions

### 4.1 Zero-Cubic — Triple-is-Seed (O(N³) budget)

**The question it answers:** What property emerges from the *configuration* of three entities?

**Motivation:** Many game mechanics are inherently triadic. Coalition tension in diplomacy requires three factions, not two — `tension(A,B,C)` is not reducible to `tension(A,B) + tension(B,C)`. Triangular trade in economics requires three ports forming a circuit. Narrative conflict requires protagonist, antagonist, and the context (the third point) that makes their relationship meaningful. Chemical reactions in procedural crafting require three reagents.

**Core formula:**
```python
def triple_hash(ax, ay, bx, by, cx, cy, salt=0):
    """Symmetric across all permutations of the three points."""
    pts = sorted([(ax, ay), (bx, by), (cx, cy)])
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqqqqq', *pts[0], *pts[1], *pts[2]))
    return h.intdigest()
```

**Symmetry requirement:** Sort all three coordinate pairs before hashing, so `triple(A,B,C) == triple(A,C,B) == triple(B,A,C)` etc.

**Natural applications:**
- Coalition diplomacy: the stability of a three-faction alliance as a single deterministic value
- Triangular trade circuits: does this three-port route form a viable economic loop?
- Procedural religion: does a three-site configuration activate a sacred triangle?
- Group dynamics: does a three-NPC configuration produce a stable social triad or a volatile one?

**Budget consideration:** N³ grows fast. N=16 gives 4096 triples — still manageable for a room-scale system. N=64 gives 262,144 — borderline. The proximity guard is mandatory: `if any pair in triple exceeds max_dist: return 0.0`.

---

### 4.2 Zero-Temporal — Time-as-Dimension

**The question it answers:** What is at this position *at this moment in world-time?*

**Motivation:** ZeroBytes is stateless across time — the terrain is always the terrain. But many properties are intrinsically temporal: weather, seasonal biome shifts, tidal cycles, economic boom-bust rhythms, dynastic rise-and-fall. Zero-Temporal treats time as an additional coordinate dimension, preserving O(1) access while adding a fourth axis.

**Core formula:**
```python
def temporal_hash(x, y, z, epoch, salt=0):
    """epoch is a discretised world-time tick — not wall-clock time."""
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqqq', x, y, z, epoch))
    return h.intdigest()
```

**Key design insight:** `epoch` must be a *world-defined* integer, not real time. The world defines its own temporal granularity — perhaps one epoch per in-game day, or per season. This preserves determinism: the same world-time tick produces the same result everywhere, regardless of when the query is made in real time.

**Layering with ZeroBytes:**
```python
def weather(x, y, epoch, world_seed):
    base_climate = coherent_value(x*0.005, y*0.005, world_seed)    # O(1), ZeroBytes
    seasonal_cycle = math.sin(epoch * (2*math.pi / 365))            # period = 1 year
    storm_seed = temporal_hash(x//32, y//32, 0, epoch//7, world_seed+1)
    storm_chance = hash_to_float(storm_seed)
    return {
        "temperature": base_climate + 0.3 * seasonal_cycle,
        "storm": storm_chance > 0.85
    }
```

**Applications:**
- Weather systems that are fully deterministic and reconstructable at any past epoch
- Seasonal terrain state (snow cover, flood plains, crop cycles)
- NPC schedules and patrol routes derived from time-of-day epoch
- Economic cycles: deterministic boom-bust tied to world epoch, without simulation
- Historical archaeology: what did this tile look like 500 epochs ago? — just query with the old epoch

**Critical anti-pattern to avoid:** `epoch = int(real_time.time())` — this breaks determinism. Epoch must always be a world-internal value derived from simulated ticks, not wall-clock time.

---

### 4.3 Zero-Spectral — Frequency Domain Hashing

**The question it answers:** What is the *pattern* across a region, rather than the value at a point?

**Motivation:** ZeroBytes' coherent noise is already a primitive form of this — it layers octaves of spatial frequency to produce regionally coherent values. Zero-Spectral formalises this into a first-class methodology: rather than hashing individual positions and then smoothing, it operates directly in frequency space, generating the *spectrum* of a region and then reconstructing point values via inverse transform.

**Core idea:**
```python
def spectral_region(region_x, region_y, region_seed, resolution=64):
    """Generate a 2D region by seeding frequency components deterministically."""
    frequencies = []
    for freq_band in range(8):          # 8 octave bands
        band_seed = position_hash(region_x, region_y, freq_band, region_seed)
        amplitude = hash_to_float(band_seed) / (2 ** freq_band)
        phase     = hash_to_float(band_seed >> 32) * 2 * math.pi
        frequencies.append((amplitude, phase, 2**freq_band))
    # Reconstruct: each point is a sum of sinusoidal components
    return frequencies  # caller reconstructs at any resolution
```

**Why this is different from coherent noise:** Coherent noise adds octave contributions point-by-point. Zero-Spectral generates the frequency decomposition of the *entire region* from a single region seed, then lets you reconstruct any point within it analytically. The region's spectral fingerprint is fixed; its spatial resolution is variable.

**Applications:**
- Adaptive level of detail: query a region at coarse resolution for overview, fine resolution for detail, same seed
- Terrain power spectra: different biomes have characteristic frequency signatures (smooth plains vs jagged mountains vs fractal coastlines encoded as spectral shapes)
- Procedural music: generate a musical region whose tonal character derives from world position — caves sound different from forests, encoded as frequency profiles rather than audio samples
- Signal propagation: ley line interference patterns, magical resonance fields, or radio-frequency terrain for sci-fi settings

---

### 4.4 Zero-Graph — Topology-is-Seed

**The question it answers:** What is the *topological structure* of a local region — which nodes connect, and how?

**Motivation:** Zero-Quadratic can determine the *strength* of a relationship between any two entities. But it does not generate the *graph structure* itself — which nodes exist, and which connections exceed the threshold to form the network. Zero-Graph proposes seeding the entire local graph topology from a single region seed, generating node count, positions, and edge existence as a unified deterministic structure.

**Core idea:**
```python
def local_graph(region_x, region_y, region_seed, max_nodes=16):
    """Generate a local node-edge graph deterministically from region position."""
    region_s = position_hash(region_x, region_y, 0, region_seed)
    n_nodes   = 2 + int(hash_to_float(region_s) * (max_nodes - 2))
    nodes = []
    for i in range(n_nodes):
        node_s = position_hash(region_s, i, 0, 0)
        nodes.append({
            "id":   i,
            "x":    hash_to_float(node_s) * 100,
            "y":    hash_to_float(node_s >> 16) * 100,
            "type": int(hash_to_float(node_s >> 32) * 4)
        })
    # Edges: use pair_hash on node IDs, threshold to produce sparse graph
    edges = []
    for a in range(n_nodes):
        for b in range(a+1, n_nodes):
            edge_strength = hash_to_float(pair_hash(a, 0, b, 0, region_s))
            if edge_strength > 0.6:
                edges.append((a, b, edge_strength))
    return nodes, edges
```

**Applications:**
- Dungeon room connection graphs: the layout of a dungeon wing is a graph seeded from wing coordinates
- Settlement road networks: which towns connect to which via which roads, generated from regional seed
- Neural-style connection diagrams for sci-fi computers or magical networks
- Quest structure generation: the dependency graph of a procedural quest seeded from quest location

---

### 4.5 Zero-Causal — Dependency Chain Hashing

**The question it answers:** What *caused* this, and what will it *cause next?*

**Motivation:** All existing family members are synchronic — they describe what exists at a moment or between positions. Zero-Causal introduces diachronic reasoning: a deterministic causal chain where each event is seeded by its predecessor's hash, but the entire chain is reconstructable from the chain's origin seed without stepping through every intermediate event.

**Core idea:**
```python
def causal_chain(origin_seed, depth):
    """Jump directly to any event in a causal chain. O(1) per event, no iteration."""
    # The key insight: hash(origin_seed, depth) gives the event at depth directly
    event_seed = position_hash(origin_seed, depth, 0, 0)
    return {
        "event_type": int(hash_to_float(event_seed) * 8),
        "magnitude":  hash_to_float(event_seed >> 16),
        "causes_next": hash_to_float(event_seed >> 32) > 0.3  # branching probability
    }
```

**The critical property:** Unlike a Markov chain, Zero-Causal does not require stepping through events 0→1→2→...→N to reach event N. Event N is directly addressable via `position_hash(origin, N, 0, 0)`. The chain is deterministic *and* O(1)-accessible at any depth.

**Applications:**
- Procedural history: the history of a city is a causal chain seeded by its founding. Reconstruct any historical epoch without simulating intermediate history.
- Genetic lineage: the evolutionary line of a procedural species, accessible at any generation
- Technology trees: which invention led to which, for a given civilisation seed
- Criminal investigation gameplay: a crime's causal chain is seeded at the crime's location and epoch, allowing the player to reconstruct the chain of events

---

### 4.6 Zero-Field — Continuous Influence Fields

**The question it answers:** What is the aggregate *influence* on this point from all nearby entities?

**Motivation:** Both ZeroBytes and Zero-Quadratic are entity-first — they describe specific positions or pairs. Zero-Field is field-first: it generates a continuous scalar (or vector) field that represents the accumulated influence of all entities in a region on any given point, without iterating over entities at query time.

**Core idea:**

Rather than summing contributions from nearby entities (which requires knowing which entities exist), Zero-Field treats influence as a property of space itself — a spatial function that happens to be coherent near entity positions and falls off deterministically away from them.

```python
def influence_field(x, y, world_seed, field_type_salt):
    """Continuous influence field — no entity enumeration required."""
    # Generate field as layered spatial hash, not as sum of entity contributions
    macro = coherent_value(x*0.002, y*0.002, world_seed + field_type_salt, octaves=3)
    meso  = coherent_value(x*0.02,  y*0.02,  world_seed + field_type_salt + 1, octaves=2)
    micro = coherent_value(x*0.1,   y*0.1,   world_seed + field_type_salt + 2, octaves=1)
    return 0.6*macro + 0.3*meso + 0.1*micro
```

**The inversion:** Instead of computing which entities are nearby and summing their influence, the field value at any point is computed directly. Entity *positions* are then derived as local maxima of the field — entities exist where the field is strongest, not the other way round. This is architecturally opposite to traditional entity-first design.

**Applications:**
- Magic intensity maps: mana density is a field; magical entities spawn at field peaks
- Pollution or disease spread without simulation: concentration is a spatial hash, not a diffusion computation
- Gravity-analogues for space games: mass distribution as field, not as enumerated body sum
- Faction control maps: territory control as a continuous field, not as a per-tile owner assignment

---

## V. The Family Tree

```
ZeroBytes (O(1) — position-is-seed)
│
│   CONFIRMED DESCENDANTS
│
├── Zero-Quadratic (O(N²) budget — pair-is-seed)
│   ├── Symmetric relationships (faction tension, NPC bonds)
│   ├── Asymmetric relationships (trade flow, aggression vectors)
│   ├── Threshold sparsity (ley lines, connection graphs)
│   └── Hierarchical pair inheritance (tension bleeds down scales)
│
│   PROPOSED DESCENDANTS
│
├── Zero-Cubic (O(N³) budget — triple-is-seed)
│   ├── Coalition dynamics
│   ├── Triangular trade circuits
│   └── Sacred geometry / three-site configurations
│
├── Zero-Temporal (O(1) — coordinate+epoch-is-seed)
│   ├── Deterministic weather systems
│   ├── Seasonal terrain state
│   ├── Historical archaeology (query any past epoch)
│   └── NPC schedules by time-of-day
│
├── Zero-Spectral (O(N log N) — frequency-is-seed)
│   ├── Adaptive level of detail
│   ├── Biome spectral fingerprints
│   ├── Procedural audio character maps
│   └── Signal interference patterns
│
├── Zero-Graph (topology-is-seed)
│   ├── Dungeon wing layout graphs
│   ├── Settlement road networks
│   └── Quest dependency structures
│
├── Zero-Causal (O(1) at any depth — chain-is-seed)
│   ├── Procedural civilisation history
│   ├── Genetic lineage at any generation
│   └── Criminal causal chain reconstruction
│
└── Zero-Field (continuous field — space-is-seed)
    ├── Mana / magic intensity maps
    ├── Disease / pollution spread without simulation
    └── Faction territory as continuous function
```

---

## VI. Cross-Cutting Concerns for All Descendants

Any new family member must resolve the same five tensions that ZeroBytes and Zero-Quadratic have already resolved:

### 6.1 Symmetry Declaration

Every new skill must explicitly choose and enforce its symmetry model:
- **Fully symmetric** (all permutations of inputs produce the same output): sort all inputs before hashing
- **Directional** (input order matters): do not sort; document this explicitly
- **Partially symmetric** (some input positions are ordered, others are not): use composite hash with sorted and unsorted components

### 6.2 Budget Declaration

The O-notation of the complexity tier must be declared as a *budget*, not an open cost. Every descendant must provide:
- A table mapping system scale to entity count (N) to operation count
- A hard-cap requirement: N must be declared at design time, not grown at runtime
- A proximity guard pattern: the standard way to exclude distant pairs/triples/etc.

### 6.3 Hierarchy Compatibility

New descendants must specify how they compose with ZeroBytes' hierarchy pattern:
- Do child seeds inherit from parent seeds, or are they independent?
- If inheritance is used, which component of the parent seed propagates (full seed? upper bits? lower bits?)
- Does the descendant introduce new hierarchy levels, or does it operate within existing ones?

### 6.4 Determinism Verification Pattern

Every descendant must ship a verification recipe — the equivalent of ZeroBytes' `verify()` and Zero-Quadratic's `verify_quadratic()`. This is not optional. The verification recipe should test:
- Same inputs → same output
- Order independence (for symmetric variants)
- Cross-machine stability (document which hash library and packing format ensure this)

### 6.5 Anti-Pattern Catalogue

Every descendant must enumerate its characteristic failure modes — the equivalent of ZeroBytes' `random.random()` and `hash(tuple)` anti-patterns. These tend to cluster around:
- Platform-dependent hash functions
- Unbounded N growth
- Storing computed values (defeats the zero-bytes principle)
- Time-based or execution-order-based seeds

---

## VII. The Philosophical Core

The ZeroByte family is unified not by a complexity class but by a *philosophy of elimination*:

| What is eliminated | Why | By which member |
|---|---|---|
| Stored world state | Memory is infinite; computation is O(1) | ZeroBytes |
| Stored relationship graphs | Edges are infinite; pairs are computable | Zero-Quadratic |
| Stored simulation history | Time is a coordinate, not a log | Zero-Temporal |
| Stored adjacency lists | Topology is hash-generatable | Zero-Graph |
| Entity enumeration for fields | Fields are spatial, not entity-derived | Zero-Field |

In each case, the elimination is made possible by the same move: *treat the identifier of the thing as its seed*. Coordinates seed terrain. Coordinate pairs seed relationships. Coordinate triples seed coalitions. Coordinate+epoch pairs seed history. The identifier *is* the computation.

This is why the family's motto is recursive:

> **Zero bytes store infinity.**

Every member of the family stores zero bytes. Every member generates infinite content. The content is not stored — it is derived, on demand, from the coordinates that identify it.

---

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{zerobytes-family-skills,
  title = {ZeroBytes Family Skills: Zero Skill Collection},
  author = {[Drift Johnson]},
  year = {2025},
  url = {[https://github.com/MushroomFleet/project-name](https://github.com/MushroomFleet/ZeroBytes-Family-Skills)},
  version = {1.0.0}
}
```

### Donate:


[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
