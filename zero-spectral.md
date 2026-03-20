---
name: zero-spectral
description: Frequency-domain-is-seed procedural generation methodology. Extends ZeroBytes from point-by-point spatial hashing into region-level spectral fingerprinting. Use when a developer asks for adaptive level of detail, biome spectral signatures, terrain with characteristic frequency profiles, procedural audio character maps, signal interference patterns, region-scale pattern generation, or any system where the shape and texture of a region (its frequency content) matters as much as individual point values. Triggers on phrases like "spectral fingerprint", "frequency domain", "adaptive LOD", "terrain texture profile", "procedural audio map", "biome spectral signature", "zero-spectral", "frequency-based generation", "region pattern", or when ZeroBytes coherent noise needs richer regional character or variable resolution output from a single seed.
---

# Zero-Spectral: Frequency-Domain-is-Seed Procedural Generation

Extends [ZeroBytes](zerobytes/SKILL.md) by operating in the frequency domain rather than the spatial domain. Instead of hashing individual points and smoothing afterward, Zero-Spectral seeds the entire *spectral decomposition* of a region from a single region coordinate, then reconstructs any point within it analytically at any resolution.

## The Conceptual Leap from ZeroBytes

ZeroBytes asks: *What is the value at this position?*
Zero-Spectral asks: *What is the spectral character of this region, and what value does it imply at this position?*

ZeroBytes' coherent noise is already a primitive form of spectral thinking — it layers octaves of spatial frequency. Zero-Spectral formalises this into a first-class methodology: the region's *frequency decomposition* is the generative object, not the individual point values. From that decomposition, any point can be reconstructed analytically, at any resolution, with no loss of determinism.

**The key difference:** ZeroBytes generates points and smooths them. Zero-Spectral generates the smoothing function itself and then evaluates it at points. This makes the frequency profile of a region a designable, inspectable property — not an emergent accident of octave layering.

## The Five Extended Laws

Every spectral procedural system must satisfy:

1. **Region-First Access**: The spectral fingerprint of a region is computable in O(band_count) from the region coordinate; individual point values are derived from the fingerprint, not stored independently
2. **Resolution Independence**: The same region seed produces correct values at any spatial resolution — coarse overview and fine detail are reconstructions of the same spectral object
3. **Spectral Coherence**: Adjacent regions produce related frequency profiles — the spectral fingerprint changes smoothly across region boundaries, preventing spectral seams
4. **Hierarchy**: Regional spectra inherit from continental-scale spectra; local point spectra inherit from regional spectra; frequency content flows downward through scales
5. **Determinism**: Same region coordinate → same spectral fingerprint → same point value at any resolution, across all machines

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

def spectral_fingerprint(region_x, region_y, region_seed, band_count=8):
    """
    Generate the spectral decomposition of a region from its coordinate.
    Returns a list of (amplitude, phase, wavelength) tuples — one per frequency band.
    The fingerprint IS the region's generative identity.
    """
    region_s = position_hash(region_x, region_y, 0, region_seed)
    bands = []
    for band in range(band_count):
        band_seed = position_hash(region_s, band, 0, 0)
        # Amplitude decays with frequency (higher bands = finer detail = lower amplitude)
        base_amplitude = hash_to_float(band_seed) / (2.0 ** band)
        phase          = hash_to_float(band_seed >> 16) * 2 * math.pi
        wavelength     = 2.0 ** (band_count - band)  # coarse to fine
        bands.append((base_amplitude, phase, wavelength))
    return bands

def evaluate_spectral(local_x, local_y, fingerprint):
    """
    Reconstruct the value at a local position from a spectral fingerprint.
    local_x, local_y: position within the region (normalised 0.0–1.0)
    """
    value = 0.0
    for amplitude, phase, wavelength in fingerprint:
        freq = 1.0 / wavelength
        value += amplitude * math.sin(2 * math.pi * freq * local_x + phase)
        value += amplitude * math.cos(2 * math.pi * freq * local_y + phase * 0.7)
    # Normalise to -1.0..1.0 range
    max_amplitude = sum(a for a, _, _ in fingerprint)
    return value / (max_amplitude * 2) if max_amplitude > 0 else 0.0

def spectral_value(world_x, world_y, world_seed, region_size=256, band_count=8):
    """
    Full pipeline: world coordinate → region → spectral fingerprint → point value.
    Resolution-independent: same output at any query density.
    """
    region_x = world_x // region_size
    region_y = world_y // region_size
    local_x  = (world_x % region_size) / region_size
    local_y  = (world_y % region_size) / region_size

    fingerprint = spectral_fingerprint(region_x, region_y, world_seed, band_count)
    return evaluate_spectral(local_x, local_y, fingerprint)
