---
name: zero-field
description: Space-is-seed continuous influence field methodology. Extends ZeroBytes from discrete entity properties into continuous scalar and vector fields that exist independently of any entity set. Use when a developer asks for mana or magic intensity maps, disease or pollution spread without simulation, gravity or mass distribution fields, faction territory as a continuous function, heat or radiation zones, signal strength maps, predator pressure fields, or any system where a spatially continuous influence exists across the world and entities are derived from the field (not the other way round). Triggers on phrases like "influence field", "continuous field", "territory map", "mana density", "radiation zone", "field-first", "entity-free", "signal strength map", "zero-field", "spawn from field peaks", "no entity enumeration", or when a ZeroBytes system needs a property that permeates space rather than residing at discrete entity positions.
---

# Zero-Field: Space-is-Seed Continuous Influence Field Generation

Extends [ZeroBytes](zerobytes/SKILL.md) from discrete entity-addressed generation into continuous scalar and vector fields that are properties of space itself. The spatial coordinate IS the seed for the field value — no entity enumeration, no influence summation, no simulation. Entities are optional derivations from field peaks, not the source of the field.

## The Conceptual Leap from ZeroBytes

ZeroBytes asks: *What exists at this discrete position?*
Zero-Field asks: *What continuous influence permeates this point in space?*

ZeroBytes generates entities at positions. Zero-Field generates fields that permeate all positions. The philosophical inversion is critical: **entities do not produce fields; fields produce entities.** Mana doesn't radiate from magical creatures — magical creatures exist where mana density is highest. Faction territory isn't the union of claimed tiles — territory is a continuous field and the border is where two fields reach equilibrium.

This inversion eliminates the most expensive pattern in traditional game systems: iterating over all entities near a point to sum their contributions. Zero-Field computes field strength at any point in O(1) from the spatial coordinate alone.

## The Five Extended Laws

Every continuous field system must satisfy:

1. **O(1) Field Access**: Field strength at any point is computable directly from that point's coordinate — no iteration over entities, no distance summation, no simulation propagation
2. **Spatial Continuity**: Adjacent points produce smoothly varying field values — the field is differentiable; sharp discontinuities must be intentionally designed, not accidental
3. **Multi-Scale Coherence**: Fields have meaningful structure at multiple spatial scales — macro-scale regional character, meso-scale local variation, micro-scale fine texture — encoded via layered octaves
4. **Field Orthogonality**: Independent field types (mana, disease, faction_A, faction_B) are computed from independent seeds and can be queried and combined without cross-contamination
5. **Determinism**: Same spatial coordinate → same field value across all machines and execution orders

## Core Pattern

```python
import struct
import math
import xxhash

def position_hash(x, y, z, salt=0):
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqq', x, y, z))
    return h.intdigest()

def hash_to_float(h):
    return (h & 0xFFFFFFFF) / 0x100000000

def coherent_value(x, y, seed, octaves=4):
    """ZeroBytes coherent noise — the foundation of Zero-Field."""
    value, amp, freq, max_amp = 0.0, 1.0, 1.0, 0.0
    for i in range(octaves):
        x0, y0 = int(x * freq), int(y * freq)
        sx = (x * freq) % 1; sx = sx * sx * (3 - 2 * sx)
        sy = (y * freq) % 1; sy = sy * sy * (3 - 2 * sy)
        n00 = hash_to_float(position_hash(x0,   y0,   0, seed+i)) * 2 - 1
        n10 = hash_to_float(position_hash(x0+1, y0,   0, seed+i)) * 2 - 1
        n01 = hash_to_float(position_hash(x0,   y0+1, 0, seed+i)) * 2 - 1
        n11 = hash_to_float(position_hash(x0+1, y0+1, 0, seed+i)) * 2 - 1
        nx0 = n00*(1-sx) + n10*sx
        nx1 = n01*(1-sx) + n11*sx
        value += amp * (nx0*(1-sy) + nx1*sy)
        max_amp += amp; amp *= 0.5; freq *= 2.0
    return value / max_amp

def scalar_field(x, y, world_seed, field_salt,
                 macro_freq=0.002, meso_freq=0.02, micro_freq=0.1,
                 macro_weight=0.6, meso_weight=0.3, micro_weight=0.1,
                 octaves=4):
    """
    General-purpose continuous scalar field.
    Returns a value in approximately -1.0..1.0.
    field_salt: unique integer per field type — ensures orthogonality.
    """
    macro = coherent_value(x * macro_freq, y * macro_freq,
                           world_seed + field_salt,     octaves=octaves)
    meso  = coherent_value(x * meso_freq,  y * meso_freq,
                           world_seed + field_salt + 1, octaves=max(1, octaves-2))
    micro = coherent_value(x * micro_freq, y * micro_freq,
                           world_seed + field_salt + 2, octaves=1)
    return macro * macro_weight + meso * meso_weight + micro * micro_weight

def vector_field(x, y, world_seed, field_salt):
    """
    Continuous 2D vector field — direction and magnitude at any point.
    Used for wind, ocean currents, magical flow, migration pressure.
    """
    dx = scalar_field(x, y, world_seed, field_salt,
                      macro_freq=0.003, meso_freq=0.03)
    dy = scalar_field(x, y, world_seed, field_salt + 10000,
                      macro_freq=0.003, meso_freq=0.03)
    magnitude = (dx**2 + dy**2) ** 0.5
    if magnitude < 1e-6:
        return {"dx": 0.0, "dy": 0.0, "magnitude": 0.0, "direction": 0.0}
    return {
        "dx":        dx / magnitude,
        "dy":        dy / magnitude,
        "magnitude": magnitude,
        "direction": math.atan2(dy, dx)
    }
```

