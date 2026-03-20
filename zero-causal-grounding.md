---
name: zero-causal
description: Chain-is-seed O(1) causal chain procedural determinism methodology. Extends ZeroBytes from static spatial generation into deterministic causal histories and futures. Use when a developer asks for procedural civilisation history, genetic lineage generation, criminal causal chain reconstruction, technology tree evolution, narrative event chains, disease outbreak propagation history, geological event sequences, or any system where a chain of causally-linked events must be reproducible at any point in the chain without stepping through all preceding events. Triggers on phrases like "procedural history", "causal chain", "event sequence", "direct jump to epoch", "lineage generation", "chain-is-seed", "zero-causal", "deterministic timeline", "event at depth N", "without replaying", or when a Zero-Temporal system needs not just temporal properties but a narrative chain of discrete causal events that can be queried at any depth in O(1).
---

# Zero-Causal: Chain-is-Seed Deterministic Causal Event Generation

Extends [ZeroBytes](zerobytes/SKILL.md) and [Zero-Temporal](zero-temporal/SKILL.md) from coordinate-addressed properties into deterministic causal chains. The chain origin + depth IS the seed. Any event at any depth in any causal history is directly computable in O(1) without replaying preceding events.

## The Conceptual Leap from Zero-Temporal

Zero-Temporal asks: *What is the state of this position at this epoch?*
Zero-Causal asks: *What discrete event occurred at step N in this causal chain, and what did it cause?*

Zero-Temporal treats time as a continuous coordinate — smooth seasonal cycles, daily weather variation. Zero-Causal models time as a sequence of discrete causal events where each event is a specific, narratively meaningful occurrence: a battle, a founding, a betrayal, an invention, a mutation. The chain has internal causal logic — some events make the next event more or less probable — but the entire chain is queryable at any depth without stepping through it.

**The critical property:** Unlike a Markov chain, which requires sequential computation to reach step N, Zero-Causal allows direct access to event N via `position_hash(origin_seed, N, 0, 0)`. The chain is reconstructable from any point in either direction.

## The Five Extended Laws

Every causal chain system must satisfy:

1. **O(1) Depth Access**: Event N is directly computable from `(origin_seed, N)` without computing events 0 through N-1 — the depth IS the coordinate
2. **Causal Plausibility**: Event N is statistically influenced by the *type* of event N-1 without requiring N-1 to be computed — encode transition probabilities as hash-derived weights biased by event type index
3. **Chain Individuality**: Different origins produce statistically distinct chain patterns — a maritime city's history differs from a mountain city's; encode origin character in the chain seed, not as runtime state
4. **Branching Determinism**: If a chain branches, both branches are independently seeable from the same origin seed at the same depth — use salt to distinguish branches
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
    """
    Derive the root seed for a causal chain from its spatial origin.
    Different chain types (political, military, economic) use different salts.
    """
    return position_hash(origin_x, origin_y, chain_type_salt, world_seed)

def event_at_depth(origin_seed, depth, salt=0):
    """
    Core Zero-Causal function: O(1) access to any event in the chain.
    depth: integer ≥ 0; higher = further along the causal chain.
    NEVER iterate depth 0→N to reach depth N — always query directly.
    """
    return position_hash(origin_seed, depth, 0, salt)

def event_float(origin_seed, depth, channel=0):
    """Derive a float property from the event at this depth."""
    return hash_to_float(event_at_depth(origin_seed, depth, salt=channel))

def event_int(origin_seed, depth, modulus, channel=0):
    """Derive a bounded integer from the event at this depth."""
    return int(event_float(origin_seed, depth, channel) * modulus)
