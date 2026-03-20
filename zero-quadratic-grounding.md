# Zero-Quadratic Grounding: O(N²) Relational Procedural Determinism

> **Extends:** [Zerobytes](zerobytes/SKILL.md) — Position-is-Seed O(1) determinism  
> **New primitive:** The *pair* is the seed. Relationships between positions generate emergent structure that no single coordinate can encode alone.

---

## The Core Insight

Zerobytes answers: *"What is this position?"*  
Zero-Quadratic answers: *"What is the relationship between these two positions?"*

O(1) gives you deterministic points.  
O(N²) gives you deterministic **fields of influence** — gravity, faction tension, trade routes, signal propagation, narrative causality — all derivable without storage, without iteration, from pure coordinate-pair hashing.

```
zerobytes:         hash(A)          → value at A
zero-quadratic:    hash(A, B)       → relationship between A and B
                   hash(A, B, A+B)  → emergent property of their interaction
```

The universe springs not just from points, but from the space *between* points.

---

## The Five Extended Laws

Extending the Zerobytes Five Laws for relational systems:

1. **O(N²) Pair Access**: Any relationship between two positions is computable directly from their coordinates — no graph traversal, no stored adjacency
2. **Symmetric Coherence**: `rel(A, B) == rel(B, A)` unless intentionally asymmetric; hash packing must enforce this
3. **Locality Falloff**: Relationship strength decays deterministically with distance — encoded in the hash result, not post-processed
4. **Hierarchy Inheritance**: Child pair seeds derive from parent pair seeds plus local offset pair
5. **Determinism**: Same coordinate pair → same relationship value across all machines, all execution orders

---

## Core Pattern

```python
import struct
import xxhash

def pair_hash(ax: int, ay: int, bx: int, by: int, salt: int = 0) -> int:
    """
    Symmetric pair hash: pair_hash(A, B) == pair_hash(B, A)
    Achieved by sorting coordinates before packing.
    """
    # Sort pairs so (A,B) == (B,A) deterministically
    p1, p2 = sorted([(ax, ay), (bx, by)])
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqqq', p1[0], p1[1], p2[0], p2[1]))
    return h.intdigest()

def asymmetric_pair_hash(ax: int, ay: int, bx: int, by: int, salt: int = 0) -> int:
    """
    Directional pair hash: rel(A→B) ≠ rel(B→A)
    For rivers, trade winds, faction aggression.
    """
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqqq', ax, ay, bx, by))
    return h.intdigest()

def hash_to_float(h: int) -> float:
    return (h & 0xFFFFFFFF) / 0x100000000

def relationship_strength(ax, ay, bx, by, salt, max_dist=100.0) -> float:
    """
    Deterministic relationship intensity with distance falloff.
    Returns 0.0 (no relation) to 1.0 (maximum relation).
    """
    dist = ((bx - ax)**2 + (by - ay)**2) ** 0.5
    if dist > max_dist:
        return 0.0
    base = hash_to_float(pair_hash(ax, ay, bx, by, salt))
    falloff = 1.0 - (dist / max_dist) ** 2  # quadratic falloff
    return base * falloff
```

---

## The Complexity Budget

O(N²) is not a cost — it is a **budget declaration**. You are asserting:  
*"For N notable positions in this region, we will evaluate N² relationships."*

The key discipline: **N must be bounded by design**, not left open-ended.

| Context | Typical N | Pairs Evaluated | Use Case |
|---------|-----------|-----------------|----------|
| Room-scale | 8–16 | 64–256 | Item interactions, trap logic, NPC social graph |
| Chunk-scale | 32–64 | 1K–4K | Faction influence, resource veins, ley lines |
| Region-scale | 128–256 | 16K–65K | Trade routes, biome pressure, climate cells |
| World-scale | 512–1024 | 256K–1M | Tectonic plates, empire history, stellar neighbourhoods |

Design rule: **Declare N at system design time. Never let N grow unbounded at runtime.**

---

## Quadratic Recipes

### Faction Tension Map