## Field Topology: Finding Peaks and Boundaries

The power of Zero-Field emerges from its topology — where fields are strongest (entity spawn zones), weakest (empty zones), or in equilibrium (territorial borders).

```python
def field_local_maximum(x, y, world_seed, field_salt, search_radius=5, step=1):
    """
    Is this point a local maximum of the field?
    Used to determine if an entity should spawn here.
    Checks cardinal and diagonal neighbours — no global search required.
    """
    centre = scalar_field(x, y, world_seed, field_salt)
    for dx in range(-search_radius, search_radius+1, step):
        for dy in range(-search_radius, search_radius+1, step):
            if dx == 0 and dy == 0: continue
            if scalar_field(x+dx, y+dy, world_seed, field_salt) > centre:
                return False
    return True

def field_gradient(x, y, world_seed, field_salt, epsilon=1.0):
    """
    The gradient (slope direction) of the field at this point.
    Useful for entity movement: agents move up-gradient to find field peaks.
    """
    fx  = scalar_field(x,         y,         world_seed, field_salt)
    fxp = scalar_field(x+epsilon, y,         world_seed, field_salt)
    fyp = scalar_field(x,         y+epsilon, world_seed, field_salt)
    gx = (fxp - fx) / epsilon
    gy = (fyp - fx) / epsilon
    return {"gx": gx, "gy": gy, "magnitude": (gx**2 + gy**2)**0.5}

def territorial_border(x, y, world_seed, faction_a_salt, faction_b_salt, threshold=0.05):
    """
    Is this point on the border between two faction fields?
    True when the two fields are within threshold of each other.
    """
    field_a = scalar_field(x, y, world_seed, faction_a_salt)
    field_b = scalar_field(x, y, world_seed, faction_b_salt)
    return abs(field_a - field_b) < threshold
```

## Quick Recipes

### Mana / Magic Intensity Field
```python
MANA_FIELD_SALT   = 1000
LEYLINE_FIELD_SALT = 1001

def mana_field(x, y, world_seed):
    """
    Continuous mana density at any world point.
    High mana → magical entities spawn; low mana → mundane terrain.
    """
    # Macro: continental mana wells (large, smooth)
    macro = coherent_value(x*0.001, y*0.001, world_seed + MANA_FIELD_SALT, octaves=4)
    # Meso: regional mana veins (medium scale)
    meso  = coherent_value(x*0.01,  y*0.01,  world_seed + MANA_FIELD_SALT + 1, octaves=3)
    # Micro: local mana sparks (fine detail)
    micro = coherent_value(x*0.05,  y*0.05,  world_seed + MANA_FIELD_SALT + 2, octaves=2)

    raw = macro*0.5 + meso*0.35 + micro*0.15
    # Ley line overlay: sharp intensity bands
    ley  = coherent_value(x*0.005, y*0.005, world_seed + LEYLINE_FIELD_SALT, octaves=2)
    leyline_boost = max(0.0, (abs(ley) - 0.7) * 3.0)  # spikes near leylines

    intensity = max(0.0, raw + leyline_boost)
    return {
        "intensity":       intensity,
        "is_leyline":      leyline_boost > 0.2,
        "spawn_magical":   intensity > 0.6,
        "spawn_tier":      int(intensity * 5)  # 0 = mundane, 4 = arcane elite
    }
```

