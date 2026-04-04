---
name: zero-causal
description: Chain-is-seed O(1) causal chain procedural determinism methodology. Extends ZeroBytes from static spatial generation into deterministic causal histories and futures. Use when a developer asks for procedural civilisation history, genetic lineage generation, criminal causal chain reconstruction, technology tree evolution, narrative event chains, disease outbreak propagation history, geological event sequences, or any system where a chain of causally-linked events must be reproducible at any point in the chain without stepping through all preceding events. Triggers on phrases like "procedural history", "causal chain", "event sequence", "direct jump to epoch", "lineage generation", "chain-is-seed", "zero-causal", "deterministic timeline", "event at depth N", "without replaying", or when a Zero-Temporal system needs not just temporal properties but a narrative chain of discrete causal events that can be queried at any depth in O(1).
---

# Zero-Causal: Chain-is-Seed Deterministic Causal Event Generation

Extends [ZeroBytes](../zerobytes/SKILL.md) and [Zero-Temporal](../zero-temporal/SKILL.md) from coordinate-addressed properties into deterministic causal chains. The chain origin + depth IS the seed. Any event at any depth in any causal history is directly computable in O(1) without replaying preceding events.

**The critical distinction from Zero-Temporal:** Zero-Temporal generates continuous properties (weather, season). Zero-Causal generates discrete narrative events (battle, founding, betrayal) with internal causal logic — without requiring sequential computation to reach any depth.

## The Five Extended Laws

1. **O(1) Depth Access**: Event N is directly computable from `(origin_seed, N)` — depth IS the coordinate; never iterate 0→N to reach N
2. **Causal Plausibility**: Event N is statistically influenced by the *type index* of event N-1 without requiring N-1's full computation — encode transition weights keyed by type index
3. **Chain Individuality**: Different spatial origins produce statistically distinct chains — a maritime city's history differs from a mountain city's via origin seed, not runtime state
4. **Branching Determinism**: Both sides of a chain fork are independently seedable from the same origin + depth via branch salt
5. **Determinism**: Same origin seed + same depth → same event across all machines and execution orders

## Core Pattern

```python
import struct
import xxhash

def position_hash(x, y, z, salt=0):
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqq', x, y, z))
    return h.intdigest()

def hash_to_float(h):
    return (h & 0xFFFFFFFF) / 0x100000000

def chain_origin_seed(origin_x, origin_y, world_seed, chain_type_salt=0):
    """Root seed for a causal chain. Use chain_type_salt to distinguish chain types
    (political=100, genetic=200, criminal=300) at the same spatial origin."""
    return position_hash(origin_x, origin_y, chain_type_salt, world_seed)

def event_at_depth(origin_seed, depth, salt=0):
    """O(1) access to any event. depth must be a Python int, never a float."""
    return position_hash(origin_seed, depth, 0, salt)

def event_float(origin_seed, depth, channel=0):
    return hash_to_float(event_at_depth(origin_seed, depth, salt=channel))

def event_int(origin_seed, depth, modulus, channel=0):
    return int(event_float(origin_seed, depth, channel) * modulus)

def biased_event_type(origin_seed, depth, event_types, transition_bias):
    """
    Select event type biased by the type of the previous event.
    Causal plausibility without sequential computation — only the previous
    event's type index (cheaply computable) is needed, not its full value.
    transition_bias: {prev_type_index: [weight_per_next_type, ...]}
    """
    prev_type = 0 if depth == 0 else event_int(origin_seed, depth-1, len(event_types))
    weights   = list(transition_bias.get(prev_type, [1.0] * len(event_types)))
    raw       = event_float(origin_seed, depth)
    total     = sum(weights); cumulative = 0.0
    for i, w in enumerate(weights):
        cumulative += w / total
        if raw < cumulative: return i
    return len(event_types) - 1
```

## Quick Recipes

### Civilisation History
```python
POLITICAL_EVENTS = ["founding","expansion","civil_war","golden_age",
                    "plague","invasion","reform","collapse","revival"]

POLITICAL_TRANSITIONS = {
    0: [1.0, 2.0, 0.5, 1.0, 0.5, 0.5, 1.0, 0.2, 0.5],  # founding → expansion likely
    2: [0.5, 0.5, 0.3, 0.5, 1.5, 2.0, 1.0, 1.5, 0.5],  # civil_war → invasion/collapse
    3: [0.5, 1.5, 0.3, 0.5, 0.8, 0.8, 1.5, 0.2, 0.5],  # golden_age → expansion/reform
    7: [0.5, 0.5, 0.5, 0.5, 0.5, 0.5, 1.0, 0.3, 2.5],  # collapse → revival likely
}

def civilisation_event(city_x, city_y, world_seed, depth):
    """O(1) access to any moment in this city's history. No replay."""
    origin  = chain_origin_seed(city_x, city_y, world_seed, chain_type_salt=100)
    ev_type = biased_event_type(origin, depth, POLITICAL_EVENTS, POLITICAL_TRANSITIONS)
    return {
        "depth":          depth,
        "event":          POLITICAL_EVENTS[ev_type],
        "magnitude":      event_float(origin, depth, channel=1),
        "duration":       1 + event_int(origin, depth, 20, channel=2),
        "lasting_effect": event_float(origin, depth, channel=1) > 0.75
    }

def city_history(city_x, city_y, world_seed, from_depth, to_depth):
    return [civilisation_event(city_x, city_y, world_seed, d) for d in range(from_depth, to_depth)]
```