```

## Spectral Profile Design

The power of Zero-Spectral over naive coherent noise is that the *shape* of the frequency spectrum is designable. Different biomes, terrain types, or audio environments have characteristic spectral profiles.

### Predefined Spectral Shapes

```python
def biome_spectral_profile(region_x, region_y, world_seed, band_count=8):
    """
    Choose spectral profile based on biome type.
    The biome IS defined by the shape of its spectrum — not by explicit assignment.
    """
    region_s  = position_hash(region_x, region_y, 0, world_seed)
    biome_val = hash_to_float(region_s)

    if biome_val < 0.25:
        profile = "smooth_plains"      # low-frequency dominated, gentle
    elif biome_val < 0.5:
        profile = "rolling_hills"      # balanced mid-frequency
    elif biome_val < 0.75:
        profile = "jagged_mountains"   # high-frequency dominated, rough
    else:
        profile = "fractal_coastline"  # all frequencies equal (pink noise)

    return _build_profile(region_x, region_y, world_seed, band_count, profile)

def _build_profile(region_x, region_y, world_seed, band_count, profile):
    region_s = position_hash(region_x, region_y, 0, world_seed)
    bands = []
    for band in range(band_count):
        band_seed = position_hash(region_s, band, 0, 0)
        phase     = hash_to_float(band_seed >> 16) * 2 * math.pi
        wavelength = 2.0 ** (band_count - band)

        # Amplitude shape determines the spectral character
        if profile == "smooth_plains":
            # Strong low-frequency, exponential rolloff — very smooth
            amplitude = hash_to_float(band_seed) / (4.0 ** band)
        elif profile == "rolling_hills":
            # Balanced: 1/f pink noise character
            amplitude = hash_to_float(band_seed) / (2.0 ** band)
        elif profile == "jagged_mountains":
            # High-frequency emphasis — rough and spiky
            amplitude = hash_to_float(band_seed) * (band / band_count) / (1.5 ** band)
        elif profile == "fractal_coastline":
            # Flat spectrum — equal energy at all scales
            amplitude = hash_to_float(band_seed) * 0.15
        else:
            amplitude = hash_to_float(band_seed) / (2.0 ** band)

        bands.append((amplitude, phase, wavelength))
    return bands
```

### Spectral Seam Blending

```python
def blended_spectral_value(world_x, world_y, world_seed, region_size=256, band_count=8, blend_zone=32):
    """
    Blend spectral fingerprints at region boundaries to eliminate seams.
    The blend zone width determines how smooth the transition between regions is.
    """
    rx = world_x // region_size
    ry = world_y // region_size
    lx = (world_x % region_size) / region_size
    ly = (world_y % region_size) / region_size

    # Smooth blend weights at boundaries
    def smooth(t): return t * t * (3 - 2 * t)
    bx = (world_x % region_size) / blend_zone
    by = (world_y % region_size) / blend_zone
    wx = smooth(min(1.0, bx)) if bx < 1.0 else 1.0 - smooth(min(1.0, (region_size - world_x % region_size) / blend_zone))
    wy = smooth(min(1.0, by)) if by < 1.0 else 1.0 - smooth(min(1.0, (region_size - world_y % region_size) / blend_zone))

    # Sample up to 4 surrounding regions and blend
    fp_00 = spectral_fingerprint(rx,   ry,   world_seed, band_count)
    fp_10 = spectral_fingerprint(rx+1, ry,   world_seed, band_count)
    fp_01 = spectral_fingerprint(rx,   ry+1, world_seed, band_count)
    fp_11 = spectral_fingerprint(rx+1, ry+1, world_seed, band_count)

    v00 = evaluate_spectral(lx, ly, fp_00)
    v10 = evaluate_spectral(lx, ly, fp_10)
    v01 = evaluate_spectral(lx, ly, fp_01)
    v11 = evaluate_spectral(lx, ly, fp_11)

    # Bilinear blend
    return (v00*(1-wx)*(1-wy) + v10*wx*(1-wy) +
            v01*(1-wx)*wy     + v11*wx*wy)
```

## Quick Recipes

### Adaptive Level of Detail
```python
def terrain_lod(world_x, world_y, world_seed, lod_level, region_size=256):
    """
    Same seed, different resolution. LOD 0 = coarse, LOD 4 = fine.
    Zero-Spectral naturally supports variable resolution from one fingerprint.
    """
    # Use fewer bands for coarser LOD — only low-frequency components
    active_bands = max(1, lod_level + 1)
    return spectral_value(world_x, world_y, world_seed, region_size, band_count=active_bands)

def lod_normal(world_x, world_y, world_seed, lod_level, region_size=256, epsilon=0.5):
    """Surface normal from spectral terrain — useful for lighting."""
    h   = terrain_lod(world_x,         world_y,         world_seed, lod_level, region_size)
    hx  = terrain_lod(world_x+epsilon, world_y,         world_seed, lod_level, region_size)
    hy  = terrain_lod(world_x,         world_y+epsilon, world_seed, lod_level, region_size)
    dx, dy = (hx - h) / epsilon, (hy - h) / epsilon
    length = (dx**2 + dy**2 + 1.0) ** 0.5
    return (-dx/length, -dy/length, 1.0/length)