```python
def faction_tension(city_a_pos, city_b_pos, world_seed):
    """
    Deterministic political tension between two cities.
    No state. No history. Derived entirely from position pair.
    """
    ax, ay = city_a_pos
    bx, by = city_b_pos
    
    # Base hostility from pair hash (symmetric — they dislike each other equally)
    hostility = hash_to_float(pair_hash(ax, ay, bx, by, world_seed + 1))
    
    # Cultural distance as coherent noise divergence
    culture_a = coherent_value(ax * 0.01, ay * 0.01, world_seed + 2, octaves=3)
    culture_b = coherent_value(bx * 0.01, by * 0.01, world_seed + 2, octaves=3)
    cultural_gap = abs(culture_a - culture_b)
    
    # Geographic pressure (mountains, deserts raise tension)
    border_roughness = hash_to_float(pair_hash(ax, ay, bx, by, world_seed + 3))
    
    return 0.4 * hostility + 0.4 * cultural_gap + 0.2 * border_roughness
```

### Trade Route Viability

```python
def trade_viability(port_a, port_b, world_seed):
    """
    Directional trade strength: A exports TO B.
    Asymmetric — spice flows east, timber flows west.
    """
    ax, ay = port_a
    bx, by = port_b
    
    # Directional resource asymmetry
    resource_a = hash_to_float(asymmetric_pair_hash(ax, ay, bx, by, world_seed + 10))
    resource_b = hash_to_float(asymmetric_pair_hash(bx, by, ax, ay, world_seed + 10))
    
    # Trade only flows if A has what B lacks
    complementarity = max(0.0, resource_a - resource_b)
    
    # Distance cost (quadratic falloff from pair hash)
    dist = ((bx - ax)**2 + (by - ay)**2) ** 0.5
    distance_penalty = min(1.0, dist / 500.0) ** 2
    
    return complementarity * (1.0 - distance_penalty)
```

### NPC Social Graph

```python
def npc_relationship(npc_a_id, npc_b_id, location_seed):
    """
    Procedural relationship between two NPCs in a location.
    Friend, rival, indifferent — fully deterministic from IDs.
    """
    # Use IDs directly as coordinates in abstract space
    bond = hash_to_float(pair_hash(npc_a_id, 0, npc_b_id, 0, location_seed))
    rivalry = hash_to_float(pair_hash(npc_a_id, 1, npc_b_id, 1, location_seed))
    history = hash_to_float(pair_hash(npc_a_id, 2, npc_b_id, 2, location_seed))
    
    if bond > 0.7:   return {"type": "allies",  "strength": bond}
    if rivalry > 0.7: return {"type": "rivals",  "strength": rivalry}
    return           {"type": "neutral", "history_depth": history}
```

### Ley Line Network

```python
def ley_connection(node_a, node_b, world_seed, threshold=0.65):
    """
    Magical energy conduit between two nodes.
    Connection exists only if pair hash exceeds threshold.
    Creates sparse, deterministic graph without storing edges.
    """
    ax, ay = node_a
    bx, by = node_b
    
    strength = hash_to_float(pair_hash(ax, ay, bx, by, world_seed))
    dist = ((bx - ax)**2 + (by - ay)**2) ** 0.5
    
    if strength < threshold:
        return None  # No connection
    
    return {
        "exists": True,
        "strength": strength,
        "frequency": hash_to_float(pair_hash(ax, ay, bx, by, world_seed + 99)),
        "bidirectional": hash_to_float(pair_hash(ax, ay, bx, by, world_seed + 7)) > 0.5
    }
```

---

## Hierarchy Pattern (Extended)

The zerobytes hierarchy flows from parent seed → child seed via position.  
Zero-Quadratic adds a second hierarchy axis: **the relationship tree**.

```
World pair (continent_A, continent_B)
  └─ Region pair (region_A, region_B) derived from continent relationship
       └─ City pair (city_A, city_B) derived from regional relationship
            └─ NPC pair (npc_A, npc_B) derived from city relationship
```

```python
def hierarchical_pair_seed(parent_pair_seed, local_a, local_b):
    """
    Child relationship seed inherits from parent relationship context.
    Ensures continent-scale hostility bleeds into city-scale tension.
    """
    return pair_hash(local_a, parent_pair_seed, local_b, parent_pair_seed >> 32, 0)
```