```

## Causal Plausibility Without Sequential Computation

The challenge of causal chains is that event N should *feel* caused by event N-1 — a war is more likely to follow a border dispute than a harvest festival. Zero-Causal solves this without requiring N-1 to be computed to reach N:

```python
def biased_event_type(origin_seed, depth, event_types, transition_bias):
    """
    Select event type at depth, biased by the *type* of the previous event.
    transition_bias: dict mapping (prev_type_index → weight_adjustments)
    The previous event's *type index* (not its full value) encodes the bias.
    """
    # Determine previous event type without fetching full event N-1
    if depth == 0:
        prev_type_index = 0  # root event: neutral prior
    else:
        prev_raw        = event_int(origin_seed, depth - 1, len(event_types), channel=0)
        prev_type_index = prev_raw  # type of the preceding event

    # Apply transition weights
    weights = [1.0] * len(event_types)
    if prev_type_index in transition_bias:
        for i, adjustment in enumerate(transition_bias[prev_type_index]):
            weights[i] *= adjustment

    # Weighted selection using event hash at this depth
    raw        = event_float(origin_seed, depth, channel=0)
    total      = sum(weights)
    cumulative = 0.0
    for i, w in enumerate(weights):
        cumulative += w / total
        if raw < cumulative:
            return i
    return len(event_types) - 1
```

## Quick Recipes

### Civilisation History
```python
POLITICAL_EVENTS = [
    "founding", "expansion", "civil_war", "golden_age",
    "plague",   "invasion",  "reform",   "collapse",   "revival"
]

# Transition bias: what events tend to follow which
POLITICAL_TRANSITIONS = {
    0: [1.0, 2.0, 0.5, 1.0, 0.5, 0.5, 1.0, 0.2, 0.5],  # founding → expansion likely
    2: [0.5, 0.5, 0.3, 0.5, 1.5, 2.0, 1.0, 1.5, 0.5],  # civil_war → invasion/collapse likely
    3: [0.5, 1.5, 0.3, 0.5, 0.8, 0.8, 1.5, 0.2, 0.5],  # golden_age → expansion/reform likely
    7: [0.5, 0.5, 0.5, 0.5, 0.5, 0.5, 1.0, 0.3, 2.5],  # collapse → revival likely
}

def civilisation_event(city_x, city_y, world_seed, historical_depth):
    """
    What happened to this city at this point in its history?
    Direct O(1) access — no need to compute events 0 through depth-1.
    """
    origin_seed = chain_origin_seed(city_x, city_y, world_seed, chain_type_salt=100)
    event_type  = biased_event_type(origin_seed, historical_depth,
                                    POLITICAL_EVENTS, POLITICAL_TRANSITIONS)
    magnitude   = event_float(origin_seed, historical_depth, channel=1)
    duration    = 1 + event_int(origin_seed, historical_depth, 20, channel=2)

    return {
        "depth":      historical_depth,
        "event":      POLITICAL_EVENTS[event_type],
        "magnitude":  magnitude,
        "duration":   duration,  # in epochs
        "lasting_effect": magnitude > 0.75  # high-magnitude events leave permanent marks
    }

def city_history(city_x, city_y, world_seed, from_depth, to_depth):
    """Generate a sequence of historical events. Each is O(1) — total is O(range)."""
    return [civilisation_event(city_x, city_y, world_seed, d)
            for d in range(from_depth, to_depth)]

def historical_archaeology(city_x, city_y, world_seed, target_depth):
    """Jump directly to a specific historical moment. No replay."""
    return civilisation_event(city_x, city_y, world_seed, target_depth)
```

### Genetic Lineage
```python
MUTATION_TYPES = [
    "beneficial_metabolism", "neutral_coloration", "harmful_immunity",
    "beneficial_size",       "neutral_behaviour",  "beneficial_cognition",
    "harmful_reproduction",  "neutral_lifespan"
]

