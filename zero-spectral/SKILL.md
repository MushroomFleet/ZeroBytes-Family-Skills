---
name: zero-spectral
description: Frequency-domain-is-seed procedural generation methodology. Extends ZeroBytes from point-by-point spatial hashing into region-level spectral fingerprinting. Use when a developer asks for adaptive level of detail, biome spectral signatures, terrain with characteristic frequency profiles, procedural audio character maps, signal interference patterns, region-scale pattern generation, or any system where the shape and texture of a region (its frequency content) matters as much as individual point values. Triggers on phrases like "spectral fingerprint", "frequency domain", "adaptive LOD", "terrain texture profile", "procedural audio map", "biome spectral signature", "zero-spectral", "frequency-based generation", "region pattern", or when ZeroBytes coherent noise needs richer regional character or variable resolution output from a single seed.
---

# Zero-Spectral: Frequency-Domain-is-Seed Procedural Generation

Extends [ZeroBytes](../zerobytes/SKILL.md) by operating in the frequency domain rather than the spatial domain. Instead of hashing individual points and smoothing afterward, Zero-Spectral seeds the entire *spectral decomposition* of a region from a single region coordinate, then reconstructs any point within it analytically at any resolution.

**The key difference:** ZeroBytes generates points and smooths them. Zero-Spectral generates the smoothing function itself and evaluates points from it. The frequency profile of a region becomes a designable property — not an emergent accident of octave layering.

## The Five Extended Laws

1. **Region-First Access**: The spectral fingerprint of a region is computable in O(band_count) from the region coordinate; point values derive from the fingerprint, not stored independently
2. **Resolution Independence**: The same region seed produces correct values at any spatial resolution — coarse and fine are reconstructions of the same spectral object
3. **Spectral Coherence**: Adjacent regions produce related frequency profiles; fingerprints change smoothly across boundaries, preventing seams
4. **Hierarchy**: Regional spectra inherit from continental spectra; frequency content flows downward through scales
5. **Determinism**: Same region coordinate → same spectral fingerprint → same point value at any resolution, on all machines

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

def spectral_fingerprint(region_x, region_y, region_seed, band_count=8):
    """
    Seed the spectral decomposition of a region from its coordinate.
    Returns list of (amplitude, phase, wavelength) — one per frequency band.
    Generate once per region; evaluate many points from the same fingerprint.
    """
    region_s = position_hash(region_x, region_y, 0, region_seed)
    bands = []
    for band in range(band_count):
        band_seed = position_hash(region_s, band, 0, 0)
        amplitude = hash_to_float(band_seed) / (2.0 ** band)       # decays with frequency
        phase     = hash_to_float(band_seed >> 16) * 2 * math.pi
        wavelength = 2.0 ** (band_count - band)                    # coarse → fine
        bands.append((amplitude, phase, wavelength))
    return bands

def evaluate_spectral(local_x, local_y, fingerprint):
    """
    Reconstruct value at a local position (0.0–1.0) from a spectral fingerprint.
    """
    value = 0.0
    for amplitude, phase, wavelength in fingerprint:
        freq   = 1.0 / wavelength
        value += amplitude * math.sin(2 * math.pi * freq * local_x + phase)
        value += amplitude * math.cos(2 * math.pi * freq * local_y + phase * 0.7)
    max_amp = sum(a for a, _, _ in fingerprint)
    return value / (max_amp * 2) if max_amp > 0 else 0.0

def spectral_value(world_x, world_y, world_seed, region_size=256, band_count=8):
    """Full pipeline: world coord → region → fingerprint → point value."""
    rx = world_x // region_size;  ry = world_y // region_size
    lx = (world_x % region_size) / region_size
    ly = (world_y % region_size) / region_size
    return evaluate_spectral(lx, ly, spectral_fingerprint(rx, ry, world_seed, band_count))
```

## Spectral Profile Design

The amplitude shape per band is the creative design space — it defines the texture character of a region.

```python
def biome_spectral_profile(region_x, region_y, world_seed, band_count=8):
    """Biome IS defined by the shape of its spectrum, not by explicit assignment."""
    region_s  = position_hash(region_x, region_y, 0, world_seed)
    biome_val = hash_to_float(region_s)
    profile   = (["smooth_plains", "rolling_hills", "jagged_mountains", "fractal_coastline"]
                 [int(biome_val * 4)])
    bands = []
    for band in range(band_count):
        band_seed  = position_hash(region_s, band, 0, 0)
        phase      = hash_to_float(band_seed >> 16) * 2 * math.pi
        wavelength = 2.0 ** (band_count - band)
        raw        = hash_to_float(band_seed)
        if   profile == "smooth_plains":     amplitude = raw / (4.0 ** band)          # strong low-freq
        elif profile == "rolling_hills":     amplitude = raw / (2.0 ** band)          # 1/f pink noise
        elif profile == "jagged_mountains":  amplitude = raw * (band/band_count) / (1.5 ** band)  # high-freq emphasis
        else:                                amplitude = raw * 0.15                   # flat / fractal
        bands.append((amplitude, phase, wavelength))
    return bands
