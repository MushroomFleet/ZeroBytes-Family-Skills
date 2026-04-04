---
name: zero-field
description: Space-is-seed continuous influence field methodology. Extends ZeroBytes from discrete entity properties into continuous scalar and vector fields that exist independently of any entity set. Use when a developer asks for mana or magic intensity maps, disease or pollution spread without simulation, gravity or mass distribution fields, faction territory as a continuous function, heat or radiation zones, signal strength maps, predator pressure fields, or any system where a spatially continuous influence exists across the world and entities are derived from the field (not the other way round). Triggers on phrases like "influence field", "continuous field", "territory map", "mana density", "radiation zone", "field-first", "entity-free", "signal strength map", "zero-field", "spawn from field peaks", "no entity enumeration", or when a ZeroBytes system needs a property that permeates space rather than residing at discrete entity positions.
---

# Zero-Field: Space-is-Seed Continuous Influence Field Generation

Extends [ZeroBytes](../zerobytes/SKILL.md) from discrete entity-addressed generation into continuous scalar and vector fields that are properties of space itself. The spatial coordinate IS the seed for the field value — no entity enumeration, no influence summation, no simulation.

**The philosophical inversion:** Entities do not produce fields; fields produce entities. Mana doesn't radiate from magical creatures — magical creatures exist where mana density peaks. Faction territory isn't a union of claimed tiles — it's a continuous field whose border is where two fields reach equilibrium. This eliminates the most expensive traditional pattern: iterating all nearby entities to sum their contributions.

## The Five Extended Laws

1. **O(1) Field Access**: Field strength at any point is computable directly from that point's coordinate — no entity iteration, no distance summation, no simulation
2. **Spatial Continuity**: Adjacent points produce smoothly varying values — discontinuities must be intentionally designed, not accidental
3. **Multi-Scale Coherence**: Fields have structure at macro (regional), meso (local), and micro (fine texture) scales — encoded via layered octaves with independent seeds
4. **Field Orthogonality**: Independent field types (mana, disease, faction_A) use independent salts — queried and combined without cross-contamination
5. **Determinism**: Same spatial coordinate → same field value across all machines and execution orders

## Core Pattern

```python
import struct, math
import xxhash

def position_hash(x, y, z, salt=0):
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqq', x, y, z))
    return h.intdigest()

def hash_to_float(h):
    return (h & 0xFFFFFFFF) / 0x100000000

def coherent_value(x, y, seed, octaves=4):
    """ZeroBytes coherent noise — the spatial foundation of Zero-Field."""
    value, amp, freq, max_amp = 0.0, 1.0, 1.0, 0.0
    for i in range(octaves):
        x0, y0 = int(x * freq), int(y * freq)
        sx = (x*freq)%1; sx = sx*sx*(3-2*sx)
        sy = (y*freq)%1; sy = sy*sy*(3-2*sy)
        n00 = hash_to_float(position_hash(x0,   y0,   0, seed+i))*2-1
        n10 = hash_to_float(position_hash(x0+1, y0,   0, seed+i))*2-1
        n01 = hash_to_float(position_hash(x0,   y0+1, 0, seed+i))*2-1
        n11 = hash_to_float(position_hash(x0+1, y0+1, 0, seed+i))*2-1
        nx0 = n00*(1-sx)+n10*sx;  nx1 = n01*(1-sx)+n11*sx
        value += amp*(nx0*(1-sy)+nx1*sy);  max_amp += amp;  amp *= 0.5;  freq *= 2.0
    return value / max_amp

def scalar_field(x, y, world_seed, field_salt,
                 macro_freq=0.002, meso_freq=0.02, micro_freq=0.1,
                 macro_weight=0.6, meso_weight=0.3, micro_weight=0.1, octaves=4):
    """
    General-purpose continuous scalar field. Returns ~-1.0..1.0.
    field_salt: unique integer per field type — the primary orthogonality guarantee.
    Separate salts by at least 1000 to avoid hash collision artefacts.
    """
    macro = coherent_value(x*macro_freq, y*macro_freq, world_seed+field_salt,   octaves=octaves)
    meso  = coherent_value(x*meso_freq,  y*meso_freq,  world_seed+field_salt+1, octaves=max(1,octaves-2))
    micro = coherent_value(x*micro_freq, y*micro_freq, world_seed+field_salt+2, octaves=1)
    return macro*macro_weight + meso*meso_weight + micro*micro_weight

def vector_field(x, y, world_seed, field_salt):
    """Continuous 2D vector field — wind, currents, magical flow, migration pressure."""
    dx = scalar_field(x, y, world_seed, field_salt,       macro_freq=0.003, meso_freq=0.03)
    dy = scalar_field(x, y, world_seed, field_salt+10000, macro_freq=0.003, meso_freq=0.03)
    mag = (dx**2 + dy**2)**0.5
    if mag < 1e-6: return {"dx": 0.0, "dy": 0.0, "magnitude": 0.0, "direction": 0.0}
    return {"dx": dx/mag, "dy": dy/mag, "magnitude": mag, "direction": math.atan2(dy, dx)}
```