```

### Biome Spectral Fingerprinting
```python
def biome_terrain(world_x, world_y, world_seed, region_size=256, band_count=8):
    """Terrain height that inherits the spectral character of its biome."""
    region_x = world_x // region_size
    region_y = world_y // region_size
    local_x  = (world_x % region_size) / region_size
    local_y  = (world_y % region_size) / region_size

    fingerprint = biome_spectral_profile(region_x, region_y, world_seed, band_count)
    height = evaluate_spectral(local_x, local_y, fingerprint)

    # Determine biome name for downstream use
    region_s  = position_hash(region_x, region_y, 0, world_seed)
    biome_val = hash_to_float(region_s)
    biome = ["plains", "hills", "mountains", "coast"][int(biome_val * 4)]

    return {"height": height, "biome": biome, "region": (region_x, region_y)}
```

### Procedural Audio Character Map
```python
def audio_character(world_x, world_y, world_seed):
    """
    What is the acoustic/musical character of this world region?
    Audio spaces have spectral fingerprints just as terrain does.
    """
    region_size = 128
    region_x = world_x // region_size
    region_y = world_y // region_size
    local_x  = (world_x % region_size) / region_size
    local_y  = (world_y % region_size) / region_size

    region_s = position_hash(region_x, region_y, 0, world_seed + 9000)

    # Spectral character defines the acoustic feel
    bass_energy   = hash_to_float(region_s) * 0.5 + \
                    evaluate_spectral(local_x, local_y,
                        spectral_fingerprint(region_x, region_y, world_seed+9001, band_count=3)) * 0.5
    treble_energy = hash_to_float(region_s >> 16) * 0.5 + \
                    evaluate_spectral(local_x, local_y,
                        spectral_fingerprint(region_x, region_y, world_seed+9002, band_count=3)) * 0.5
    reverb_length = hash_to_float(region_s >> 32)

    # Classify acoustic environment
    if reverb_length > 0.7:
        space = "cave" if bass_energy > 0.5 else "cathedral"
    elif bass_energy > 0.6:
        space = "forest" if treble_energy > 0.4 else "dense_forest"
    else:
        space = "open_plains"

    return {
        "space_type":    space,
        "bass_energy":   bass_energy,
        "treble_energy": treble_energy,
        "reverb":        reverb_length
    }
```

### Signal Interference Pattern
```python
def interference_field(world_x, world_y, world_seed, source_count=4):
    """
    Deterministic interference pattern — overlapping spectral sources.
    Models ley line resonance, radio frequency terrain, magical interference zones.
    """
    total = 0.0
    for source_idx in range(source_count):
        # Each source has a fixed position (ZeroBytes) and spectral character
        source_seed = position_hash(source_idx, 0, 0, world_seed + 8000)
        source_x    = hash_to_float(source_seed) * 1000
        source_y    = hash_to_float(source_seed >> 16) * 1000
        freq        = 0.01 + hash_to_float(source_seed >> 32) * 0.09

        dx = world_x - source_x
        dy = world_y - source_y
        dist = (dx**2 + dy**2) ** 0.5

        # Deterministic wave propagation from source
        phase_seed = position_hash(source_idx, 0, 0, world_seed + 8001)
        phase      = hash_to_float(phase_seed) * 2 * math.pi
        amplitude  = 1.0 / (1.0 + dist * 0.01)  # inverse distance falloff

        total += amplitude * math.sin(2 * math.pi * freq * dist + phase)

    return total / source_count  # normalised
```

## Hierarchy Pattern

ZeroBytes hierarchy: `parent_seed → child_seed via position`
Zero-Spectral adds: `parent_spectral_fingerprint → child_spectral_fingerprint via position`

```python
def hierarchical_spectral_fingerprint(continent_x, continent_y, region_x, region_y,
                                       world_seed, band_count=8):
    """
    Regional fingerprint inherits continent-scale spectral character.
    Continental roughness (high-frequency emphasis) bleeds into regional terrain.
    """
    # Continental fingerprint (coarse — few bands)
    continent_fp = spectral_fingerprint(continent_x, continent_y, world_seed, band_count=4)
    # Evaluate the continental character at the region's location
    continent_char = evaluate_spectral(
        (region_x % 8) / 8.0, (region_y % 8) / 8.0, continent_fp
    )

    # Use continental character to weight regional spectral amplitude
    region_s = position_hash(region_x, region_y, 0, world_seed + 1)
    bands = []
    for band in range(band_count):
        band_seed  = position_hash(region_s, band, 0, 0)
        phase      = hash_to_float(band_seed >> 16) * 2 * math.pi
        wavelength = 2.0 ** (band_count - band)
        # Continental roughness boosts high-frequency bands in child regions
        roughness_boost = 1.0 + continent_char * (band / band_count)
        amplitude  = hash_to_float(band_seed) / (2.0 ** band) * roughness_boost
        bands.append((amplitude, phase, wavelength))
    return bands