```

### Seam Blending

```python
def blended_spectral_value(world_x, world_y, world_seed, region_size=256, band_count=8, blend_zone=32):
    """Blend fingerprints at region boundaries — blend_zone = 10–20% of region_size."""
    rx = world_x // region_size;  ry = world_y // region_size
    lx = (world_x % region_size) / region_size
    ly = (world_y % region_size) / region_size
    def smooth(t): return t*t*(3-2*t)
    bx = (world_x % region_size) / blend_zone
    by = (world_y % region_size) / blend_zone
    wx = smooth(min(1.0, bx)) if bx < 1.0 else 1.0 - smooth(min(1.0, (region_size - world_x % region_size) / blend_zone))
    wy = smooth(min(1.0, by)) if by < 1.0 else 1.0 - smooth(min(1.0, (region_size - world_y % region_size) / blend_zone))
    v00 = evaluate_spectral(lx, ly, spectral_fingerprint(rx,   ry,   world_seed, band_count))
    v10 = evaluate_spectral(lx, ly, spectral_fingerprint(rx+1, ry,   world_seed, band_count))
    v01 = evaluate_spectral(lx, ly, spectral_fingerprint(rx,   ry+1, world_seed, band_count))
    v11 = evaluate_spectral(lx, ly, spectral_fingerprint(rx+1, ry+1, world_seed, band_count))
    return v00*(1-wx)*(1-wy) + v10*wx*(1-wy) + v01*(1-wx)*wy + v11*wx*wy
```

## Quick Recipes

### Adaptive Level of Detail
```python
def terrain_lod(world_x, world_y, world_seed, lod_level, region_size=256):
    """Same seed, variable resolution. LOD 0 = coarse (1 band), LOD 7 = fine (8 bands)."""
    return spectral_value(world_x, world_y, world_seed, region_size, band_count=max(1, lod_level+1))
```

### Biome Terrain
```python
def biome_terrain(world_x, world_y, world_seed, region_size=256):
    """Height inheriting the spectral character of its biome."""
    rx = world_x // region_size;  ry = world_y // region_size
    lx = (world_x % region_size) / region_size
    ly = (world_y % region_size) / region_size
    fp     = biome_spectral_profile(rx, ry, world_seed)
    height = evaluate_spectral(lx, ly, fp)
    biome  = ["plains","hills","mountains","coast"][int(hash_to_float(position_hash(rx,ry,0,world_seed)) * 4)]
    return {"height": height, "biome": biome}
```

### Procedural Audio Character
```python
def audio_character(world_x, world_y, world_seed):
    """Acoustic character of a world region — no stored audio, computed from position."""
    rs = 128
    rx = world_x // rs;  ry = world_y // rs
    lx = (world_x % rs) / rs;  ly = (world_y % rs) / rs
    region_s = position_hash(rx, ry, 0, world_seed + 9000)
    bass   = hash_to_float(region_s) * 0.5 + evaluate_spectral(lx, ly, spectral_fingerprint(rx, ry, world_seed+9001, 3)) * 0.5
    treble = hash_to_float(region_s >> 16) * 0.5 + evaluate_spectral(lx, ly, spectral_fingerprint(rx, ry, world_seed+9002, 3)) * 0.5
    reverb = hash_to_float(region_s >> 32)
    if   reverb > 0.7:  space = "cave" if bass > 0.5 else "cathedral"
    elif bass > 0.6:    space = "forest" if treble > 0.4 else "dense_forest"
    else:               space = "open_plains"
    return {"space_type": space, "bass": bass, "treble": treble, "reverb": reverb}
```

### Signal Interference Pattern
```python
def interference_field(world_x, world_y, world_seed, source_count=4):
    """Deterministic wave interference — ley lines, RF terrain, magical resonance zones."""
    total = 0.0
    for i in range(source_count):
        s     = position_hash(i, 0, 0, world_seed + 8000)
        sx    = hash_to_float(s) * 1000
        sy    = hash_to_float(s >> 16) * 1000
        freq  = 0.01 + hash_to_float(s >> 32) * 0.09
        dist  = ((world_x - sx)**2 + (world_y - sy)**2) ** 0.5
        phase = hash_to_float(position_hash(i, 0, 0, world_seed + 8001)) * 2 * math.pi
        total += (1.0 / (1.0 + dist * 0.01)) * math.sin(2 * math.pi * freq * dist + phase)
    return total / source_count