## Field Topology

```python
def field_local_maximum(x, y, world_seed, field_salt, search_radius=5, step=1):
    """Is this point a local field peak? Use to determine entity spawn eligibility."""
    centre = scalar_field(x, y, world_seed, field_salt)
    for dx in range(-search_radius, search_radius+1, step):
        for dy in range(-search_radius, search_radius+1, step):
            if dx == 0 and dy == 0: continue
            if scalar_field(x+dx, y+dy, world_seed, field_salt) > centre: return False
    return True

def field_gradient(x, y, world_seed, field_salt, epsilon=1.0):
    """Slope direction — agents move up-gradient toward field peaks."""
    fx  = scalar_field(x,         y,         world_seed, field_salt)
    fxp = scalar_field(x+epsilon, y,         world_seed, field_salt)
    fyp = scalar_field(x,         y+epsilon, world_seed, field_salt)
    gx, gy = (fxp-fx)/epsilon, (fyp-fx)/epsilon
    return {"gx": gx, "gy": gy, "magnitude": (gx**2+gy**2)**0.5}

def territorial_border(x, y, world_seed, salt_a, salt_b, threshold=0.05):
    """True when two faction fields are within threshold — the frontier zone."""
    return abs(scalar_field(x,y,world_seed,salt_a) - scalar_field(x,y,world_seed,salt_b)) < threshold

def territorial_ownership(x, y, world_seed, n_factions=5, base_salt=3000):
    """Which faction's field dominates this point? O(n_factions) per query."""
    fields = [(i, scalar_field(x, y, world_seed, base_salt + i*100)) for i in range(n_factions)]
    fields.sort(key=lambda f: f[1], reverse=True)
    owner, strength = fields[0];  second = fields[1][1] if len(fields) > 1 else -1.0
    return {"owner": owner, "strength": strength,
            "margin": strength-second, "contested": abs(strength-second) < 0.05}
```

## Quick Recipes

### Mana / Magic Intensity
```python
MANA_SALT    = 1000
LEYLINE_SALT = 1001

def mana_field(x, y, world_seed):
    """Continuous mana density. High mana → magical entities spawn at peaks."""
    macro = coherent_value(x*0.001, y*0.001, world_seed+MANA_SALT,   octaves=4)
    meso  = coherent_value(x*0.01,  y*0.01,  world_seed+MANA_SALT+1, octaves=3)
    micro = coherent_value(x*0.05,  y*0.05,  world_seed+MANA_SALT+2, octaves=2)
    raw   = macro*0.5 + meso*0.35 + micro*0.15
    ley   = coherent_value(x*0.005, y*0.005, world_seed+LEYLINE_SALT, octaves=2)
    leyline_boost = max(0.0, (abs(ley) - 0.7) * 3.0)
    intensity = max(0.0, raw + leyline_boost)
    return {"intensity": intensity, "is_leyline": leyline_boost > 0.2,
            "spawn_magical": intensity > 0.6, "spawn_tier": int(intensity * 5)}
```

### Faction Territory
```python
FACTION_BASE_SALT = 3000

def faction_territory(x, y, world_seed, n_factions=5):
    """No tile-ownership table. No stored borders. Territory computed in O(n_factions)."""
    return territorial_ownership(x, y, world_seed, n_factions, FACTION_BASE_SALT)
```

