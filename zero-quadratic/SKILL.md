---
name: zero-quadratic
description: Pair-is-seed O(N²) relational procedural determinism methodology. Extends zerobytes from point properties to relationship properties. Use when a developer asks for deterministic faction systems, procedural social graphs, trade route generation, NPC relationship networks, ley line or signal propagation networks, political tension maps, or any system where the interaction between two entities must be reproducible without storing a graph or adjacency matrix. Triggers on phrases like "relational procedural", "deterministic relationships", "pair-based generation", "faction tension", "procedural diplomacy", "NPC social graph", "trade route generation", "influence fields", "zero-quadratic", "emergent relationships", or when a zerobytes system needs to model how entities affect each other rather than what they are individually. Always use this skill when the user needs to generate properties that exist *between* positions rather than *at* positions.
---

# Zero-Quadratic: Pair-is-Seed Relational Procedural Determinism

Extends [Zerobytes](zerobytes/SKILL.md) O(1) point hashing into O(N²) relationship hashing. The coordinate *pair* IS the seed.

## The Five Extended Laws

Every relational procedural system must satisfy:

1. **O(N²) Pair Access**: Any relationship between two positions is computable directly from their coordinate pair — no graph traversal, no stored adjacency
2. **Symmetric Coherence**: `rel(A, B) == rel(B, A)` unless intentionally asymmetric; hash packing must enforce this via coordinate sorting
3. **Locality Falloff**: Relationship strength decays deterministically with distance — encoded in hash result, not post-processed
4. **Hierarchy Inheritance**: Child pair seeds derive from parent pair seeds plus local offset pair — tension bleeds downward through scales
5. **Determinism**: Same coordinate pair → same relationship value across all machines, all execution orders

## Core Pattern

```python
import struct
import xxhash

def pair_hash(ax: int, ay: int, bx: int, by: int, salt: int = 0) -> int:
    """Symmetric: pair_hash(A, B) == pair_hash(B, A). Sort before packing."""
    p1, p2 = sorted([(ax, ay), (bx, by)])
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqqq', p1[0], p1[1], p2[0], p2[1]))
    return h.intdigest()

def asymmetric_pair_hash(ax: int, ay: int, bx: int, by: int, salt: int = 0) -> int:
    """Directional: rel(A→B) ≠ rel(B→A). For rivers, trade winds, aggression."""
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqqq', ax, ay, bx, by))
    return h.intdigest()

def hash_to_float(h: int) -> float:
    return (h & 0xFFFFFFFF) / 0x100000000

def relationship_strength(ax, ay, bx, by, salt, max_dist=100.0) -> float:
    """Deterministic intensity with quadratic distance falloff. Returns 0.0–1.0."""
    dist = ((bx - ax)**2 + (by - ay)**2) ** 0.5
    if dist > max_dist:
        return 0.0
    base = hash_to_float(pair_hash(ax, ay, bx, by, salt))
    falloff = 1.0 - (dist / max_dist) ** 2
    return base * falloff
```

## The Complexity Budget

O(N²) is a **budget declaration**, not an open cost. Declare N at design time.

| Context | Typical N | Pairs | Use Case |
|---------|-----------|-------|----------|
| Room-scale | 8–16 | 64–256 | Item interactions, trap logic, NPC social graph |
| Chunk-scale | 32–64 | 1K–4K | Faction influence, resource veins, ley lines |
| Region-scale | 128–256 | 16K–65K | Trade routes, biome pressure, climate cells |
| World-scale | 512–1024 | 256K–1M | Tectonic plates, empire history, stellar neighbourhoods |

**Design rule: Never let N grow unbounded at runtime. Hard-code it.**

## Hash Selection by Symmetry Need

| Relationship Type | Hash | Example |
|-------------------|------|---------|
| Mutual (symmetric) | `pair_hash` (sorted) | Bond, hostility, distance |
| Directional (asymmetric) | `asymmetric_pair_hash` | Trade flow, river, aggression |
| Abstract IDs (non-spatial) | `pair_hash(id_a, 0, id_b, 0, salt)` | NPC social graph, item affinity |

## Quick Recipes

### Faction Tension
```python
def faction_tension(city_a_pos, city_b_pos, world_seed):
    ax, ay = city_a_pos; bx, by = city_b_pos
    hostility    = hash_to_float(pair_hash(ax, ay, bx, by, world_seed + 1))
    culture_a    = coherent_value(ax*0.01, ay*0.01, world_seed + 2, octaves=3)
    culture_b    = coherent_value(bx*0.01, by*0.01, world_seed + 2, octaves=3)
    cultural_gap = abs(culture_a - culture_b)
    roughness    = hash_to_float(pair_hash(ax, ay, bx, by, world_seed + 3))
    return 0.4 * hostility + 0.4 * cultural_gap + 0.2 * roughness
```

### Trade Route Viability (Directional)
```python
def trade_viability(port_a, port_b, world_seed):
    ax, ay = port_a; bx, by = port_b
    resource_a      = hash_to_float(asymmetric_pair_hash(ax, ay, bx, by, world_seed + 10))
    resource_b      = hash_to_float(asymmetric_pair_hash(bx, by, ax, ay, world_seed + 10))
    complementarity = max(0.0, resource_a - resource_b)
    dist            = ((bx-ax)**2 + (by-ay)**2) ** 0.5
    distance_penalty = min(1.0, dist / 500.0) ** 2
    return complementarity * (1.0 - distance_penalty)
```

