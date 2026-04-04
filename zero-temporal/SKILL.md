---
name: zero-temporal
description: Coordinate+epoch-is-seed O(1) temporal procedural determinism methodology. Extends ZeroBytes from static spatial generation into deterministic world-time. Use when a developer asks for procedural weather systems, seasonal terrain changes, deterministic NPC schedules, historical archaeology (querying past world states), economic boom-bust cycles, tidal systems, day-night cycles, or any system where a spatial property changes over world-time and must be reconstructable at any past or future epoch without simulation or stored logs. Triggers on phrases like "deterministic weather", "seasonal changes", "procedural history", "time as a coordinate", "world epoch", "NPC schedule", "historical state", "zero-temporal", "time-based procedural", or when a ZeroBytes system needs properties that evolve over simulated time.
---

# Zero-Temporal: Coordinate+Epoch-is-Seed Temporal Procedural Determinism

Extends [ZeroBytes](../zerobytes/SKILL.md) O(1) position hashing by adding world-time as a fourth coordinate dimension. The position+epoch IS the seed. Any world state at any moment — past, present, or future — is directly computable without simulation, logs, or stored history.

## The Five Extended Laws

Every temporal procedural system must satisfy:

1. **O(1) Temporal Access**: Compute any world-state at any epoch without iterating preceding epochs — jump directly to epoch N as easily as epoch 1
2. **World-Time Isolation**: `epoch` must be a world-defined integer tick, never wall-clock time — same world-time must produce same result regardless of when the query is made in real time
3. **Temporal Coherence**: Encode cycles (seasons, tides, rhythms) as periodic functions of epoch, not accumulated state; adjacent epochs produce related values
4. **Spatial-Temporal Hierarchy**: Large regions change slowly (climate), small regions change quickly (weather); child temporal seeds inherit parent temporal context
5. **Determinism**: Same position + same epoch → same output across all machines and execution orders

## Core Pattern

```python
import struct, math
import xxhash

def temporal_hash(x, y, z, epoch, salt=0):
    """epoch is a world-defined integer tick. NEVER pass time.time() — destroys determinism."""
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqqq', x, y, z, epoch))
    return h.intdigest()

def position_hash(x, y, z, salt=0):
    """ZeroBytes base — for static spatial properties."""
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqq', x, y, z))
    return h.intdigest()

def hash_to_float(h):
    return (h & 0xFFFFFFFF) / 0x100000000

def coherent_value(x, y, seed, octaves=4):
    value, amp, freq, max_amp = 0.0, 1.0, 1.0, 0.0
    for i in range(octaves):
        x0, y0 = int(x * freq), int(y * freq)
        sx = (x * freq) % 1; sx = sx*sx*(3-2*sx)
        sy = (y * freq) % 1; sy = sy*sy*(3-2*sy)
        n00 = hash_to_float(position_hash(x0,   y0,   0, seed+i)) * 2 - 1
        n10 = hash_to_float(position_hash(x0+1, y0,   0, seed+i)) * 2 - 1
        n01 = hash_to_float(position_hash(x0,   y0+1, 0, seed+i)) * 2 - 1
        n11 = hash_to_float(position_hash(x0+1, y0+1, 0, seed+i)) * 2 - 1
        nx0 = n00*(1-sx)+n10*sx; nx1 = n01*(1-sx)+n11*sx
        value += amp*(nx0*(1-sy)+nx1*sy); max_amp += amp; amp *= 0.5; freq *= 2.0
    return value / max_amp
```

## Epoch Design

The epoch tick size is a world design decision. Choose it at system startup and never change it.

| Tick Size | Use Case |
|-----------|----------|
| 1 tick = 1 in-game hour | Detailed NPC schedules, tidal cycles |
| 1 tick = 1 in-game day | Weather, crop cycles, patrol routes |
| 1 tick = 1 in-game season | Biome state, economic quarters |
| 1 tick = 1 in-game year | Dynastic history, tectonic drift |

### Cyclic vs Stochastic