This means: cities in feuding continents will *statistically* generate more hostile NPC pairs — not through explicit data, but through seed inheritance.

---

## Anti-Patterns

```python
# BAD: Storing the relationship graph
edges = {}
for a in nodes:
    for b in nodes:
        edges[(a,b)] = compute_relation(a, b)  # Defeats the purpose

# BAD: Non-symmetric hash for symmetric relationship
def bad_bond(a, b):
    return hash(str(a) + str(b))  # bond(A,B) ≠ bond(B,A)

# BAD: Unbounded N
all_entities = get_all_entities_in_world()  # N = 100,000, N² = 10 billion
for a in all_entities:
    for b in all_entities: ...

# GOOD: Bounded N, direct computation, symmetric hash
def good_bond(a_pos, b_pos, seed):
    return hash_to_float(pair_hash(*a_pos, *b_pos, seed))

# GOOD: Query on demand — compute only the pair you need
tension = faction_tension(player_city, target_city, world_seed)
```

---

## Composing O(1) and O(N²)

Zerobytes and Zero-Quadratic are **complementary layers**, not competing approaches.

| Layer | Complexity | Answers | Examples |
|-------|-----------|---------|---------|
| Zerobytes | O(1) | What is *this* position? | Terrain height, biome, ore density |
| Zero-Quadratic | O(N²) | What is the *relationship* between positions? | Trade routes, political tension, signal paths |
| Composition | O(N²) max | What is this position *given its relationships*? | City prosperity shaped by trade + terrain |

```python
def city_prosperity(city_pos, all_city_positions, world_seed):
    """
    Composed: O(1) terrain base + O(N) trade relationships (N = nearby cities)
    """
    cx, cy = city_pos
    
    # O(1): intrinsic geographic value
    terrain_value = coherent_value(cx * 0.01, cy * 0.01, world_seed)
    
    # O(N): sum trade viability with nearby cities (N bounded by proximity)
    trade_income = 0.0
    for other_pos in all_city_positions:
        if other_pos == city_pos:
            continue
        dist = ((other_pos[0]-cx)**2 + (other_pos[1]-cy)**2)**0.5
        if dist < 300:  # Only evaluate nearby pairs — bounding N
            trade_income += trade_viability(city_pos, other_pos, world_seed)
    
    return 0.5 * terrain_value + 0.5 * min(1.0, trade_income / 5.0)
```

---

## Verification

```python
def verify_quadratic(rel_fn, seed, position_pairs):
    """
    Verify O(N²) system satisfies symmetry and determinism.
    """
    # Determinism: same pair → same result
    for a, b in position_pairs:
        r1 = rel_fn(a, b, seed)
        r2 = rel_fn(a, b, seed)
        assert r1 == r2, f"Non-deterministic at {a},{b}"
    
    # Symmetry: rel(A,B) == rel(B,A) for symmetric relations
    for a, b in position_pairs:
        assert rel_fn(a, b, seed) == rel_fn(b, a, seed), f"Asymmetric at {a},{b}"
    
    # Order independence: evaluating pairs in any order gives same results
    fwd  = {(a,b): rel_fn(a, b, seed) for a, b in position_pairs}
    rev  = {(a,b): rel_fn(a, b, seed) for a, b in reversed(position_pairs)}
    assert all(fwd[k] == rev[k] for k in fwd), "Order-dependent!"
```

---

## Design Checklist

When implementing a Zero-Quadratic system:

1. **Declare N** — what is the maximum number of entities in the pair space? Hard-code this at design time
2. **Choose symmetry** — is this relationship symmetric (bond, distance) or directional (aggression, trade flow)?
3. **Compose with O(1)** — use zerobytes for intrinsic properties, zero-quadratic for relational properties
4. **Design falloff** — how does relationship strength decay with distance? Linear, quadratic, or threshold?
5. **Inherit hierarchy** — should child-level relationships inherit tension from parent-level relationships?
6. **Verify** — run `verify_quadratic` with representative pairs before shipping

---

## Core Principle

> **The space between positions is as generative as the positions themselves.**  
> Zero-Quadratic stores no edges, no graphs, no histories.  
> Every relationship in the universe springs complete from coordinate pairs.  
> Infinite relational complexity. Zero bytes.