```

## Hierarchy Pattern

```python
def hierarchical_spectral_fingerprint(continent_x, continent_y, region_x, region_y,
                                       world_seed, band_count=8):
    """Regional fingerprint inherits continent-scale roughness character."""
    continent_char = evaluate_spectral(
        (region_x % 8) / 8.0, (region_y % 8) / 8.0,
        spectral_fingerprint(continent_x, continent_y, world_seed, band_count=4)
    )
    region_s = position_hash(region_x, region_y, 0, world_seed + 1)
    bands = []
    for band in range(band_count):
        band_seed       = position_hash(region_s, band, 0, 0)
        phase           = hash_to_float(band_seed >> 16) * 2 * math.pi
        wavelength      = 2.0 ** (band_count - band)
        roughness_boost = 1.0 + continent_char * (band / band_count)
        amplitude       = hash_to_float(band_seed) / (2.0 ** band) * roughness_boost
        bands.append((amplitude, phase, wavelength))
    return bands
```

## Anti-Patterns

```python
# BAD: Generate points then smooth — spectral character is accidental
values   = [[hash_to_float(position_hash(x,y,0,seed)) for x in range(256)] for y in range(256)]
smoothed = gaussian_blur(values)

# BAD: Same octave rolloff for all regions — no biome differentiation
def bad_terrain(x, y, seed):
    return sum(coherent_value(x*2**i, y*2**i, seed) / 2**i for i in range(8))

# BAD: Regenerating fingerprint per point — defeats region-first access
for x in range(256):
    for y in range(256):
        fp = spectral_fingerprint(x//256, y//256, seed)   # 65536 redundant calls

# GOOD: Generate fingerprint once, evaluate all points from it
fp = spectral_fingerprint(region_x, region_y, seed)
for lx in range(256):
    for ly in range(256):
        v = evaluate_spectral(lx/256, ly/256, fp)
```

## Debugging Checklist

When spectral values differ across machines:
1. `math.sin`/`math.cos` are IEEE 754 deterministic — verify no fast-math compiler flags
2. Check struct format — `'<qqq'` little-endian signed 64-bit
3. `max_amplitude` must be computed from the same fingerprint used for evaluation

When region boundaries show visible seams:
- Use `blended_spectral_value`; blend zone ≥ 10% of region size
- Verify `local_x`/`local_y` normalised to 0.0–1.0

When terrain feels the same everywhere despite different biomes:
- Check amplitude shape is changing, not just phase — phase variation alone preserves texture character
- Add continental hierarchy to ensure macro-scale differentiation

## Determinism Verification

```python
def verify_spectral(world_seed, test_positions, region_size=256, band_count=8):
    for x, y in test_positions:
        v1 = spectral_value(x, y, world_seed, region_size, band_count)
        v2 = spectral_value(x, y, world_seed, region_size, band_count)
        assert abs(v1 - v2) < 1e-9, f"Non-deterministic at ({x},{y})"
    # Resolution independence: fingerprint identical on re-derivation
    x, y = test_positions[0]
    fp1 = spectral_fingerprint(x//region_size, y//region_size, world_seed, band_count)
    fp2 = spectral_fingerprint(x//region_size, y//region_size, world_seed, band_count)
    for (a1,p1,w1),(a2,p2,w2) in zip(fp1, fp2):
        assert abs(a1-a2) < 1e-9 and abs(p1-p2) < 1e-9 and w1==w2, "Fingerprint non-deterministic!"
    # LOD consistency: coarse mean should not diverge from fine mean
    coarse = [spectral_value(x,y,world_seed,region_size,2) for x,y in test_positions]
    fine   = [spectral_value(x,y,world_seed,region_size,band_count) for x,y in test_positions]
    assert abs(sum(coarse)/len(coarse) - sum(fine)/len(fine)) < 0.3, "LOD levels diverge!"
```

## Usage

1. **Define region size** — spatial extent of one fingerprint; larger = more homogeneous areas
2. **Choose band count** — 4–8 typical; LOD uses fewer bands at lower detail levels
3. **Design spectral profiles** — amplitude shape per band is the creative space; defines roughness character
4. **Handle seams** — `blended_spectral_value` at boundaries; blend zone = 10–20% of region size
5. **Apply hierarchy** — continental character modulates regional amplitude weights
6. **Generate fingerprint once** — per region per render pass; evaluate all points from the cached fingerprint
7. **Verify** — run `verify_spectral` confirming resolution independence and determinism

**Core principle:** The character of a region is its spectrum. Zero-Spectral seeds the spectrum first and derives points from it. Resolution becomes a query parameter, not a generation parameter.
