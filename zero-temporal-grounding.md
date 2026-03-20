---
name: zero-temporal
description: Coordinate+epoch-is-seed O(1) temporal procedural determinism methodology. Extends ZeroBytes from static spatial generation into deterministic world-time. Use when a developer asks for procedural weather systems, seasonal terrain changes, deterministic NPC schedules, historical archaeology (querying past world states), economic boom-bust cycles, tidal systems, day-night cycles, or any system where a spatial property changes over world-time and must be reconstructable at any past or future epoch without simulation or stored logs. Triggers on phrases like "deterministic weather", "seasonal changes", "procedural history", "time as a coordinate", "world epoch", "NPC schedule", "historical state", "zero-temporal", "time-based procedural", or when a ZeroBytes system needs properties that evolve over simulated time.
---

# Zero-Temporal: Coordinate+Epoch-is-Seed Temporal Procedural Determinism

Extends [ZeroBytes](zerobytes/SKILL.md) O(1) position hashing by adding world-time as a fourth coordinate dimension. The position+epoch pair IS the seed. Any world state at any moment — past, present, or future — is directly computable without simulation, logs, or stored history.

## The Conceptual Leap from ZeroBytes

ZeroBytes asks: *What is at this position?*
Zero-Temporal asks: *What is at this position at this moment in world-time?*

ZeroBytes is stateless across time — the terrain is always the terrain, the biome never changes. This is appropriate for properties that are geologically stable. But weather, seasons, economic cycles, NPC schedules, dynastic rise and fall — these are intrinsically temporal. Zero-Temporal treats time as simply another coordinate axis, preserving all five ZeroBytes laws while adding the capacity to generate different values at the same position for different epochs.

**The key philosophical move:** Time is not a log of what happened. Time is a coordinate. Query it like a coordinate.

## The Five Extended Laws

Every temporal procedural system must satisfy:

1. **O(1) Temporal Access**: Compute any world-state at any epoch without iterating through preceding epochs — jump directly to epoch N as easily as epoch 1
2. **World-Time Isolation**: `epoch` must be a world-defined integer tick, never wall-clock time — determinism requires that the same world-time produces the same result regardless of when the query is made in real time
3. **Temporal Coherence**: Adjacent epochs produce related values; distant epochs may diverge — encode cycles (seasons, tides, economic rhythms) as periodic functions of epoch, not as accumulated state
4. **Spatial-Temporal Hierarchy**: Temporal granularity varies by scale — large regions change slowly (climate), small regions change quickly (weather); child temporal seeds inherit parent temporal context
5. **Determinism**: Same position + same epoch → same output across all machines and execution orders

## Core Pattern

```python
import struct
import math
import xxhash

def temporal_hash(x, y, z, epoch, salt=0):
    """
    epoch is a world-defined integer — one tick per in-game hour, day, or season.
    NEVER pass real-time (time.time()) as epoch — this destroys determinism.
    """
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqqq', x, y, z, epoch))
    return h.intdigest()

def hash_to_float(h):
    return (h & 0xFFFFFFFF) / 0x100000000

def position_hash(x, y, z, salt=0):
    """ZeroBytes base — for static spatial properties."""
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqq', x, y, z))
    return h.intdigest()

def coherent_value(x, y, seed, octaves=4):
    """ZeroBytes coherent noise — for regionally smooth spatial properties."""
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
```

## Epoch Design Patterns

### Choosing Temporal Granularity

The epoch tick size is a world design decision. Choose it at system startup and never change it.

| Tick Size | Epochs per Year | Use Case |
|-----------|----------------|----------|
| 1 tick = 1 real hour | 8,760 | Detailed NPC schedules, tidal cycles |
| 1 tick = 1 in-game day | 365 | Weather, crop cycles, patrol routes |
| 1 tick = 1 in-game season | 4 | Biome state, economic quarters |
| 1 tick = 1 in-game year | 1 | Dynastic history, tectonic drift |

### Cyclic vs Stochastic Temporal Properties