```python
# CYCLIC: periodic function of epoch — no hash needed
def day_phase(epoch, ticks_per_day=24):
    return (epoch % ticks_per_day) / ticks_per_day

def seasonal_offset(epoch, ticks_per_year=365):
    return math.sin((epoch % ticks_per_year) / ticks_per_year * 2 * math.pi)

# STOCHASTIC: hash-based — deterministic but aperiodic
def storm_event(x, y, epoch, world_seed, ticks_per_week=7):
    return hash_to_float(temporal_hash(x//32, y//32, 0, epoch//ticks_per_week, world_seed+100)) > 0.85

# COMPOSED: cyclic baseline + stochastic variation
def temperature(x, y, epoch, world_seed, ticks_per_year=365):
    base    = coherent_value(x*0.005, y*0.005, world_seed)
    season  = 0.3 * seasonal_offset(epoch, ticks_per_year)
    noise   = (hash_to_float(temporal_hash(x//8, y//8, 0, epoch, world_seed+1)) - 0.5) * 0.1
    return base + season + noise
```

## Quick Recipes

### Weather System
```python
def weather(x, y, epoch, world_seed, ticks_per_day=24, ticks_per_year=365):
    """Full weather state at any position, any epoch. O(1)."""
    base_temp     = coherent_value(x*0.005, y*0.005, world_seed)
    base_moisture = coherent_value(x*0.005, y*0.005, world_seed + 500)
    season_phase  = seasonal_offset(epoch, ticks_per_year)
    storm_seed    = temporal_hash(x//32, y//32, 0, epoch//7, world_seed + 100)
    daily_seed    = temporal_hash(x//8,  y//8,  0, epoch//ticks_per_day, world_seed + 200)
    cloud_cover   = hash_to_float(daily_seed)
    temp = base_temp + 0.3*season_phase + 0.05*(hash_to_float(daily_seed >> 16) - 0.5)
    return {
        "temperature":   temp,
        "cloud_cover":   cloud_cover,
        "storm_active":  hash_to_float(storm_seed) > 0.85 and base_moisture > 0.4,
        "precipitation": max(0.0, cloud_cover - 0.5) * base_moisture
    }
```

### Seasonal Terrain State
```python
def terrain_state(x, y, epoch, world_seed, ticks_per_year=365):
    """Static base + temporal seasonal overlay."""
    elevation = coherent_value(x*0.02, y*0.02, world_seed)
    moisture  = coherent_value(x*0.02, y*0.02, world_seed + 1000)
    is_winter = seasonal_offset(epoch, ticks_per_year) < -0.5
    if elevation < -0.2:
        biome = "frozen_sea" if is_winter else "ocean"
    elif elevation < 0.3:
        biome = "snow_plains" if (is_winter and moisture > 0.2) else ("forest" if moisture > 0.3 else "plains")
    else:
        biome = "snow_mountain" if is_winter else "mountain"
    if elevation < -0.1 and seasonal_offset(epoch, ticks_per_year) > 0.3 and moisture > 0.6:
        biome = "flood_plain"
    return {"biome": biome, "elevation": elevation, "moisture": moisture}
```

### NPC Schedule
```python
def npc_schedule(npc_id, epoch, world_seed, ticks_per_day=24):
    """Where is this NPC and what are they doing at this epoch? O(1)."""
    time_of_day = epoch % ticks_per_day
    day_index   = epoch // ticks_per_day
    home_seed   = position_hash(npc_id, 0, 0, world_seed)
    home        = (int(hash_to_float(home_seed) * 100), int(hash_to_float(home_seed >> 16) * 100))
    day_seed    = temporal_hash(npc_id, 0, 0, day_index, world_seed + 300)
    dest        = (int(hash_to_float(day_seed) * 100), int(hash_to_float(day_seed >> 16) * 100))
    errand      = ["market","temple","tavern","fields","workshop"][int(hash_to_float(day_seed >> 32) * 5)]
    if time_of_day < 6:    return {"location": home, "activity": "sleeping"}
    elif time_of_day < 8:  return {"location": home, "activity": "morning_routine"}
    elif time_of_day < 17: return {"location": dest, "activity": errand}
    elif time_of_day < 20: return {"location": dest, "activity": "returning"}
    else:                  return {"location": home, "activity": "evening"}
```

### Economic Boom-Bust
```python
def market_price(commodity_id, market_x, market_y, epoch, world_seed, cycle_length=200):
    """Deterministic commodity price at any market at any epoch."""
    base_seed  = position_hash(commodity_id, market_x//10, market_y//10, world_seed)
    base_value = 10 + hash_to_float(base_seed) * 90
    cycle      = math.sin((epoch / cycle_length) * 2 * math.pi)
    vol_seed   = temporal_hash(commodity_id, market_x//20, market_y//20, epoch//30, world_seed+500)
    volatility = (hash_to_float(vol_seed) - 0.5) * 0.4
    return base_value * (1 + 0.3 * cycle + volatility)
```