### NPC Social Graph (Abstract IDs)
```python
def npc_relationship(npc_a_id, npc_b_id, location_seed):
    bond    = hash_to_float(pair_hash(npc_a_id, 0, npc_b_id, 0, location_seed))
    rivalry = hash_to_float(pair_hash(npc_a_id, 1, npc_b_id, 1, location_seed))
    history = hash_to_float(pair_hash(npc_a_id, 2, npc_b_id, 2, location_seed))
    if bond > 0.7:    return {"type": "allies",  "strength": bond}
    if rivalry > 0.7: return {"type": "rivals",  "strength": rivalry}
    return                   {"type": "neutral", "history_depth": history}
```

### Ley Line Network (Threshold Sparse Graph)
```python
def ley_connection(node_a, node_b, world_seed, threshold=0.65):
    ax, ay = node_a; bx, by = node_b
    strength = hash_to_float(pair_hash(ax, ay, bx, by, world_seed))
    dist     = ((bx-ax)**2 + (by-ay)**2) ** 0.5
    if strength < threshold:
        return None
    return {
        "strength":      strength,
        "frequency":     hash_to_float(pair_hash(ax, ay, bx, by, world_seed + 99)),
        "bidirectional": hash_to_float(pair_hash(ax, ay, bx, by, world_seed + 7)) > 0.5
    }
```

## Hierarchy Pattern (Extended)

Zerobytes hierarchy: `parent_seed → child_seed via position`
Zero-Quadratic adds: `parent_pair_seed → child_pair_seed via local pair`

```
World pair     (continent_A ↔ continent_B)
  └─ Region pair  (region_A ↔ region_B)   ← inherits continent tension
       └─ City pair  (city_A ↔ city_B)    ← inherits regional tension
            └─ NPC pair  (npc_A ↔ npc_B)  ← inherits city tension
```

```python
def hierarchical_pair_seed(parent_pair_seed, local_a, local_b):
    """Child relationship inherits parent relationship context."""
    return pair_hash(local_a, parent_pair_seed, local_b, parent_pair_seed >> 32, 0)
```

Cities in feuding continents generate statistically more hostile NPC pairs — through seed inheritance, not stored data.

## Composing with Zerobytes O(1)

| Layer | Complexity | Answers | Example |
|-------|-----------|---------|---------|
| Zerobytes | O(1) | What is *this* position? | Terrain, biome, ore |
| Zero-Quadratic | O(N²) | What is the *relationship*? | Tension, trade, signal |
| Composed | O(N²) max | Position *given* relationships? | City prosperity |

```python
def city_prosperity(city_pos, all_city_positions, world_seed):
    cx, cy = city_pos
    terrain_value = coherent_value(cx*0.01, cy*0.01, world_seed)   # O(1)
    trade_income  = 0.0
    for other in all_city_positions:
        if other == city_pos: continue
        dist = ((other[0]-cx)**2 + (other[1]-cy)**2)**0.5
        if dist < 300:   # Bound N by proximity
            trade_income += trade_viability(city_pos, other, world_seed)
    return 0.5 * terrain_value + 0.5 * min(1.0, trade_income / 5.0)
```

## Anti-Patterns

```python
# BAD: Storing the graph — defeats the purpose
edges = {(a,b): compute(a,b) for a in nodes for b in nodes}

# BAD: Non-symmetric hash for symmetric relationship
def bad_bond(a, b): return hash(str(a) + str(b))  # bond(A,B) ≠ bond(B,A)

# BAD: Unbounded N
for a in get_all_world_entities():   # N = 100K → N² = 10 billion
    for b in get_all_world_entities(): ...

# GOOD: Query on demand, bounded N, symmetric hash
tension = faction_tension(player_city, target_city, world_seed)
```

## Debugging Checklist

When relationships differ across machines:
1. Check for `hash(tuple)` instead of `pair_hash` (platform-dependent)
2. Check for float accumulation in distance calc
3. Check sort stability in symmetric hash

When relationships "feel random with no structure":
- Add `coherent_value` layer from zerobytes to modulate pair hash output regionally
- Frequency 0.005–0.05 typical for faction-scale coherence

When N grows unbounded at runtime:
- Add proximity guard (`if dist > max_dist: return 0.0`)
- Hard-cap N in the caller, not inside the hash function

## Determinism Verification

```python
def verify_quadratic(rel_fn, seed, position_pairs):
    # Determinism
    for a, b in position_pairs:
        assert rel_fn(a, b, seed) == rel_fn(a, b, seed), f"Non-deterministic: {a},{b}"
    # Symmetry (for symmetric relations only)
    for a, b in position_pairs:
        assert rel_fn(a, b, seed) == rel_fn(b, a, seed), f"Asymmetric: {a},{b}"
    # Order independence
    fwd = {(a,b): rel_fn(a, b, seed) for a, b in position_pairs}
    rev = {(a,b): rel_fn(a, b, seed) for a, b in reversed(position_pairs)}
    assert all(fwd[k] == rev[k] for k in fwd), "Order-dependent!"
```

## Usage

When implementing a Zero-Quadratic system:

1. **Declare N** — hard-code max entities in pair space at design time
2. **Choose symmetry** — symmetric (`pair_hash`) or directional (`asymmetric_pair_hash`)?
3. **Compose with O(1)** — zerobytes for intrinsic properties, zero-quadratic for relational
4. **Design falloff** — linear, quadratic, or threshold? Encode in hash result, not post-process
5. **Inherit hierarchy** — should child relationships inherit parent tension via `hierarchical_pair_seed`?
6. **Verify** — run `verify_quadratic` with representative pairs before shipping

**Core principle:** The space between positions is as generative as the positions themselves. Zero-Quadratic stores no edges, no graphs, no histories. Infinite relational complexity. Zero bytes.