### Disease / Pollution (No Simulation)
```python
DISEASE_SALT = 2000

def disease_field(x, y, epoch, world_seed):
    """Disease concentration — no diffusion simulation. Compose with Zero-Temporal for outbreaks."""
    geo_susceptibility = coherent_value(x*0.008, y*0.008, world_seed+DISEASE_SALT, octaves=3)
    # outbreak_epoch ties disease intensity to a world-time tick (Zero-Temporal composure)
    from zero_temporal import temporal_hash
    outbreak_seed  = temporal_hash(x//200, y//200, 0, epoch, world_seed+DISEASE_SALT)
    outbreak_local = hash_to_float(outbreak_seed)
    density = max(0.0, geo_susceptibility) * outbreak_local
    return {"density": density, "is_hotspot": density > 0.7,
            "virulence": hash_to_float(outbreak_seed >> 32)}
```

### Wind / Ocean Current
```python
WIND_SALT    = 4000
CURRENT_SALT = 4001

def wind_at(x, y, world_seed):
    """Deterministic wind vector — no pressure map, no fluid simulation."""
    wind  = vector_field(x, y, world_seed, WIND_SALT)
    speed = max(0.0, scalar_field(x, y, world_seed, WIND_SALT+50,
                                  macro_freq=0.004, meso_freq=0.04) + 0.5) * 100
    return {"direction": wind["direction"], "speed_kmh": speed,
            "is_gale": speed > 75, "vector": (wind["dx"]*speed, wind["dy"]*speed)}
```

### Heat / Radiation (3D)
```python
HEAT_SALT = 6000

def heat_field(x, y, z, world_seed):
    """3D heat field — caves, volcanic terrain, space. z < 0 = underground."""
    geothermal   = max(0.0, -z * 0.01)
    surface_heat = max(0.0, scalar_field(x, y, world_seed, HEAT_SALT, macro_freq=0.004, meso_freq=0.04))
    volcanic     = coherent_value(x*0.02, y*0.02, world_seed+HEAT_SALT+1, octaves=2)
    hotspot      = max(0.0, (volcanic - 0.5) * 4.0)
    total        = geothermal + surface_heat*0.4 + hotspot
    return {"temperature": total, "is_hazardous": total > 1.5,
            "lava_present": hotspot > 2.0, "heat_damage": max(0.0, total-1.0)*10}
```

### Ecosystem Pressure
```python
PREDATOR_SALT   = 5000
PREY_SALT       = 5001
VEGETATION_SALT = 5002

def ecosystem_field(x, y, world_seed):
    """Field-first ecology — entities spawn at peaks, not the other way round."""
    veg      = scalar_field(x, y, world_seed, VEGETATION_SALT, macro_freq=0.005, meso_freq=0.02)
    prey_raw = scalar_field(x, y, world_seed, PREY_SALT,       macro_freq=0.006, meso_freq=0.025)
    pred_raw = scalar_field(x, y, world_seed, PREDATOR_SALT,   macro_freq=0.003, meso_freq=0.01)
    prey     = max(0.0, prey_raw) * max(0.0, veg + 0.3)
    predator = max(0.0, pred_raw) * max(0.0, prey + 0.2)
    return {"vegetation": max(0.0,veg), "prey_density": prey, "predator_density": predator,
            "spawn_herbivore": prey > 0.5, "spawn_predator": predator > 0.6,
            "danger_level": predator}
```

## Composing with ZeroBytes and Zero-Quadratic

| Layer | Type | Answers |
|-------|------|---------|
| Zero-Field | Continuous | What permeates this point? |
| ZeroBytes | Discrete point | What entity is at this position? |
| Zero-Quadratic | Discrete pair | What is the relationship between two entities? |

Fields set ambient context. ZeroBytes generates discrete entities within that context. Entities spawn at field peaks — the field determines *where*, ZeroBytes determines *what*.

## Anti-Patterns