### Disease / Pollution Spread (No Simulation)
```python
DISEASE_FIELD_SALT = 2000

def disease_field(x, y, world_seed, outbreak_epoch=0):
    """
    Disease concentration at any point — no simulation of spread required.
    The spatial distribution is seeded directly; it does not propagate from a source.
    Compose with Zero-Temporal for time-varying outbreaks.
    """
    # Base geographic susceptibility (static ZeroBytes layer)
    geo_susceptibility = coherent_value(x*0.008, y*0.008, world_seed + DISEASE_FIELD_SALT, octaves=3)

    # Outbreak character: seeded from epoch (Zero-Temporal composure)
    from zero_temporal import temporal_hash
    outbreak_seed  = temporal_hash(x//200, y//200, 0, outbreak_epoch, world_seed + DISEASE_FIELD_SALT)
    outbreak_local = hash_to_float(outbreak_seed)

    # Density: product of susceptibility and current outbreak intensity
    density = max(0.0, geo_susceptibility) * outbreak_local

    return {
        "density":          density,
        "is_hotspot":       density > 0.7,
        "containment_risk": hash_to_float(outbreak_seed >> 16) > 0.8,
        "strain_virulence": hash_to_float(outbreak_seed >> 32)
    }
```

### Faction Territory (Continuous, No Tile Ownership)
```python
def faction_control_field(x, y, world_seed, faction_id, n_factions=5):
    """
    This faction's field strength at this point.
    Territory = where this faction's field exceeds all other factions' fields.
    """
    faction_salt = 3000 + faction_id * 100
    return scalar_field(x, y, world_seed, faction_salt,
                        macro_freq=0.003, meso_freq=0.015, micro_freq=0.05,
                        macro_weight=0.7, meso_weight=0.25, micro_weight=0.05)

def territorial_ownership(x, y, world_seed, n_factions=5):
    """
    Which faction controls this point, and by what margin?
    No tile-ownership table. No stored borders. Computed in O(n_factions).
    """
    fields = [(i, faction_control_field(x, y, world_seed, i, n_factions))
              for i in range(n_factions)]
    fields.sort(key=lambda f: f[1], reverse=True)
    dominant_faction, dominant_strength = fields[0]
    second_strength = fields[1][1] if len(fields) > 1 else -1.0

    return {
        "owner":       dominant_faction,
        "strength":    dominant_strength,
        "margin":      dominant_strength - second_strength,
        "is_frontier": abs(dominant_strength - second_strength) < 0.1,
        "contested":   abs(dominant_strength - second_strength) < 0.05
    }
```

### Wind / Ocean Current Vector Field
```python
WIND_FIELD_SALT    = 4000
CURRENT_FIELD_SALT = 4001

def wind_at(x, y, world_seed):
    """
    Deterministic wind vector at any world point.
    No fluid simulation. No stored pressure maps.
    """
    wind = vector_field(x, y, world_seed, WIND_FIELD_SALT)
    # Wind speed: separate scalar field
    speed_base  = scalar_field(x, y, world_seed, WIND_FIELD_SALT + 50,
                               macro_freq=0.004, meso_freq=0.04)
    wind_speed  = max(0.0, speed_base + 0.5) * 100  # 0–100 km/h

    return {
        "direction":  wind["direction"],
        "speed_kmh":  wind_speed,
        "is_gale":    wind_speed > 75,
        "vector":     (wind["dx"] * wind_speed, wind["dy"] * wind_speed)
    }

def ocean_current(x, y, world_seed):
    """Deterministic ocean current — temperature and flow direction."""
    current = vector_field(x, y, world_seed, CURRENT_FIELD_SALT)
    temp    = scalar_field(x, y, world_seed, CURRENT_FIELD_SALT + 100,
                           macro_freq=0.002, meso_freq=0.01)
    return {
        "direction":   current["direction"],
        "temperature": temp,  # -1.0 = arctic, +1.0 = tropical
        "is_warm":     temp > 0.2,
        "intensity":   current["magnitude"]
    }
```

