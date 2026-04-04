---
name: zero-cubic
description: Triple-is-seed O(N³) triadic procedural determinism methodology. Extends Zero-Quadratic from pairwise relationships to emergent properties of three-entity configurations. Use when a developer asks for coalition diplomacy, triangular trade circuits, three-faction tension, sacred geometry activation, group social dynamics, procedural crafting reactions, or any system where a property emerges from the configuration of exactly three entities and cannot be reduced to a sum of pairwise relationships. Triggers on phrases like "coalition stability", "three-faction", "triangular trade", "triadic relationship", "sacred triangle", "three-body problem", "group dynamics", "three-reagent crafting", "zero-cubic", or when a Zero-Quadratic system needs to model emergent properties that only exist when three entities are considered simultaneously.
---

# Zero-Cubic: Triple-is-Seed Triadic Procedural Determinism

Extends [Zero-Quadratic](../zero-quadratic/SKILL.md) O(N²) pair hashing into O(N³) triple hashing. The coordinate *triple* IS the seed. Some properties only exist when three things meet — Zero-Cubic generates them without storing the configuration.

## The Five Extended Laws

Every triadic procedural system must satisfy:

1. **O(N³) Triple Access**: Any property of a three-entity configuration is computable directly from the triple coordinate — no graph traversal, no stored triangle
2. **Full Permutation Symmetry**: `prop(A,B,C) == prop(B,A,C) == prop(C,B,A)` (all 6 permutations) unless intentionally ordered; enforced by sorting all three coordinates before hashing
3. **Proximity Budgeting**: Proximity guard must apply to all three pair-distances — the weakest pair drives decay
4. **Hierarchy Inheritance**: Child triple seeds derive from parent triple seeds plus local offsets — coalition tension bleeds down through scales
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
    def d(x1, y1, x2, y2): return ((x2-x1)**2 + (y2-y1)**2) ** 0.5
    return d(ax,ay,bx,by) > max_dist or d(bx,by,cx,cy) > max_dist or d(ax,ay,cx,cy) > max_dist

def triple_strength(ax, ay, bx, by, cx, cy, salt, max_dist=200.0):
    """Deterministic triadic intensity with proximity falloff. Returns 0.0–1.0."""
    if triple_proximity_guard(ax, ay, bx, by, cx, cy, max_dist):
        return 0.0
    def d(x1, y1, x2, y2): return ((x2-x1)**2 + (y2-y1)**2) ** 0.5
    weakest = max(d(ax,ay,bx,by), d(bx,by,cx,cy), d(ax,ay,cx,cy))
    falloff = 1.0 - (weakest / max_dist) ** 2
    return hash_to_float(triple_hash(ax, ay, bx, by, cx, cy, salt)) * falloff
```

## Complexity Budget

O(N³) is a **budget declaration**, not an open cost. Declare N at design time.

| Context | Typical N | Triples | Use Case |
|---------|-----------|---------|----------|
| Room-scale | 4–8 | 4–56 | Item reaction triads, NPC group dynamics |
| Chunk-scale | 8–16 | 56–560 | Faction coalition triangles, sacred sites |
| Region-scale | 16–32 | 560–4960 | Trade circuit viability, political alliances |
| World-scale | 32–64 | 4960–39984 | Empire coalitions, tectonic triple-junctions |

**Design rule: Never let N grow unbounded at runtime. N=64 is the ceiling for interactive use.**

## Quick Recipes

### Coalition Stability
```python
def coalition_stability(faction_a, faction_b, faction_c, world_seed):
    """Is this three-faction alliance stable? Not reducible to pairwise relations."""
    ax, ay = faction_a; bx, by = faction_b; cx, cy = faction_c
    base = hash_to_float(triple_hash(ax, ay, bx, by, cx, cy, world_seed))
    militarism_a = hash_to_float(position_hash(ax, ay, 0, world_seed + 10))
    militarism_b = hash_to_float(position_hash(bx, by, 0, world_seed + 10))
    militarism_c = hash_to_float(position_hash(cx, cy, 0, world_seed + 10))
    return base * (1.0 - 0.4 * ((militarism_a + militarism_b + militarism_c) / 3.0))
```

### Triangular Trade Circuit
```python
def trade_circuit_viability(port_a, port_b, port_c, world_seed):
    """Does this three-port circuit form a viable closed trade loop?"""
    ax, ay = port_a; bx, by = port_b; cx, cy = port_c
    circuit_viability = hash_to_float(triple_hash(ax, ay, bx, by, cx, cy, world_seed + 20))
    leg_ab = hash_to_float(pair_hash(ax, ay, bx, by, world_seed + 21))
    leg_bc = hash_to_float(pair_hash(bx, by, cx, cy, world_seed + 21))
    leg_ca = hash_to_float(pair_hash(cx, cy, ax, ay, world_seed + 21))
    return 0.5 * min(leg_ab, leg_bc, leg_ca) + 0.5 * circuit_viability
```

### Sacred Triangle Activation
```python
def sacred_triangle(site_a, site_b, site_c, world_seed, threshold=0.72):
    """Does this three-site configuration form an active sacred triangle?"""
    ax, ay = site_a; bx, by = site_b; cx, cy = site_c
    resonance = triple_strength(ax, ay, bx, by, cx, cy, world_seed + 30, max_dist=500.0)
    if resonance < threshold:
        return None
    t = triple_hash(ax, ay, bx, by, cx, cy, world_seed + 30)
    return {
        "resonance":   resonance,
        "element":     ["fire","water","earth","air","void"][int(hash_to_float(t) * 5)],
        "intensity":   hash_to_float(t >> 16),
        "directional": hash_to_float(t >> 32) > 0.5
    }