```python
# CYCLIC: Deterministic periodic function of epoch (no hash needed)
def day_phase(epoch, ticks_per_day=24):
    """Returns 0.0 (midnight) to 1.0 (midnight) — pure cycle."""
    return (epoch % ticks_per_day) / ticks_per_day

def seasonal_temperature_offset(epoch, ticks_per_year=365):
    """Smooth sinusoidal seasonal cycle."""
    phase = (epoch % ticks_per_year) / ticks_per_year
    return math.sin(phase * 2 * math.pi)  # +1.0 = peak summer, -1.0 = peak winter

# STOCHASTIC: Hash-based temporal randomness (deterministic but aperiodic)
def storm_event(x, y, epoch, world_seed, ticks_per_week=7):
    """Is there a storm at this location this week? Deterministic but aperiodic."""
    week = epoch // ticks_per_week
    storm_seed = temporal_hash(x // 32, y // 32, 0, week, world_seed + 100)
    return hash_to_float(storm_seed) > 0.85

# COMPOSED: Cyclic baseline + stochastic variation
def temperature(x, y, epoch, world_seed, ticks_per_year=365):
    base_climate    = coherent_value(x*0.005, y*0.005, world_seed)       # static ZeroBytes
    seasonal_offset = 0.3 * seasonal_temperature_offset(epoch, ticks_per_year)
    noise_seed      = temporal_hash(x//8, y//8, 0, epoch, world_seed + 1)
    daily_noise     = (hash_to_float(noise_seed) - 0.5) * 0.1
    return base_climate + seasonal_offset + daily_noise
```

## Quick Recipes

### Weather System
```python
def weather(x, y, epoch, world_seed, ticks_per_day=24, ticks_per_year=365):
    """Full weather state at any position, any epoch. O(1)."""
    # Static base climate (ZeroBytes layer — never changes)
    base_temp     = coherent_value(x*0.005, y*0.005, world_seed)
    base_moisture = coherent_value(x*0.005, y*0.005, world_seed + 500)

    # Seasonal cycle (pure periodic — no hash)
    season_phase  = seasonal_temperature_offset(epoch, ticks_per_year)

    # Weekly storm pattern (temporal hash — aperiodic)
    week          = epoch // 7
    storm_seed    = temporal_hash(x//32, y//32, 0, week, world_seed + 100)
    storm_chance  = hash_to_float(storm_seed)

    # Daily variation (temporal hash at day granularity)
    day           = epoch // ticks_per_day
    daily_seed    = temporal_hash(x//8, y//8, 0, day, world_seed + 200)
    cloud_cover   = hash_to_float(daily_seed)

    temp = base_temp + 0.3 * season_phase + 0.05 * (hash_to_float(daily_seed >> 16) - 0.5)

    return {
        "temperature":  temp,
        "cloud_cover":  cloud_cover,
        "storm_active": storm_chance > 0.85 and base_moisture > 0.4,
        "precipitation": max(0.0, cloud_cover - 0.5) * base_moisture
    }
```

### Seasonal Terrain State
```python
def terrain_state(x, y, epoch, world_seed, ticks_per_year=365):
    """What does this tile look like right now? Static base + temporal overlay."""
    # Static properties (ZeroBytes)
    elevation = coherent_value(x*0.02, y*0.02, world_seed)
    moisture  = coherent_value(x*0.02, y*0.02, world_seed + 1000)

    # Seasonal overlay
    season_t  = seasonal_temperature_offset(epoch, ticks_per_year)
    is_winter = season_t < -0.5

    # Derive current terrain type
    if elevation < -0.2:
        biome = "frozen_sea" if is_winter else "ocean"
    elif elevation < 0.3:
        if is_winter and moisture > 0.2:
            biome = "snow_plains"
        elif moisture > 0.3:
            biome = "forest"
        else:
            biome = "plains"
    else:
        biome = "snow_mountain" if is_winter else "mountain"

    # Flood plains: temporarily inundated during wet season
    wet_season = season_t > 0.3
    if elevation < -0.1 and wet_season and moisture > 0.6:
        biome = "flood_plain"

    return {"biome": biome, "elevation": elevation, "moisture": moisture}
```

### NPC Schedule
```python
def npc_schedule(npc_id, epoch, world_seed, ticks_per_day=24):
    """Where is this NPC and what are they doing at this epoch? O(1)."""
    time_of_day = epoch % ticks_per_day
    day_index   = epoch // ticks_per_day

    # NPC's base home location (ZeroBytes — static)
    home_seed = position_hash(npc_id, 0, 0, world_seed)
    home_x    = int(hash_to_float(home_seed) * 100)
    home_y    = int(hash_to_float(home_seed >> 16) * 100)

    # Day-specific destination (changes day by day, deterministically)
    day_seed     = temporal_hash(npc_id, 0, 0, day_index, world_seed + 300)
    dest_x       = int(hash_to_float(day_seed) * 100)
    dest_y       = int(hash_to_float(day_seed >> 16) * 100)
    daily_errand = ["market","temple","tavern","fields","workshop"][int(hash_to_float(day_seed >> 32) * 5)]

    # Determine current activity by time-of-day
    if time_of_day < 6:
        return {"location": (home_x, home_y), "activity": "sleeping"}
    elif time_of_day < 8:
        return {"location": (home_x, home_y), "activity": "morning_routine"}
    elif time_of_day < 17:
        return {"location": (dest_x, dest_y), "activity": daily_errand}
    elif time_of_day < 20:
        return {"location": (dest_x, dest_y), "activity": "returning"}
    else:
        return {"location": (home_x, home_y), "activity": "evening"}
```