### Predator Pressure / Ecosystem Field
```python
PREDATOR_FIELD_SALT = 5000
PREY_FIELD_SALT     = 5001
VEGETATION_FIELD_SALT = 5002

def ecosystem_field(x, y, world_seed):
    """
    Ecosystem pressure at any point — predator density, prey availability, vegetation.
    No entity population simulation. Field-first: entities spawn at field peaks.
    """
    vegetation = scalar_field(x, y, world_seed, VEGETATION_FIELD_SALT,
                              macro_freq=0.005, meso_freq=0.02)
    # Prey abundance tracks vegetation (with offset)
    prey = scalar_field(x, y, world_seed, PREY_FIELD_SALT,
                        macro_freq=0.006, meso_freq=0.025)
    prey_modulated = max(0.0, prey) * max(0.0, vegetation + 0.3)  # prey needs vegetation

    # Predator density tracks prey (with larger-scale structure)
    predator = scalar_field(x, y, world_seed, PREDATOR_FIELD_SALT,
                            macro_freq=0.003, meso_freq=0.01)
    predator_modulated = max(0.0, predator) * max(0.0, prey_modulated + 0.2)

    return {
        "vegetation":      max(0.0, vegetation),
        "prey_density":    prey_modulated,
        "predator_density": predator_modulated,
        "spawn_herbivore": prey_modulated > 0.5,
        "spawn_predator":  predator_modulated > 0.6,
        "danger_level":    predator_modulated
    }
```

### Radiation / Heat Zone
```python
HEAT_FIELD_SALT = 6000

def heat_field(x, y, z, world_seed):
    """
    Three-dimensional heat/radiation field — useful for caves, space, volcanic terrain.
    z-axis: depth or altitude.
    """
    # Base geothermal layer: increases with depth
    geothermal = max(0.0, -z * 0.01)  # increases as z becomes more negative

    # Surface variation
    surface = scalar_field(x, y, world_seed, HEAT_FIELD_SALT,
                           macro_freq=0.004, meso_freq=0.04)
    surface_heat = max(0.0, surface)

    # Volcanic hotspot layer (sharp, localised peaks)
    volcanic = coherent_value(x*0.02, y*0.02, world_seed + HEAT_FIELD_SALT + 1, octaves=2)
    hotspot  = max(0.0, (volcanic - 0.5) * 4.0)  # spikes above 0.5 threshold

    total_heat = geothermal + surface_heat * 0.4 + hotspot
    return {
        "temperature":   total_heat,
        "is_hazardous":  total_heat > 1.5,
        "lava_present":  hotspot > 2.0,
        "heat_damage":   max(0.0, total_heat - 1.0) * 10  # DPS above threshold
    }
```

## Composing with ZeroBytes and Zero-Quadratic

| Layer | Type | Answers |
|-------|------|---------|
| Zero-Field | Continuous | What is the field strength at this point? |
| ZeroBytes | Discrete point | What entity is at this position? |
| Zero-Quadratic | Discrete pair | What is the relationship between two entities? |
| Composed | Full world model | What is here, what does it relate to, what field does it sit in? |

```python
def full_world_point(x, y, world_seed):
    """
    Complete world model at a point: field + entity + relationships.
    """
    # Zero-Field: continuous properties
    mana    = mana_field(x, y, world_seed)
    faction = territorial_ownership(x, y, world_seed, n_factions=5)
    wind    = wind_at(x, y, world_seed)

    # ZeroBytes: discrete entity at this position
    from zerobytes import tile
    terrain = tile(x, y, world_seed)

    # Entity spawning derived from field peaks (field-first design)
    entity = None
    if mana["spawn_magical"] and field_local_maximum(x, y, world_seed, MANA_FIELD_SALT, search_radius=3):
        entity = {"type": "magical_creature", "tier": mana["spawn_tier"]}

    return {
        "terrain":  terrain,
        "mana":     mana,
        "faction":  faction,
        "wind":     wind,
        "entity":   entity
    }
```

## Anti-Patterns

```python
# BAD: Entity-first field computation — iterates all entities to compute field at a point
def bad_mana(x, y, all_magical_entities):
    return sum(entity["power"] / dist(x,y, entity["x"], entity["y"])
               for entity in all_magical_entities)  # O(N) per query, entities must be stored

# BAD: Storing the field — defeats the zero-bytes principle
mana_map = [[mana_field(x, y, seed) for x in range(1000)] for y in range(1000)]  # 1M entries

# BAD: Treating field and entity as the same abstraction
def bad_territory(x, y):
    return nearest_entity_owner(x, y, all_entities)  # requires stored entity set

# BAD: Overlapping field salts — fields contaminate each other
mana_field    = scalar_field(x, y, seed, salt=0)
disease_field = scalar_field(x, y, seed, salt=0)  # same salt = same field!

# GOOD: Orthogonal salts, computed on demand
mana    = scalar_field(x, y, world_seed, MANA_FIELD_SALT)
disease = scalar_field(x, y, world_seed, DISEASE_FIELD_SALT)

# GOOD: Entities spawned at field peaks, not the other way round
if field_local_maximum(x, y, world_seed, MANA_FIELD_SALT):
    spawn_entity(x, y, "magical_creature")
```

