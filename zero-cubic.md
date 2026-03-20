---
name: zero-cubic
description: Triple-is-seed O(N³) triadic procedural determinism methodology. Extends Zero-Quadratic from pairwise relationships to emergent properties of three-entity configurations. Use when a developer asks for coalition diplomacy, triangular trade circuits, three-faction tension, sacred geometry activation, group social dynamics, procedural crafting reactions, or any system where a property emerges from the configuration of exactly three entities and cannot be reduced to a sum of pairwise relationships. Triggers on phrases like "coalition stability", "three-faction", "triangular trade", "triadic relationship", "sacred triangle", "three-body problem", "group dynamics", "three-reagent crafting", "zero-cubic", or when a Zero-Quadratic system needs to model emergent properties that only exist when three entities are considered simultaneously.
---

# Zero-Cubic: Triple-is-Seed Triadic Procedural Determinism

Extends [Zero-Quadratic](zero-quadratic/SKILL.md) O(N²) pair hashing into O(N³) triple hashing. The coordinate *triple* IS the seed. Some properties only exist when three things meet — Zero-Cubic generates them without storing the configuration.

## The Conceptual Leap from Zero-Quadratic

Zero-Quadratic asks: *What exists between two positions?*
Zero-Cubic asks: *What emerges from the configuration of three positions?*

This is not reducible. `property(A,B,C) ≠ f(pair(A,B), pair(B,C), pair(A,C))` in the general case. Coalition stability, triangular trade viability, sacred geometry resonance — these are genuinely triadic: they emerge from the three-body configuration as a whole, not from a weighted sum of its edges. Zero-Cubic seeds this emergent property directly from the triple coordinate.

## The Five Extended Laws

Every triadic procedural system must satisfy:

1. **O(N³) Pair Access**: Any property of a three-entity configuration is computable directly from the triple coordinate — no graph traversal, no stored triangle
2. **Full Permutation Symmetry**: `prop(A,B,C) == prop(A,C,B) == prop(B,A,C) == prop(B,C,A) == prop(C,A,B) == prop(C,B,A)` unless intentionally ordered; enforced by sorting all three coordinates before hashing
3. **Proximity Budgeting**: Triple strength decays when any pair in the triple exceeds max distance — the proximity guard must apply to all three pair-distances, not just one
4. **Hierarchy Inheritance**: Child triple seeds derive from parent triple seeds plus local offsets — coalition tension bleeds down from empire level to city level to NPC level through seed inheritance
5. **Determinism**: Same coordinate triple → same emergent property value across all machines and execution orders

## Core Pattern

```python
import struct
import xxhash

def triple_hash(ax, ay, bx, by, cx, cy, salt=0):
    """Fully symmetric: all 6 permutations of (A,B,C) produce the same hash."""
    pts = sorted([(ax, ay), (bx, by), (cx, cy)])
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqqqqq', pts[0][0], pts[0][1],
                                    pts[1][0], pts[1][1],
                                    pts[2][0], pts[2][1]))
    return h.intdigest()

def asymmetric_triple_hash(ax, ay, bx, by, cx, cy, salt=0):
    """Ordered: prop(A,B,C) ≠ prop(A,C,B). For directed triadic flows."""
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqqqqq', ax, ay, bx, by, cx, cy))
    return h.intdigest()

def hash_to_float(h):
    return (h & 0xFFFFFFFF) / 0x100000000

def triple_proximity_guard(ax, ay, bx, by, cx, cy, max_dist):
    """Returns True if any pair in the triple exceeds max_dist."""
    def dist(x1, y1, x2, y2): return ((x2-x1)**2 + (y2-y1)**2) ** 0.5
    return (dist(ax, ay, bx, by) > max_dist or
            dist(bx, by, cx, cy) > max_dist or
            dist(ax, ay, cx, cy) > max_dist)

def triple_strength(ax, ay, bx, by, cx, cy, salt, max_dist=200.0):
    """Deterministic triadic property with proximity falloff. Returns 0.0–1.0."""
    if triple_proximity_guard(ax, ay, bx, by, cx, cy, max_dist):
        return 0.0
    base = hash_to_float(triple_hash(ax, ay, bx, by, cx, cy, salt))
    # Falloff: weakest pair distance drives the decay
    def dist(x1, y1, x2, y2): return ((x2-x1)**2 + (y2-y1)**2) ** 0.5
    d_ab = dist(ax, ay, bx, by)
    d_bc = dist(bx, by, cx, cy)
    d_ac = dist(ax, ay, cx, cy)
    weakest = max(d_ab, d_bc, d_ac)
    falloff = 1.0 - (weakest / max_dist) ** 2
    return base * falloff
```