### Historical Archaeology
```python
def historical_state(x, y, past_epoch, world_seed):
    """What was this tile like at a past epoch? Query it directly — no replay needed."""
    # Zero-Temporal: the past is just a different epoch coordinate
    return terrain_state(x, y, past_epoch, world_seed)

def civilisation_era(city_x, city_y, epoch, world_seed, founding_epoch=0):
    """What era is this city in at this epoch?"""
    age = epoch - founding_epoch
    if age < 0:
        return {"era": "pre-founding", "population": 0}

    # Economic cycle: boom-bust with ~200 epoch period
    economic_phase = math.sin(age * (2 * math.pi / 200))

    # Stability: stochastic events at decade scale
    decade = age // 10
    stability_seed = temporal_hash(city_x, city_y, 0, decade, world_seed + 400)
    stability = hash_to_float(stability_seed)

    population_base = coherent_value(city_x*0.01, city_y*0.01, world_seed) * 10000
    population = max(0, int(population_base * (1 + 0.3*economic_phase) * stability))

    return {
        "era":        ["founding","growth","peak","decline","recovery"][min(4, age // 100)],
        "population": population,
        "prosperity": 0.5 + 0.3*economic_phase,
        "stability":  stability
    }
```

### Economic Boom-Bust Cycle
```python
def market_price(commodity_id, market_x, market_y, epoch, world_seed, cycle_length=200):
    """Deterministic commodity price at any market at any epoch."""
    # Base value: ZeroBytes static property of the commodity+location
    base_value_seed = position_hash(commodity_id, market_x // 10, market_y // 10, world_seed)
    base_value = 10 + hash_to_float(base_value_seed) * 90  # 10–100

    # Regional economic cycle (slow, coherent)
    cycle_phase = math.sin((epoch / cycle_length) * 2 * math.pi)

    # Monthly volatility (stochastic)
    month = epoch // 30
    vol_seed = temporal_hash(commodity_id, market_x // 20, market_y // 20, month, world_seed + 500)
    volatility = (hash_to_float(vol_seed) - 0.5) * 0.4

    return base_value * (1 + 0.3 * cycle_phase + volatility)
```

## Hierarchy Pattern

ZeroBytes hierarchy: `parent_seed → child_seed via position`
Zero-Temporal adds: `parent_temporal_seed → child_temporal_seed via position+epoch`

```python
def regional_epoch_seed(region_x, region_y, epoch, world_seed, region_scale=32):
    """A region's temporal seed: changes slowly — chunked by region and epoch-block."""
    epoch_block = epoch // region_scale           # regions update every N ticks
    return temporal_hash(region_x // region_scale,
                         region_y // region_scale,
                         0, epoch_block, world_seed)

def local_epoch_seed(x, y, epoch, region_seed, local_scale=8):
    """Local temporal seed inherits regional context."""
    local_epoch_block = epoch // local_scale
    return temporal_hash(x // local_scale, y // local_scale,
                         region_seed & 0xFFFF, local_epoch_block, 0)
```

**Practical consequence:** A region experiencing a "drought epoch" generates statistically drier local tiles and more depleted market prices — through seed inheritance, not stored drought state.

## Composing with ZeroBytes and Zero-Quadratic

| Layer | Temporal? | Answers | Example |
|-------|-----------|---------|---------|
| ZeroBytes | Static | What is permanently *at* this position? | Elevation, ore veins |
| Zero-Temporal | Dynamic | What is at this position *right now*? | Weather, seasonal biome |
| Zero-Quadratic | Static relational | What is the permanent *relationship*? | Cultural distance |
| Zero-Quadratic + Temporal | Dynamic relational | How does the relationship *feel today*? | Current trade pressure |