def genetic_generation(species_x, species_y, world_seed, generation):
    """
    What mutation characterises this generation of this species?
    Direct O(1) access to any generation — no need to simulate all preceding ones.
    """
    origin_seed   = chain_origin_seed(species_x, species_y, world_seed, chain_type_salt=200)
    mutation_idx  = event_int(origin_seed, generation, len(MUTATION_TYPES), channel=0)
    fitness_delta = (event_float(origin_seed, generation, channel=1) - 0.5) * 2  # -1.0 to +1.0
    is_dominant   = event_float(origin_seed, generation, channel=2) > 0.6

    return {
        "generation":   generation,
        "mutation":     MUTATION_TYPES[mutation_idx],
        "fitness":      fitness_delta,
        "is_dominant":  is_dominant,
        "causes_split": abs(fitness_delta) > 0.8  # strong mutations may cause speciation
    }

def lineage_at_generation(species_x, species_y, world_seed, generation):
    """What is the cumulative fitness of this lineage at this generation?"""
    # We can estimate cumulative fitness without replaying all generations
    # by sampling key generations and interpolating
    cumulative = 0.0
    sample_interval = max(1, generation // 10)
    for g in range(0, generation, sample_interval):
        event = genetic_generation(species_x, species_y, world_seed, g)
        cumulative += event["fitness"] * sample_interval
    return cumulative
```

### Criminal Causal Chain
```python
CRIME_EVENTS = [
    "initial_grievance", "planning",    "recruitment",  "resource_acquisition",
    "first_offence",     "escalation",  "detection",    "evasion",
    "accomplice_betrayal","confrontation","capture",     "escape"
]

def crime_chain_event(crime_x, crime_y, crime_epoch, world_seed, step):
    """
    Reconstruct any step in a crime's causal chain.
    Useful for procedural investigation gameplay: piece together the chain backwards.
    """
    # Origin seed includes the crime's location AND time
    from zero_temporal import temporal_hash  # compose with Zero-Temporal
    temporal_s  = temporal_hash(crime_x, crime_y, 0, crime_epoch, world_seed)
    origin_seed = position_hash(temporal_s, 0, 0, 500)

    event_type  = event_int(origin_seed, step, len(CRIME_EVENTS), channel=0)
    perpetrator = event_int(origin_seed, step, 1000, channel=1)  # NPC ID
    location    = (
        int(event_float(origin_seed, step, channel=2) * 100),
        int(event_float(origin_seed, step, channel=3) * 100)
    )
    left_evidence = event_float(origin_seed, step, channel=4) > 0.4

    return {
        "step":         step,
        "event":        CRIME_EVENTS[event_type % len(CRIME_EVENTS)],
        "perpetrator":  perpetrator,
        "location":     location,
        "evidence":     left_evidence,
        "clue_quality": event_float(origin_seed, step, channel=5) if left_evidence else 0.0
    }

def reconstruct_crime(crime_x, crime_y, crime_epoch, world_seed, max_steps=12):
    """Reconstruct the full causal chain of a crime. Each step is O(1)."""
    chain = []
    for step in range(max_steps):
        event = crime_chain_event(crime_x, crime_y, crime_epoch, world_seed, step)
        chain.append(event)
        # Chain terminates at capture or escape
        if event["event"] in ("capture", "escape"):
            break
    return chain
```

### Technology Tree Evolution
```python
TECH_CATEGORIES = ["agriculture", "metallurgy", "navigation", "medicine",
                   "architecture", "warfare",    "philosophy", "engineering"]

def civilisation_tech(city_x, city_y, world_seed, era):
    """
    What technology did this civilisation develop in this era?
    Each era's tech is seeded from the era number — no simulation of preceding eras.
    """
    origin_seed  = chain_origin_seed(city_x, city_y, world_seed, chain_type_salt=300)
    tech_cat     = event_int(origin_seed, era, len(TECH_CATEGORIES), channel=0)
    advancement  = event_float(origin_seed, era, channel=1)
    is_discovery = advancement > 0.85  # rare breakthrough vs incremental progress

    # Tech tends to build on previous category — causal plausibility
    if era > 0:
        prev_cat = event_int(origin_seed, era - 1, len(TECH_CATEGORIES), channel=0)
        # 40% chance of continuing previous category
        if event_float(origin_seed, era, channel=6) < 0.4:
            tech_cat = prev_cat

    return {
        "era":         era,
        "category":    TECH_CATEGORIES[tech_cat],
        "advancement": advancement,
        "discovery":   is_discovery,
        "tech_level":  era + int(advancement * 3)
    }
```

### Branching Causal Chains
```python
def branching_event(origin_seed, depth, branch=0):
    """
    Query a specific branch of a causal chain.
    Branch 0 = main timeline; branch > 0 = alternate outcome.
    Both branches are deterministic from the same origin seed.
    """
    # Use branch index as an additional salt dimension
    branch_seed = position_hash(origin_seed, depth, branch, 0)
    return {
        "event_type":    int(hash_to_float(branch_seed) * 10),
        "outcome":       hash_to_float(branch_seed >> 16),
        "leads_to_split": hash_to_float(branch_seed >> 32) > 0.7,
        "branch":        branch
    }

def timeline_fork(origin_seed, fork_depth):
    """
    At a fork point, generate both branches deterministically.
    Useful for alternate-history gameplay or narrative choice trees.
    """
    main_branch = branching_event(origin_seed, fork_depth, branch=0)
    alt_branch  = branching_event(origin_seed, fork_depth, branch=1)
    return {"fork_depth": fork_depth, "branch_a": main_branch, "branch_b": alt_branch}
```

## Composing with Zero-Temporal

Zero-Causal and Zero-Temporal are complementary:
- **Zero-Temporal** handles *continuous* temporal properties: weather, season, market price
- **Zero-Causal** handles *discrete* narrative events: battles, discoveries, betrayals

```python
def world_state_at_epoch(x, y, epoch, world_seed):
    """
    Full world state: continuous properties (Zero-Temporal) +
    discrete historical events (Zero-Causal).
    """
    # Continuous layer (Zero-Temporal)
    from zero_temporal import weather, terrain_state
    continuous = {
        "weather":  weather(x, y, epoch, world_seed),
        "terrain":  terrain_state(x, y, epoch, world_seed)
    }

    # Discrete layer (Zero-Causal): find which historical era this epoch falls in
    era = epoch // 100  # each era = 100 epochs
    causal = civilisation_event(x, y, world_seed, era)

    # Compose: a civilisation in golden_age has better weather resilience
    if causal["event"] == "golden_age":
        continuous["weather"]["storm_active"] = False  # cities manage weather better

    return {"continuous": continuous, "discrete": causal, "epoch": epoch}
```

## Hierarchy Pattern

ZeroBytes hierarchy: `parent_seed → child_seed via position`
Zero-Causal adds: `parent_chain_seed → child_chain_seed via local depth`

```python
def hierarchical_chain_seed(parent_chain_seed, parent_depth, local_origin_x, local_origin_y):
    """
    A local chain whose origin inherits the parent chain's state at a given depth.
    A city's founding in era 5 of its empire's history inherits the empire's state at era 5.
    """
    parent_event_seed = event_at_depth(parent_chain_seed, parent_depth)
    return position_hash(local_origin_x, local_origin_y, 0, parent_event_seed)
```

**Practical consequence:** Cities founded during an empire's "golden_age" era inherit statistically better founding conditions — through seed inheritance, not stored empire state.

## Anti-Patterns

```python
# BAD: Sequential iteration to reach depth N — defeats O(1) access
state = initial_seed
for depth in range(target_depth):
    state = hash_next(state)           # O(N) and unnecessary

# BAD: Storing computed chain events
chain_cache = {}
for d in range(1000):
    chain_cache[d] = compute_event(origin, d)   # 1000 entries accumulate

# BAD: Real-time clock in chain seed
import time
origin_seed = int(time.time()) ^ position_hash(x, y, 0, world_seed)  # non-deterministic

# BAD: Requiring N-1 to compute N
def bad_event(origin, depth):
    prev = bad_event(origin, depth-1)  # recursive, O(N), stack overflow at depth
    return hash_based_on(prev)

# GOOD: Direct depth access
event = civilisation_event(city_x, city_y, world_seed, target_depth)

# GOOD: Historical archaeology in O(1)
ancient = historical_archaeology(city_x, city_y, world_seed, depth=500)
```

## Debugging Checklist

When chain events differ across machines:
1. Check struct pack format — `'<qqq'` (little-endian signed 64-bit) must be consistent
2. Check `origin_seed` derivation — verify `chain_origin_seed` uses same salt values everywhere
3. Check depth type — depth must be a Python `int`, not a float; `struct.pack('<qqq', seed, float_depth, 0)` will raise

When chains feel random with no causal structure:
- Increase transition bias weights — `2.0` and `3.0` multipliers create stronger causal pull
- Add explicit continuation probability: `if depth > 0 and event_float(seed, depth, 6) < 0.4: use_prev_type`
- Reduce the event type count — fewer categories = stronger pattern emergence

When chains "always end the same way":
- Check that terminal event detection uses a threshold, not a fixed depth
- Introduce chain-specific character via origin salt — maritime cities should not follow identical chains to mountain cities

## Determinism Verification

```python
def verify_causal(origin_x, origin_y, world_seed, max_depth=50):
    """Verify O(1) access and determinism at arbitrary depths."""
    origin_seed = chain_origin_seed(origin_x, origin_y, world_seed)

    # Determinism at same depth
    for depth in [0, 1, 10, 25, max_depth]:
        e1 = event_at_depth(origin_seed, depth)
        e2 = event_at_depth(origin_seed, depth)
        assert e1 == e2, f"Non-deterministic at depth {depth}"

    # O(1): depth 50 must match whether or not depth 49 was queried
    deep_direct  = event_at_depth(origin_seed, max_depth)
    _            = event_at_depth(origin_seed, max_depth - 1)  # query N-1 after
    deep_after   = event_at_depth(origin_seed, max_depth)
    assert deep_direct == deep_after, "Depth access is order-dependent!"

    # Historical archaeology: past depth stable on re-query
    past = event_at_depth(origin_seed, 0)
    _    = event_at_depth(origin_seed, max_depth)  # query far depth
    past_again = event_at_depth(origin_seed, 0)
    assert past == past_again, "Historical depth non-deterministic after later query!"

    # Chain individuality: different origins produce different chains
    other_seed = chain_origin_seed(origin_x + 1, origin_y, world_seed)
    same_count = sum(1 for d in range(max_depth)
                     if event_at_depth(origin_seed, d) == event_at_depth(other_seed, d))
    assert same_count < max_depth // 2, "Chains from different origins are too similar!"

    print(f"Zero-Causal verification passed: depth range 0–{max_depth}.")
```

## Usage

When implementing a Zero-Causal system:

1. **Define the chain type** — what kind of events does this chain model? (political, genetic, criminal); use `chain_type_salt` to distinguish types at the same spatial origin
2. **Define the event vocabulary** — a bounded list of event types; fewer = stronger causal patterns; more = richer variation
3. **Design transition biases** — which event types tend to follow which; these create causal plausibility without sequential computation
4. **Choose depth semantics** — what does one depth unit mean? (one era, one generation, one crime step); declare this at design time
5. **Handle terminal events** — if chains should end (capture, extinction, collapse), detect terminal event types and stop querying deeper
6. **Compose with Zero-Temporal** — use Zero-Temporal for continuous state; Zero-Causal for discrete events; compose them at the world-state layer
7. **Apply hierarchy** — child chain seeds inherit from parent chain events at key depths
8. **Verify** — run `verify_causal` confirming O(1) access and chain individuality

**Core principle:** The past is not a log. History is not a simulation. A causal chain is a coordinate space — depth is a dimension, and every event at every depth is directly addressable. Zero-Causal jumps to any moment in any history in O(1). Zero bytes of history are stored.