### Genetic Lineage
```python
MUTATION_TYPES = ["beneficial_metabolism","neutral_coloration","harmful_immunity",
                  "beneficial_size","neutral_behaviour","beneficial_cognition",
                  "harmful_reproduction","neutral_lifespan"]

def genetic_generation(species_x, species_y, world_seed, generation):
    """O(1) access to any generation. No need to simulate all preceding ones."""
    origin = chain_origin_seed(species_x, species_y, world_seed, chain_type_salt=200)
    mut    = event_int(origin, generation, len(MUTATION_TYPES))
    return {
        "generation":    generation,
        "mutation":      MUTATION_TYPES[mut],
        "fitness_delta": (event_float(origin, generation, channel=1) - 0.5) * 2,
        "is_dominant":   event_float(origin, generation, channel=2) > 0.6,
        "causes_split":  abs((event_float(origin, generation, channel=1) - 0.5) * 2) > 0.8
    }
```

### Criminal Causal Chain
```python
CRIME_EVENTS = ["initial_grievance","planning","recruitment","resource_acquisition",
                "first_offence","escalation","detection","evasion",
                "accomplice_betrayal","confrontation","capture","escape"]

def crime_chain_event(crime_x, crime_y, crime_epoch, world_seed, step):
    """Reconstruct any step of a crime chain — useful for investigation gameplay."""
    from zero_temporal import temporal_hash   # compose with Zero-Temporal
    temporal_s  = temporal_hash(crime_x, crime_y, 0, crime_epoch, world_seed)
    origin      = position_hash(temporal_s, 0, 0, 500)
    ev          = event_int(origin, step, len(CRIME_EVENTS))
    left_evidence = event_float(origin, step, channel=4) > 0.4
    return {
        "step":         step,
        "event":        CRIME_EVENTS[ev % len(CRIME_EVENTS)],
        "perpetrator":  event_int(origin, step, 1000, channel=1),
        "location":     (int(event_float(origin, step, channel=2) * 100),
                         int(event_float(origin, step, channel=3) * 100)),
        "evidence":     left_evidence,
        "clue_quality": event_float(origin, step, channel=5) if left_evidence else 0.0
    }
```

### Technology Tree
```python
TECH_CATEGORIES = ["agriculture","metallurgy","navigation","medicine",
                   "architecture","warfare","philosophy","engineering"]

def civilisation_tech(city_x, city_y, world_seed, era):
    """What technology did this civilisation develop in this era? O(1)."""
    origin   = chain_origin_seed(city_x, city_y, world_seed, chain_type_salt=300)
    tech_cat = event_int(origin, era, len(TECH_CATEGORIES))
    # 40% chance of continuing previous category — causal continuity without sequential compute
    if era > 0 and event_float(origin, era, channel=6) < 0.4:
        tech_cat = event_int(origin, era-1, len(TECH_CATEGORIES))
    adv = event_float(origin, era, channel=1)
    return {"era": era, "category": TECH_CATEGORIES[tech_cat],
            "advancement": adv, "discovery": adv > 0.85}
```

### Branching Chains (Alternate History)
```python
def branching_event(origin_seed, depth, branch=0):
    """Branch 0 = main timeline; branch > 0 = alternate outcome. Both deterministic."""
    branch_seed = position_hash(origin_seed, depth, branch, 0)
    return {"event_type": int(hash_to_float(branch_seed) * 10),
            "outcome":    hash_to_float(branch_seed >> 16),
            "forks":      hash_to_float(branch_seed >> 32) > 0.7,
            "branch":     branch}

def timeline_fork(origin_seed, fork_depth):
    return {"fork_depth": fork_depth,
            "branch_a":   branching_event(origin_seed, fork_depth, branch=0),
            "branch_b":   branching_event(origin_seed, fork_depth, branch=1)}
```

## Hierarchy Pattern