## The Complexity Budget

O(N³) is a **budget declaration**, not an open cost. Declare N at design time and hard-cap it.

| Context | Typical N | Triples | Use Case |
|---------|-----------|---------|----------|
| Room-scale | 4–8 | 4–56 | Item reaction triads, NPC group dynamics |
| Chunk-scale | 8–16 | 56–560 | Faction coalition triangles, sacred sites |
| Region-scale | 16–32 | 560–4960 | Trade circuit viability, political alliances |
| World-scale | 32–64 | 4960–39984 | Empire coalition history, tectonic triple-junctions |

**Design rule: Never let N grow unbounded at runtime. Hard-code it. N=64 is near the ceiling for interactive use.**

## Quick Recipes

### Coalition Stability
```python
def coalition_stability(faction_a, faction_b, faction_c, world_seed):
    """Is this three-faction alliance stable? Not reducible to pairwise relations."""
    ax, ay = faction_a; bx, by = faction_b; cx, cy = faction_c

    # Emergent triadic property — the configuration as a whole
    base_stability = hash_to_float(triple_hash(ax, ay, bx, by, cx, cy, world_seed))

    # Each faction's individual disposition (ZeroBytes O(1) layer)
    militarism_a = hash_to_float(position_hash(ax, ay, 0, world_seed + 10))
    militarism_b = hash_to_float(position_hash(bx, by, 0, world_seed + 10))
    militarism_c = hash_to_float(position_hash(cx, cy, 0, world_seed + 10))
    avg_militarism = (militarism_a + militarism_b + militarism_c) / 3.0

    # High mutual militarism destabilises coalitions
    return base_stability * (1.0 - 0.4 * avg_militarism)
```

### Triangular Trade Circuit
```python
def trade_circuit_viability(port_a, port_b, port_c, world_seed):
    """Does this three-port circuit form a viable closed trade loop?"""
    ax, ay = port_a; bx, by = port_b; cx, cy = port_c

    # The circuit's emergent viability — a property of the triangle, not the edges
    circuit_seed = triple_hash(ax, ay, bx, by, cx, cy, world_seed + 20)
    circuit_viability = hash_to_float(circuit_seed)

    # Each leg's complementarity (Zero-Quadratic layer)
    leg_ab = hash_to_float(pair_hash(ax, ay, bx, by, world_seed + 21))
    leg_bc = hash_to_float(pair_hash(bx, by, cx, cy, world_seed + 21))
    leg_ca = hash_to_float(pair_hash(cx, cy, ax, ay, world_seed + 21))
    weakest_leg = min(leg_ab, leg_bc, leg_ca)

    # Circuit only as strong as its weakest leg, modulated by emergent factor
    return 0.5 * weakest_leg + 0.5 * circuit_viability
```

### Sacred Triangle Activation
```python
def sacred_triangle(site_a, site_b, site_c, world_seed, activation_threshold=0.72):
    """Does this three-site configuration form an active sacred triangle?"""
    ax, ay = site_a; bx, by = site_b; cx, cy = site_c

    resonance = triple_strength(ax, ay, bx, by, cx, cy, world_seed + 30, max_dist=500.0)
    if resonance < activation_threshold:
        return None

    # Secondary properties of the activated triangle
    t_seed = triple_hash(ax, ay, bx, by, cx, cy, world_seed + 30)
    return {
        "resonance":   resonance,
        "element":     ["fire","water","earth","air","void"][int(hash_to_float(t_seed) * 5)],
        "intensity":   hash_to_float(t_seed >> 16),
        "directional": hash_to_float(t_seed >> 32) > 0.5  # rotational asymmetry
    }
```