## Debugging Checklist

When field values differ across machines:
1. Check struct pack format — `'<qqq'` (little-endian 64-bit) must be consistent
2. Check frequency parameters — `float` arithmetic in `coherent_value` must use the same precision; avoid platform-specific fast-math flags
3. Check octave count — different octave counts produce different values; always use the same count per field type

When field has visible tile boundaries:
- Reduce `macro_freq` to smooth out regional transitions
- Increase octave count to add intermediate-scale coherence
- Use `blended_spectral_value` from Zero-Spectral for boundary blending if still needed

When field peaks are too rare or too common:
- Adjust the `macro_weight`/`meso_weight` balance — macro-heavy = large sparse peaks; meso-heavy = many small peaks
- For spawn threshold tuning: plot field value histogram, set threshold at desired percentile

When two independent fields appear correlated:
- Check salt values — they must differ by at least `octaves + 3` to avoid hash collision artefacts
- Use widely separated salts: 1000, 2000, 3000 — not 1, 2, 3

## Determinism Verification

```python
def verify_field(world_seed, test_positions, field_salt=MANA_FIELD_SALT):
    """Verify scalar field determinism and continuity."""
    # Determinism
    for (x, y) in test_positions:
        v1 = scalar_field(x, y, world_seed, field_salt)
        v2 = scalar_field(x, y, world_seed, field_salt)
        assert abs(v1 - v2) < 1e-9, f"Non-deterministic at ({x},{y})"

    # Continuity: adjacent points must not wildly diverge (smoothness check)
    for (x, y) in test_positions:
        v0 = scalar_field(x, y,   world_seed, field_salt)
        vx = scalar_field(x+1, y, world_seed, field_salt)
        vy = scalar_field(x, y+1, world_seed, field_salt)
        assert abs(v0 - vx) < 0.5, f"Discontinuity in X at ({x},{y}): {abs(v0-vx)}"
        assert abs(v0 - vy) < 0.5, f"Discontinuity in Y at ({x},{y}): {abs(v0-vy)}"

    # Orthogonality: two fields with different salts should not be correlated
    other_salt = field_salt + 1000
    vals_a = [scalar_field(x, y, world_seed, field_salt) for x, y in test_positions]
    vals_b = [scalar_field(x, y, world_seed, other_salt) for x, y in test_positions]
    mean_a = sum(vals_a) / len(vals_a)
    mean_b = sum(vals_b) / len(vals_b)
    correlation = sum((a-mean_a)*(b-mean_b) for a,b in zip(vals_a,vals_b))
    assert abs(correlation) < len(test_positions) * 0.3, "Fields are correlated — check salts!"

    print(f"Zero-Field verification passed: {len(test_positions)} positions.")
```

## Usage

When implementing a Zero-Field system:

1. **Identify the field type** — what continuous influence does this model? (mana, territory, pressure, heat); give it a unique salt constant
2. **Choose frequency parameters** — `macro_freq` for large-scale structure, `meso_freq` for regional variation; start with `macro_freq = 0.002–0.005`
3. **Choose weight balance** — `macro_weight = 0.6` for smooth fields; increase `meso_weight` for more local variation
4. **Design peaks** — if entities spawn at field peaks, tune `field_local_maximum` search radius to control spawn density
5. **Design boundaries** — if field represents territory, use `territorial_border` to detect frontier zones and `territorial_ownership` to assign control
6. **Ensure orthogonality** — separate each field type's salt by at least 1000 to prevent correlation
7. **Compose with ZeroBytes** — field sets the ambient context; ZeroBytes generates discrete entities within that context
8. **Verify** — run `verify_field` confirming determinism, continuity, and orthogonality

**Core principle:** Space is not a container for entities — space has properties of its own. Zero-Field generates those properties directly from coordinates, without reference to any entity set. Entities are optional: they are peaks, not sources. Zero bytes describe the field. The field describes the world.