### Historical Archaeology
```python
# The past is just a different epoch coordinate — query it directly, no replay
def historical_state(x, y, past_epoch, world_seed):
    return terrain_state(x, y, past_epoch, world_seed)
```

## Hierarchy Pattern

```python
def regional_epoch_seed(region_x, region_y, epoch, world_seed, region_scale=32):
    """Region seed changes slowly — chunked by region and epoch-block."""
    return temporal_hash(region_x//region_scale, region_y//region_scale,
                         0, epoch//region_scale, world_seed)

def local_epoch_seed(x, y, epoch, region_seed, local_scale=8):
    """Local seed inherits regional context."""
    return temporal_hash(x//local_scale, y//local_scale,
                         region_seed & 0xFFFF, epoch//local_scale, 0)
```

A region in "drought epoch" generates statistically drier local tiles and depleted market prices — through seed inheritance, not stored drought state.

## Composing with ZeroBytes and Zero-Quadratic

| Layer | Answers | Example |
|-------|---------|---------|
| ZeroBytes | What is permanently at this position? | Elevation, ore veins |
| Zero-Temporal | What is here right now? | Weather, seasonal biome |
| Zero-Quadratic | What is the permanent relationship? | Cultural distance |
| ZQ + Temporal | How does the relationship feel today? | Current trade pressure |

## Anti-Patterns

```python
# BAD: Wall-clock time as epoch — non-deterministic across machines
epoch = int(time.time())

# BAD: Accumulated state — simulation, not Zero-Temporal
world_state[epoch] = simulate(world_state[epoch - 1])

# BAD: Iterating to reach epoch N — defeats O(1) access
state = initial_state
for tick in range(target_epoch): state = update(state)

# BAD: Storing computed values — defeats zero-bytes principle
weather_cache[(x, y, epoch)] = compute_weather(x, y, epoch)

# GOOD: Direct epoch coordinate query
w    = weather(x, y, current_epoch, world_seed)
past = terrain_state(x, y, epoch_500_years_ago, world_seed)
```

## Debugging Checklist

When temporal properties differ across machines:
1. **Check epoch source** — world tick counter, never `time.time()`
2. **Check struct format** — `'<qqqq'` little-endian 64-bit; `epoch` must be a Python `int`
3. **Check cyclic precision** — `math.sin` on large float epoch can drift; use `(epoch % period)` as integer first

When properties don't feel like they change over time:
- Chunking too coarse — reduce epoch divisor so hash inputs change more frequently
- Add periodic sin/cos layer — hash alone is aperiodic and feels random, not rhythmic

When NPC schedules feel incoherent:
- Time-of-day must be `epoch % ticks_per_day` (cyclic), not a temporal hash
- Reserve temporal hashing for day-to-day variation, not hour-to-hour

## Determinism Verification

```python
def verify_temporal(temporal_fn, world_seed, position_epoch_pairs):
    for x, y, epoch in position_epoch_pairs:
        assert temporal_fn(x,y,epoch,world_seed) == temporal_fn(x,y,epoch,world_seed), \
            f"Non-deterministic at ({x},{y}) epoch={epoch}"
    x, y = position_epoch_pairs[0][:2]
    direct = temporal_fn(x, y, 10000, world_seed)
    _      = temporal_fn(x, y, 9999,  world_seed)   # query neighbour after
    assert temporal_fn(x, y, 10000, world_seed) == direct, "Order-dependent!"
    hist = temporal_fn(x, y, 0, world_seed)
    _    = temporal_fn(x, y, 10000, world_seed)
    assert temporal_fn(x, y, 0, world_seed) == hist, "Historical epoch non-deterministic!"
```

## Usage

1. **Define the epoch** — tick size at world design time; never change it
2. **Separate static from dynamic** — ZeroBytes for permanent; Zero-Temporal for time-varying
3. **Cyclic vs stochastic** — periodic (seasons, tides) → `sin`/`cos`; aperiodic events → `temporal_hash`
4. **Temporal granularity** — slow properties chunk at coarse epoch blocks; fast properties at fine
5. **Compose layers** — static base (ZeroBytes) + cycle (periodic) + events (temporal hash)
6. **Enable historical queries** — verify past epochs remain stable
7. **Verify** — run `verify_temporal` confirming O(1) access at arbitrary epochs

**Core principle:** The past is not a log. The future is not a simulation. Both are coordinates. Zero-Temporal queries them like any other dimension — in O(1), with zero bytes stored.