### NPC Group Dynamic
```python
def group_dynamic(npc_a_id, npc_b_id, npc_c_id, location_seed):
    """What social dynamic emerges when these three NPCs are together?"""
    # Abstract IDs: treat as (id, 0) coordinates
    stability = hash_to_float(triple_hash(npc_a_id, 0, npc_b_id, 0, npc_c_id, 0, location_seed))
    dominance_seed = asymmetric_triple_hash(npc_a_id, 0, npc_b_id, 0, npc_c_id, 0, location_seed + 1)
    dominant_index = int(hash_to_float(dominance_seed) * 3)  # which NPC leads

    if stability > 0.75:
        return {"type": "cohesive_group", "leader_index": dominant_index}
    elif stability < 0.25:
        return {"type": "volatile_triad", "scapegoat_index": dominant_index}
    else:
        return {"type": "shifting_alliances", "stability": stability}
```

### Procedural Crafting Reaction
```python
def crafting_reaction(reagent_a_id, reagent_b_id, reagent_c_id, world_seed):
    """Does this three-reagent combination produce a result? What result?"""
    r_seed = triple_hash(reagent_a_id, 0, reagent_b_id, 0, reagent_c_id, 0, world_seed + 50)
    reaction_strength = hash_to_float(r_seed)
    if reaction_strength < 0.6:
        return None  # No reaction
    product_id  = int(hash_to_float(r_seed >> 16) * 256)
    quality     = hash_to_float(r_seed >> 32)
    is_volatile = hash_to_float(r_seed >> 48) > 0.9  # dangerous reaction
    return {"product_id": product_id, "quality": quality, "volatile": is_volatile}
```

## Hierarchy Pattern (Extended from Zero-Quadratic)

Zero-Quadratic hierarchy: `parent_pair_seed → child_pair_seed via local pair`
Zero-Cubic adds: `parent_triple_seed → child_triple_seed via local triple`

```python
def hierarchical_triple_seed(parent_triple_seed, local_a, local_b, local_c):
    """Child triple relationship inherits parent triple context."""
    return triple_hash(
        local_a, parent_triple_seed & 0xFFFF,
        local_b, (parent_triple_seed >> 16) & 0xFFFF,
        local_c, (parent_triple_seed >> 32) & 0xFFFF,
        salt=0
    )
```

**Practical consequence:** Three continents in an unstable triple generate statistically more volatile empire-level triangles, which generate more hostile city-level triads, which generate more volatile NPC group dynamics — through seed inheritance, not stored data.

### Full Hierarchy Example
```
World triple     (continent_A ↔ continent_B ↔ continent_C)
  └─ Region triple  (region_A ↔ region_B ↔ region_C)    ← inherits continent instability
       └─ City triple  (city_A ↔ city_B ↔ city_C)       ← inherits regional instability
            └─ NPC triple  (npc_A ↔ npc_B ↔ npc_C)      ← inherits city group dynamic
```

## Composing with ZeroBytes and Zero-Quadratic

| Layer | Complexity | Answers | Example |
|-------|-----------|---------|---------|
| ZeroBytes | O(1) | What is *this* entity? | Faction type, port resources |
| Zero-Quadratic | O(N²) | What is the *pairwise* relationship? | Trade viability, hostility |
| Zero-Cubic | O(N³) | What *emerges* from three together? | Coalition stability, trade circuit |
| Composed | O(N³) max | Full diplomatic picture | City prosperity in a three-empire world |

```python
def diplomatic_situation(city_pos, all_faction_positions, world_seed):
    cx, cy = city_pos
    terrain = coherent_value(cx*0.01, cy*0.01, world_seed)          # O(1) ZeroBytes

    pairwise_tensions = []
    for other in all_faction_positions:
        if other == city_pos: continue
        pairwise_tensions.append(
            faction_tension(city_pos, other, world_seed)              # O(N²) Zero-Quadratic
        )

    triadic_stabilities = []
    factions = all_faction_positions[:16]                             # Hard-cap N
    for i, a in enumerate(factions):
        for j, b in enumerate(factions[i+1:], i+1):
            for k, c in enumerate(factions[j+1:], j+1):
                triadic_stabilities.append(
                    coalition_stability(a, b, c, world_seed)          # O(N³) Zero-Cubic
                )

    avg_tension   = sum(pairwise_tensions) / max(len(pairwise_tensions), 1)
    avg_stability = sum(triadic_stabilities) / max(len(triadic_stabilities), 1)
    return {
        "terrain_value":       terrain,
        "pairwise_tension":    avg_tension,
        "coalition_stability": avg_stability,
        "strategic_value":     terrain * (1 - avg_tension) * avg_stability
    }
```

