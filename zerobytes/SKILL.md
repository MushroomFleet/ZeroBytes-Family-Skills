---
name: zerobytes
description: Position-is-seed procedural generation methodology. Use when developer asks for infinite world generation, procedural terrain, deterministic content generation, Minecraft-style chunks, roguelike dungeons, star system generation, or any task requiring reproducible random content from coordinates. Triggers on phrases like "procedural generation", "infinite world", "seed-based", "deterministic random", "tile generation", "chunk loading", "procedural terrain", "hash-based generation", "world seed", or when debugging inconsistent procedural results across machines/sessions.
---

# Zerobytes: Position-is-Seed Procedural Generation

Replace sequential state mutation with pure coordinate hashing. The coordinate IS the seed.

## The Five Laws

Every procedural system must satisfy:

1. **O(1) Access**: Compute any position without iterating preceding positions
2. **Parallelism**: Each element depends only on its coordinates, never siblings during generation
3. **Coherence**: Adjacent coordinates produce related values via noise layering
4. **Hierarchy**: Child seeds derive from parent seeds plus local position
5. **Determinism**: Same inputs → same outputs across all machines and execution orders

## Core Pattern

```python
import struct
import xxhash

def position_hash(x: int, y: int, z: int, salt: int = 0) -> int:
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqq', x, y, z))
    return h.intdigest()

def hash_to_float(h: int) -> float:
    return (h & 0xFFFFFFFF) / 0x100000000
```

## Hash Selection by Platform

| Context | Use | Avoid |
|---------|-----|-------|
| Python | `xxhash`, `blake2b` | `hash()`, `random` |
| C/C++ | xxHash3, MurmurHash3 | `rand()` |
| GPU | PCG, Wang hash | LCG |
| JS | `xxhash-wasm` | `Math.random()` |

## Coherent Noise (for regional properties)

```python
def coherent_value(x: float, y: float, seed: int, octaves: int = 4) -> float:
    value, amp, freq, max_amp = 0.0, 1.0, 1.0, 0.0
    for i in range(octaves):
        x0, y0 = int(x * freq), int(y * freq)
        sx = ((x * freq) % 1); sx = sx * sx * (3 - 2 * sx)
        sy = ((y * freq) % 1); sy = sy * sy * (3 - 2 * sy)
        n00 = hash_to_float(position_hash(x0, y0, 0, seed+i)) * 2 - 1
        n10 = hash_to_float(position_hash(x0+1, y0, 0, seed+i)) * 2 - 1
        n01 = hash_to_float(position_hash(x0, y0+1, 0, seed+i)) * 2 - 1
        n11 = hash_to_float(position_hash(x0+1, y0+1, 0, seed+i)) * 2 - 1
        nx0, nx1 = n00*(1-sx)+n10*sx, n01*(1-sx)+n11*sx
        value += amp * (nx0*(1-sy)+nx1*sy)
        max_amp += amp; amp *= 0.5; freq *= 2.0
    return value / max_amp
```

## Anti-Patterns to Flag

```python
# BAD: Sequential iteration
for i in range(target): state = next(state)

# BAD: Platform hash
seed = hash((x, y))

# BAD: Time-based
seed = int(time.time())

# BAD: Global state
random.random()

# BAD: Neighbor during generation
def gen(x,y): return f(gen(x+1,y))

# GOOD: Direct coordinate hash
result = position_hash(x, y, z, world_seed)
```

## Quick Recipes

### Infinite Tilemap
```python
def tile(x, y, seed):
    elevation = coherent_value(x*0.02, y*0.02, seed)
    moisture = coherent_value(x*0.02, y*0.02, seed+1000)
    if elevation < -0.2: return "water"
    elif elevation < 0.3: return "forest" if moisture > 0 else "plains"
    else: return "mountain"
```

### Dungeon Room
```python
def room(rx, ry, floor, seed):
    rseed = position_hash(rx, ry, floor, seed)
    rng = [hash_to_float(position_hash(rseed, i, 0, 0)) for i in range(10)]
    return {
        "size": (5+int(rng[0]*15), 5+int(rng[1]*15)),
        "type": ["empty","combat","treasure"][int(rng[2]*3)],
        "difficulty": 1.0 + floor*0.3
    }
```

### Star System
```python
def star_system(sx, sy, sz, seed):
    sseed = position_hash(sx, sy, sz, seed)
    mass = 0.1 + hash_to_float(sseed)**2 * 10
    return {
        "mass": mass,
        "luminosity": mass**3.5,
        "planets": int(hash_to_float(position_hash(sseed,1,0,0)) * 8)
    }
```

## Hierarchy Pattern

```
Galaxy(seed)
  └─ Region(hash(seed, region_pos))
       └─ System(hash(region_seed, system_pos))
            └─ Planet(hash(system_seed, orbit_idx))
                 └─ Terrain(hash(planet_seed, lat, lon))
```

Each level inherits parent constraints. Properties at child level are bounded by parent.

## Debugging Checklist

When results inconsistent across machines:
1. Check for `random.random()` without seed
2. Check for platform `hash()` 
3. Check for `time.time()` seeds
4. Check for float accumulation
5. Check for generation order dependency

When results "too chaotic":
- Add coherent noise layer
- Frequency 0.001-0.1 typical for regional properties

When parallelization fails:
- Ensure no neighbor lookup during generation
- Move neighbor-dependent ops to post-process pass

## Determinism Verification

```python
def verify(gen_fn, seed, positions):
    fwd = {p: gen_fn(*p, seed) for p in positions}
    rev = {p: gen_fn(*p, seed) for p in reversed(positions)}
    assert all(fwd[p] == rev[p] for p in positions), "Order-dependent!"
    assert gen_fn(*positions[0], seed) == fwd[positions[0]], "Non-deterministic!"
```

## Usage

When implementing procedural generation:

1. Identify what properties need coherence (biomes, regions) vs pure random (item drops)
2. Design hierarchy: what contains what, how do seeds flow
3. Implement with `position_hash` + `coherent_value` as needed
4. Verify determinism with order-independence test
5. Profile: hash is O(1), iteration is O(n)

**Core principle:** The universe springs complete from coordinates. Zero bytes store infinity.