```

## Anti-Patterns

```python
# BAD: Generating points then smoothing — loses spectral control
values = [[hash_to_float(position_hash(x, y, 0, seed)) for x in range(256)] for y in range(256)]
smoothed = gaussian_blur(values)   # spectral character is an accident, not a design

# BAD: Fixed octave count for all regions — all regions have same spectral shape
def bad_terrain(x, y, seed):
    return sum(coherent_value(x*2**i, y*2**i, seed) / 2**i for i in range(8))

# BAD: Re-generating the full fingerprint per point — defeats region-first access
for x in range(256):
    for y in range(256):
        fp = spectral_fingerprint(x//256, y//256, seed)  # called 65536 times

# GOOD: Generate fingerprint once per region, evaluate at all points
region_fp = spectral_fingerprint(region_x, region_y, seed)
for lx in range(256):
    for ly in range(256):
        v = evaluate_spectral(lx/256, ly/256, region_fp)  # fingerprint reused
```

## Debugging Checklist

When spectral values differ across machines:
1. Check `math.sin` and `math.cos` — these are deterministic in IEEE 754 but verify platform compliance
2. Check struct pack format — `'<qqq'` for position_hash inputs
3. Check normalisation — `max_amplitude` must be computed from the same fingerprint used for evaluation

When region boundaries show visible seams:
- Use `blended_spectral_value` with a blend zone of at least 10% of region size
- Ensure adjacent regions share the same `world_seed` and `band_count`
- Check that `local_x` and `local_y` are correctly normalised to 0.0–1.0

When terrain feels "the same everywhere" despite different biomes:
- Verify that the spectral profile selection is actually changing amplitudes, not just phases
- Phase variation alone produces the same texture character — amplitude variation changes roughness
- Add continental-scale inheritance to ensure macro-scale differentiation

## Determinism Verification

```python
def verify_spectral(world_seed, test_positions, region_size=256, band_count=8):
    """Verify spectral determinism and resolution independence."""
    # Determinism at same resolution
    for (x, y) in test_positions:
        v1 = spectral_value(x, y, world_seed, region_size, band_count)
        v2 = spectral_value(x, y, world_seed, region_size, band_count)
        assert abs(v1 - v2) < 1e-9, f"Non-deterministic at ({x},{y})"

    # Resolution independence: fingerprint is the same regardless of query density
    x, y = test_positions[0]
    fp_direct   = spectral_fingerprint(x // region_size, y // region_size, world_seed, band_count)
    fp_rederived = spectral_fingerprint(x // region_size, y // region_size, world_seed, band_count)
    for (a1, p1, w1), (a2, p2, w2) in zip(fp_direct, fp_rederived):
        assert abs(a1-a2) < 1e-9 and abs(p1-p2) < 1e-9 and w1 == w2, "Fingerprint non-deterministic!"

    # Cross-resolution consistency: coarser LOD should be a smooth version of fine LOD
    coarse = [spectral_value(x, y, world_seed, region_size, 2) for x, y in test_positions]
    fine   = [spectral_value(x, y, world_seed, region_size, band_count) for x, y in test_positions]
    # Fine detail adds variation but should not drastically change mean
    coarse_mean = sum(coarse) / len(coarse)
    fine_mean   = sum(fine) / len(fine)
    assert abs(coarse_mean - fine_mean) < 0.3, "LOD levels diverge too strongly!"

    print(f"Zero-Spectral verification passed: {len(test_positions)} positions, {band_count} bands.")
```

## Usage

When implementing a Zero-Spectral system:

1. **Define the region size** — the spatial extent of one spectral fingerprint; larger regions = more homogeneous areas
2. **Choose band count** — more bands = richer detail; 4–8 is typical; LOD can use fewer bands
3. **Design spectral profiles** — define amplitude shapes for distinct terrain/biome/audio types; this is the creative design space
4. **Handle seams** — use `blended_spectral_value` at region boundaries; blend zone = 10–20% of region size
5. **Apply hierarchy** — continental spectra modulate regional amplitude weights; regions inherit macro-scale character
6. **Generate fingerprint once** — cache per region during a render pass; evaluate individual points from the cached fingerprint
7. **Verify** — run `verify_spectral` confirming resolution independence and determinism

**Core principle:** The character of a region is its spectrum. Zero-Spectral seeds the spectrum first and derives points from it — not the other way round. Resolution becomes a query parameter, not a generation parameter.