```

### NPC Group Dynamic
```python
def group_dynamic(npc_a_id, npc_b_id, npc_c_id, location_seed):
    """What social dynamic emerges when these three NPCs are together?"""
    stability      = hash_to_float(triple_hash(npc_a_id, 0, npc_b_id, 0, npc_c_id, 0, location_seed))
    dominant_index = int(hash_to_float(asymmetric_triple_hash(npc_a_id, 0, npc_b_id, 0, npc_c_id, 0, location_seed + 1)) * 3)
    if stability > 0.75:   return {"type": "cohesive_group",     "leader_index":    dominant_index}
    if stability < 0.25:   return {"type": "volatile_triad",     "scapegoat_index": dominant_index}
    return                        {"type": "shifting_alliances",  "stability":       stability}
```

### Procedural Crafting Reaction
```python
def crafting_reaction(reagent_a_id, reagent_b_id, reagent_c_id, world_seed):
    """Does this three-reagent combination produce a result? What result?"""
    r = triple_hash(reagent_a_id, 0, reagent_b_id, 0, reagent_c_id, 0, world_seed + 50)
    if hash_to_float(r) < 0.6:
        return None
    return {
        "product_id": int(hash_to_float(r >> 16) * 256),
        "quality":    hash_to_float(r >> 32),
        "volatile":   hash_to_float(r >> 48) > 0.9
    }
```

## Hierarchy Pattern

```python
def hierarchical_triple_seed(parent_triple_seed, local_a, local_b, local_c):
    """Child triple inherits parent triple context."""
    return triple_hash(
        local_a, parent_triple_seed & 0xFFFF,
        local_b, (parent_triple_seed >> 16) & 0xFFFF,
        local_c, (parent_triple_seed >> 32) & 0xFFFF,
        salt=0
    )
```

```
World triple   (continent_A ↔ continent_B ↔ continent_C)
  └─ Region triple  (region_A ↔ region_B ↔ region_C)   ← inherits continent instability
       └─ City triple  (city_A ↔ city_B ↔ city_C)      ← inherits regional instability
            └─ NPC triple  (npc_A ↔ npc_B ↔ npc_C)     ← inherits city group dynamic
```

## Composing with ZeroBytes and Zero-Quadratic

| Layer | Complexity | Answers | Example |
|-------|-----------|---------|---------|
| ZeroBytes | O(1) | What is *this* entity? | Faction type, port resources |
| Zero-Quadratic | O(N²) | What is the *pairwise* relationship? | Trade viability, hostility |
| Zero-Cubic | O(N³) | What *emerges* from three together? | Coalition stability, trade circuit |

## Anti-Patterns

```python
# BAD: Reducing triadic property to pairwise sum — loses emergent information
def bad_coalition(a, b, c): return (tension(a,b) + tension(b,c) + tension(a,c)) / 3

# BAD: Platform hash — order-dependent
def bad_triple(a, b, c): return hash(str(a) + str(b) + str(c))

# BAD: Unbounded N
for a in all_factions:      # N=1000 → N³ = 1 billion
    for b in all_factions:
        for c in all_factions: ...

# BAD: Proximity guard on only one pair
def bad_guard(a, b, c, max_d):
    if dist(a, b) > max_d: return 0.0   # misses b-c and a-c

# GOOD: Sorted triple, hard-capped N, all-pairs proximity guard
stability = coalition_stability(faction_a, faction_b, faction_c, world_seed)
```

## Debugging Checklist

When triple properties differ across machines:
1. Check for `hash(tuple)` instead of `triple_hash` — platform-dependent
2. Check sort stability — Python `sorted()` on tuples is stable and correct
3. Check struct format — `'<qqqqqq'` (little-endian signed 64-bit) must match across platforms

When triples feel "too chaotic with no regional structure":
- Add `coherent_value` modulation at region scale; frequency 0.003–0.03 typical
- Triple hash gives local emergent value; coherent layer gives regional context

When N grows unbounded:
- Hard-cap in the caller: `factions = all_factions[:32]`
- Apply proximity guard before triple_hash, not inside it

## Determinism Verification

```python
def verify_cubic(triple_fn, seed, triples):
    import itertools
    for a, b, c in triples:
        assert triple_fn(a,b,c,seed) == triple_fn(a,b,c,seed), f"Non-deterministic: {a},{b},{c}"
    for a, b, c in triples:
        ref = triple_fn(a, b, c, seed)
        for perm in itertools.permutations([a, b, c]):
            assert triple_fn(*perm, seed) == ref, f"Asymmetric permutation: {perm}"
    fwd = {(a,b,c): triple_fn(a,b,c,seed) for a,b,c in triples}
    rev = {(a,b,c): triple_fn(a,b,c,seed) for a,b,c in reversed(triples)}
    assert all(fwd[k] == rev[k] for k in fwd), "Order-dependent!"
```

## Usage

1. **Confirm the property is genuinely triadic** — reducible to pairwise? Use Zero-Quadratic instead
2. **Declare N** — hard-code maximum entities at design time; apply proximity guards
3. **Choose symmetry** — symmetric (`triple_hash`) or directed (`asymmetric_triple_hash`)?
4. **Compose layers** — ZeroBytes for entity properties, Zero-Quadratic for edges, Zero-Cubic for emergent triangles
5. **All-pairs proximity guard** — weakest pair drives decay; guard all three distances
6. **Inherit hierarchy** — child triples inherit parent context via `hierarchical_triple_seed`
7. **Verify** — run `verify_cubic` with representative triples before shipping

**Core principle:** Some properties only exist when three things meet. The triangle has a soul the edges alone cannot explain. Zero-Cubic generates that soul without storing the triangle.