## Anti-Patterns

```python
# BAD: Reducing triadic property to pairwise sum — loses emergent information
def bad_coalition(a, b, c): return (tension(a,b) + tension(b,c) + tension(a,c)) / 3

# BAD: Asymmetric hash for symmetric property
def bad_triple(a, b, c): return hash(str(a) + str(b) + str(c))  # order-dependent

# BAD: Unbounded N
for a in all_factions:            # N = 1000 → N³ = 1 billion
    for b in all_factions:
        for c in all_factions: ...

# BAD: Applying proximity guard to only one pair
def bad_guard(a, b, c, max_d):
    if dist(a, b) > max_d: return 0.0   # misses b-c and a-c distances

# GOOD: Sorted triple, hard-capped N, all-pairs proximity guard
stability = coalition_stability(faction_a, faction_b, faction_c, world_seed)
```

## Debugging Checklist

When triple properties differ across machines:
1. Check for `hash(tuple)` instead of `triple_hash` (platform-dependent)
2. Check sort stability — Python's `sorted()` on tuples is stable and correct
3. Check struct pack format — `'<qqqqqq'` (little-endian signed 64-bit) must match across platforms

When triples feel "too chaotic with no regional structure":
- Add `coherent_value` modulation from ZeroBytes at the region scale
- The triple hash gives the local emergent value; the coherent layer gives it regional context
- Frequency 0.003–0.03 typical for faction-scale coherence

When N grows unbounded at runtime:
- Hard-cap in the caller: `factions = all_factions[:32]`
- Apply proximity guard before triple_hash, not inside it
- Consider chunk-partitioning: only compute triples within the same region chunk

## Determinism Verification

```python
def verify_cubic(triple_fn, seed, triples):
    """Verify determinism, full permutation symmetry, and order independence."""
    import itertools

    # Determinism
    for a, b, c in triples:
        v1 = triple_fn(a, b, c, seed)
        v2 = triple_fn(a, b, c, seed)
        assert v1 == v2, f"Non-deterministic: {a},{b},{c}"

    # Full permutation symmetry (6 permutations)
    for a, b, c in triples:
        ref = triple_fn(a, b, c, seed)
        for perm in itertools.permutations([a, b, c]):
            assert triple_fn(*perm, seed) == ref, f"Asymmetric permutation: {perm}"

    # Order independence across query sequence
    fwd = {(a,b,c): triple_fn(a, b, c, seed) for a, b, c in triples}
    rev = {(a,b,c): triple_fn(a, b, c, seed) for a, b, c in reversed(triples)}
    assert all(fwd[k] == rev[k] for k in fwd), "Order-dependent!"

    print(f"Zero-Cubic verification passed: {len(triples)} triples, 6 permutations each.")
```

## Usage

When implementing a Zero-Cubic system:

1. **Confirm the property is genuinely triadic** — can it be reduced to pairwise? If yes, use Zero-Quadratic instead
2. **Declare N** — hard-code maximum entities in triple space at design time; apply proximity guards
3. **Choose symmetry** — fully symmetric (`triple_hash`) or directed (`asymmetric_triple_hash`)?
4. **Compose the layers** — ZeroBytes for intrinsic entity properties, Zero-Quadratic for pairwise edges, Zero-Cubic for emergent triangle properties
5. **Design the proximity guard** — apply to all three pair-distances; the weakest pair drives decay
6. **Inherit hierarchy** — should child triples inherit parent triple tension via `hierarchical_triple_seed`?
7. **Verify** — run `verify_cubic` with representative triples before shipping

**Core principle:** Some properties only exist when three things meet. The triangle has a soul the edges alone cannot explain. Zero-Cubic generates that soul without storing the triangle.