```python
def hierarchical_chain_seed(parent_chain_seed, parent_depth, local_x, local_y):
    """Local chain inherits the parent chain's state at a given depth.
    Cities founded during a golden_age era inherit better founding conditions."""
    return position_hash(local_x, local_y, 0, event_at_depth(parent_chain_seed, parent_depth))
```

```
Empire chain  (depth = era index)
  └─ City chain  (origin seed inherits empire state at founding era)
       └─ NPC chain  (origin seed inherits city state at birth era)
```

## Composing with Zero-Temporal

| Layer | Type | Handles |
|-------|------|---------|
| Zero-Temporal | Continuous | Weather, season, market price at any epoch |
| Zero-Causal | Discrete | Battles, discoveries, betrayals at any depth |
| Composed | Full world | `era = epoch // 100` bridges both layers |

```python
def world_state_at_epoch(x, y, epoch, world_seed):
    era    = epoch // 100
    causal = civilisation_event(x, y, world_seed, era)
    # Zero-Temporal continuous state modulated by causal discrete state
    return {"era_event": causal, "epoch": epoch}
```

## Anti-Patterns

```python
# BAD: Iterating to depth N — O(N), defeats direct access
state = initial_seed
for depth in range(target_depth): state = hash_next(state)

# BAD: Storing chain events — defeats zero-bytes principle
chain_cache = {d: compute_event(origin, d) for d in range(1000)}

# BAD: Wall-clock time in origin seed — non-deterministic
origin_seed = int(time.time()) ^ position_hash(x, y, 0, world_seed)

# BAD: Recursion — O(N), stack overflow at depth
def bad_event(origin, depth):
    prev = bad_event(origin, depth-1)   # requires N-1 to compute N
    return hash_based_on(prev)

# GOOD: Direct depth access, O(1)
event   = civilisation_event(city_x, city_y, world_seed, target_depth)
ancient = civilisation_event(city_x, city_y, world_seed, depth=500)
```

## Debugging Checklist

When chain events differ across machines:
1. Check struct format — `'<qqq'` little-endian signed 64-bit
2. Check `chain_type_salt` — same salt must be used at all call sites for the same chain type
3. Check depth type — must be Python `int`, never `float`; `struct.pack` will raise on float

When chains feel random with no causal structure:
- Increase transition bias multipliers — `2.0`/`3.0` create stronger causal pull
- Reduce event type count — fewer categories = stronger pattern emergence
- Add 40% continuation probability: `if event_float(seed, depth, 6) < 0.4: use_prev_type`

When all chains look the same:
- Verify `chain_origin_seed` encodes the spatial position and chain_type_salt
- Different `chain_type_salt` values for different chain types at the same location

## Determinism Verification

```python
def verify_causal(origin_x, origin_y, world_seed, max_depth=50):
    origin = chain_origin_seed(origin_x, origin_y, world_seed)
    # Determinism
    for d in [0, 1, 10, 25, max_depth]:
        assert event_at_depth(origin, d) == event_at_depth(origin, d), f"Non-deterministic at {d}"
    # O(1): depth N matches regardless of query order
    deep  = event_at_depth(origin, max_depth)
    _     = event_at_depth(origin, max_depth - 1)
    assert event_at_depth(origin, max_depth) == deep, "Order-dependent!"
    # Past stability
    past  = event_at_depth(origin, 0)
    _     = event_at_depth(origin, max_depth)
    assert event_at_depth(origin, 0) == past, "Historical depth non-deterministic!"
    # Chain individuality
    other = chain_origin_seed(origin_x + 1, origin_y, world_seed)
    same  = sum(1 for d in range(max_depth) if event_at_depth(origin,d) == event_at_depth(other,d))
    assert same < max_depth // 2, "Chains from different origins are too similar!"
```

## Usage

1. **Define chain type** — political, genetic, criminal; use `chain_type_salt` to distinguish types at the same location
2. **Define event vocabulary** — bounded list; fewer types = stronger causal patterns
3. **Design transition biases** — weight dict keyed by previous type index; creates plausibility without sequential compute
4. **Choose depth semantics** — one era, one generation, one crime step; declare at design time
5. **Handle terminals** — detect terminal event types (capture, collapse) and stop querying deeper
6. **Compose with Zero-Temporal** — continuous state there; discrete narrative events here; bridge via `era = epoch // N`
7. **Apply hierarchy** — child chain seeds inherit parent chain state at founding depth
8. **Verify** — run `verify_causal` confirming O(1) access, past stability, chain individuality

**Core principle:** The past is not a log. History is not a simulation. Depth is a coordinate. Zero-Causal jumps to any moment in any history in O(1). Zero bytes of history are stored.