```python
# BAD: Entity-first field — iterates all entities per query point
def bad_mana(x, y, entities):
    return sum(e["power"] / dist(x,y,e["x"],e["y"]) for e in entities)  # O(N), entities stored

# BAD: Storing the field — 1M entries for a 1000×1000 map
mana_map = [[mana_field(x, y, seed) for x in range(1000)] for y in range(1000)]

# BAD: Overlapping salts — fields contaminate each other
f1 = scalar_field(x, y, seed, salt=0)
f2 = scalar_field(x, y, seed, salt=0)   # same salt = identical field

# BAD: Treating territory as tile ownership
def bad_territory(x, y): return nearest_entity_owner(x, y, all_entities)

# GOOD: Orthogonal salts, computed on demand
mana    = scalar_field(x, y, world_seed, MANA_SALT)
disease = scalar_field(x, y, world_seed, DISEASE_SALT)

# GOOD: Entities are peaks, not sources
if field_local_maximum(x, y, world_seed, MANA_SALT):
    spawn_entity(x, y, "magical_creature")
```

## Debugging Checklist

When field values differ across machines:
1. Check struct format — `'<qqq'` little-endian 64-bit; avoid platform fast-math flags affecting float precision
2. Check octave count — must be identical across all call sites for the same field type
3. Check frequency parameters — `macro_freq`/`meso_freq` must match between generation and query

When field shows visible tile or chunk boundaries:
- Reduce `macro_freq` for smoother regional transitions
- Increase `octaves` for more intermediate-scale coherence

When field peaks are too rare or too common:
- Macro-heavy (`macro_weight=0.7`): large sparse peaks; meso-heavy: many small peaks
- Tune spawn threshold by sampling field histogram at known-good coordinates

When two fields appear correlated despite different salts:
- Salts must differ by at least `octaves + 3`; use 1000, 2000, 3000 — not 1, 2, 3

## Determinism Verification

```python
def verify_field(world_seed, test_positions, field_salt=MANA_SALT):
    # Determinism
    for x, y in test_positions:
        v1 = scalar_field(x, y, world_seed, field_salt)
        v2 = scalar_field(x, y, world_seed, field_salt)
        assert abs(v1-v2) < 1e-9, f"Non-deterministic at ({x},{y})"
    # Continuity: adjacent points must not wildly diverge
    for x, y in test_positions:
        v0 = scalar_field(x,   y,   world_seed, field_salt)
        vx = scalar_field(x+1, y,   world_seed, field_salt)
        vy = scalar_field(x,   y+1, world_seed, field_salt)
        assert abs(v0-vx) < 0.5, f"X discontinuity at ({x},{y})"
        assert abs(v0-vy) < 0.5, f"Y discontinuity at ({x},{y})"
    # Orthogonality: independent salts must not correlate
    other = field_salt + 1000
    va = [scalar_field(x,y,world_seed,field_salt) for x,y in test_positions]
    vb = [scalar_field(x,y,world_seed,other)      for x,y in test_positions]
    ma, mb = sum(va)/len(va), sum(vb)/len(vb)
    corr = sum((a-ma)*(b-mb) for a,b in zip(va,vb))
    assert abs(corr) < len(test_positions)*0.3, "Fields correlated — check salts!"
```

## Usage

1. **Identify field type** — mana, territory, disease, heat; assign a unique salt constant (1000, 2000, 3000…)
2. **Choose frequency parameters** — `macro_freq = 0.002–0.005` for large-scale structure; `meso_freq` = 10× macro
3. **Choose weight balance** — `macro_weight = 0.6` for smooth; increase `meso_weight` for local variation
4. **Design peaks** — tune `field_local_maximum` search radius to control entity spawn density
5. **Design boundaries** — `territorial_border` for frontier zones; `territorial_ownership` for control assignment
6. **Ensure orthogonality** — salt separation ≥ 1000 between field types
7. **Compose with ZeroBytes** — field = ambient context; ZeroBytes = discrete entities at field peaks
8. **Verify** — run `verify_field` confirming determinism, continuity, and orthogonality

**Core principle:** Space is not a container for entities — space has properties of its own. Zero-Field generates those properties from coordinates alone. Entities are optional: they are peaks, not sources. Zero bytes describe the field. The field describes the world.