```python
def current_trade_pressure(port_a, port_b, epoch, world_seed):
    """Trade pressure between two ports at a specific epoch."""
    # Static baseline (Zero-Quadratic)
    base_viability = trade_viability(port_a, port_b, world_seed)

    # Current economic states (Zero-Temporal)
    price_a = market_price(1, *port_a, epoch, world_seed)
    price_b = market_price(1, *port_b, epoch, world_seed)
    price_gradient = abs(price_a - price_b) / max(price_a, price_b)

    return base_viability * price_gradient
```

## Anti-Patterns

```python
# BAD: Wall-clock time as epoch — breaks determinism across machines and sessions
epoch = int(time.time())

# BAD: Accumulated state — this is simulation, not Zero-Temporal
world_state[epoch] = simulate(world_state[epoch-1])

# BAD: Iterating to reach epoch N — defeats O(1) access
state = initial_state
for tick in range(target_epoch): state = update(state)

# BAD: Storing computed temporal values — defeats zero-bytes principle
weather_cache[(x, y, epoch)] = compute_weather(x, y, epoch)

# GOOD: Direct epoch coordinate query
w = weather(x, y, current_world_epoch, world_seed)

# GOOD: Historical query with no replay
past = terrain_state(x, y, epoch_500_years_ago, world_seed)
```

## Debugging Checklist

When temporal properties differ across machines or sessions:
1. **Check epoch source** — is `epoch` coming from a world tick counter or from `time.time()`? Must be world tick
2. **Check struct pack format** — `'<qqqq'` (little-endian 64-bit) must be consistent; `int` must fit in signed 64-bit
3. **Check cyclic function precision** — `math.sin` with float epoch can accumulate drift at large epoch values; use integer arithmetic for cycle phase

When temporal properties "don't feel like they change over time":
- Ensure temporal hash uses epoch as a *significant* input — chunking too coarsely makes changes imperceptible
- Add a periodic function layer (sin/cos) for smooth cycles; hash alone is aperiodic and will feel random rather than rhythmic

When NPC schedules feel incoherent:
- Ensure time-of-day is `epoch % ticks_per_day` (cyclic), not a temporal hash
- Reserve temporal hashing for day-to-day variation, not hour-to-hour

## Determinism Verification

```python
def verify_temporal(temporal_fn, world_seed, position_epoch_pairs):
    """Verify O(1) access and determinism at arbitrary epochs."""
    # Determinism
    for (x, y, epoch) in position_epoch_pairs:
        v1 = temporal_fn(x, y, epoch, world_seed)
        v2 = temporal_fn(x, y, epoch, world_seed)
        assert v1 == v2, f"Non-deterministic at ({x},{y}) epoch={epoch}"

    # O(1) access: epoch 10000 must match epoch 10000 regardless of whether
    # epoch 9999 was queried first
    x, y = position_epoch_pairs[0][:2]
    far_epoch = 10000
    direct   = temporal_fn(x, y, far_epoch, world_seed)
    after    = temporal_fn(x, y, far_epoch, world_seed)  # no state between calls
    assert direct == after, "Epoch access is order-dependent!"

    # Past queryability: historical epoch produces same result on re-query
    past_epoch = 0
    hist_1 = temporal_fn(x, y, past_epoch, world_seed)
    hist_2 = temporal_fn(x, y, past_epoch, world_seed)
    assert hist_1 == hist_2, "Historical epoch is non-deterministic!"

    print(f"Zero-Temporal verification passed: {len(position_epoch_pairs)} position-epoch pairs.")
```

## Usage

When implementing a Zero-Temporal system:

1. **Define the epoch** — choose tick size (hours, days, seasons) at world design time; never change it
2. **Separate static from dynamic** — ZeroBytes handles permanent properties; Zero-Temporal handles time-varying ones
3. **Choose cyclic vs stochastic** — periodic properties (seasons, tides) use `sin`/`cos` of epoch; aperiodic events use `temporal_hash`
4. **Design temporal granularity** — slow-changing properties hash at coarse epoch chunks; fast-changing ones hash at fine epoch resolution
5. **Compose layers** — static base (ZeroBytes) + seasonal cycle (periodic function) + stochastic events (temporal hash)
6. **Enable historical queries** — verify that past epochs remain queryable and stable
7. **Verify** — run `verify_temporal` confirming O(1) access at arbitrary epochs

**Core principle:** The past is not a log. The future is not a simulation. Both are coordinates. Zero-Temporal queries them like any other dimension — in O(1), with zero bytes stored.
