# Zero-Spectral Demo Plan

## Overview

A single-page interactive HTML demo showcasing the four core Zero-Spectral capabilities, all derived from a single user-adjustable world seed. The demo runs entirely in the browser using pure JS — no server, no WebAssembly, no dependencies beyond Canvas 2D API and the Web Audio API.

---

## Architecture

### Core Engine (shared across all four panels)

```js
// xxHash-style 64-bit substitute using 32-bit integers in JS
function positionHash(x, y, z, salt = 0) {
  let h = (salt ^ 0x9e3779b9) | 0;
  h = Math.imul(h ^ x, 0x517cc1b7);
  h = Math.imul(h ^ y, 0x27d4eb2f);
  h = Math.imul(h ^ z, 0xb5026f5a);
  h ^= h >>> 16;
  h = Math.imul(h, 0x45d9f3b);
  h ^= h >>> 16;
  return (h >>> 0);          // unsigned 32-bit
}

function hashToFloat(h) { return (h >>> 0) / 0x100000000; }

function spectralFingerprint(rx, ry, seed, bandCount = 8) {
  const rs = positionHash(rx, ry, 0, seed);
  const bands = [];
  for (let b = 0; b < bandCount; b++) {
    const bs = positionHash(rs, b, 0, 0);
    const amplitude  = hashToFloat(bs) / Math.pow(2, b);
    const phase      = hashToFloat(bs >>> 16) * 2 * Math.PI;
    const wavelength = Math.pow(2, bandCount - b);
    bands.push({ amplitude, phase, wavelength });
  }
  return bands;
}

function evaluateSpectral(lx, ly, fingerprint) {
  let value = 0;
  let maxAmp = 0;
  for (const { amplitude, phase, wavelength } of fingerprint) {
    const freq = 1.0 / wavelength;
    value  += amplitude * Math.sin(2 * Math.PI * freq * lx + phase);
    value  += amplitude * Math.cos(2 * Math.PI * freq * ly + phase * 0.7);
    maxAmp += amplitude;
  }
  return maxAmp > 0 ? value / (maxAmp * 2) : 0;
}
```

---

## Panel 1 — Adaptive Level of Detail

**Goal:** Show the same terrain at LOD 0–7 (1–8 frequency bands). All panels share the same seed and region; only `bandCount` changes.

**Visual:** 8 small 128×128 canvases in a row labelled "LOD 0" to "LOD 7". As LOD increases, fine detail progressively appears — the coarse silhouette remains identical, high-frequency ripples accumulate.

**Spectral recipe:**
```js
function terrainLOD(wx, wy, seed, lod, regionSize = 256) {
  const rx = Math.floor(wx / regionSize);
  const ry = Math.floor(wy / regionSize);
  const lx = (wx % regionSize) / regionSize;
  const ly = (wy % regionSize) / regionSize;
  const fp = spectralFingerprint(rx, ry, seed, Math.max(1, lod + 1));
  return evaluateSpectral(lx, ly, fp);
}
```

**Color map:** Greyscale 0–1 mapped to elevation palette (deep blue → sand → green → grey → white snow cap).

---

## Panel 2 — Biome Spectral Fingerprinting

**Goal:** Demonstrate that smooth plains, rolling hills, jagged mountains, and fractal coastlines have distinct amplitude profiles — not just different values, but different *textures*.

**Visual:** A 512×256 world canvas with four contiguous biome regions. Clicking a pixel shows its biome type + a small bar chart of its spectral amplitude profile (band 0–7).

**Biome amplitude shaping:**
```js
function biomefingerprint(rx, ry, seed, bandCount = 8) {
  const rs    = positionHash(rx, ry, 0, seed);
  const bv    = hashToFloat(rs);
  const biome = ["smooth_plains","rolling_hills","jagged_mountains","fractal_coast"][Math.floor(bv * 4)];
  const bands = [];
  for (let b = 0; b < bandCount; b++) {
    const bs    = positionHash(rs, b, 0, 0);
    const phase = hashToFloat(bs >>> 16) * 2 * Math.PI;
    const wl    = Math.pow(2, bandCount - b);
    const raw   = hashToFloat(bs);
    let amp;
    if      (biome === "smooth_plains")    amp = raw / Math.pow(4, b);               // steep low-freq rolloff
    else if (biome === "rolling_hills")    amp = raw / Math.pow(2, b);               // 1/f pink
    else if (biome === "jagged_mountains") amp = raw * (b / bandCount) / Math.pow(1.5, b); // high-freq emphasis
    else                                   amp = raw * 0.15;                          // flat fractal
    bands.push({ amplitude: amp, phase, wavelength: wl });
  }
  return { biome, bands };
}
```

**Color scheme per biome:** plains = pale yellow-green; hills = muted olive; mountains = slate grey with snow tint; coast = teal-blue gradient.

---

## Panel 3 — Procedural Audio Character Map

**Goal:** Navigate a 512×512 world map; clicking any cell plays a short synthesised tone whose timbre is determined by the audio character computed for that position — caves are boomy/reverberant, forests are warm/mid-range, plains are bright and dry.

**Visual:** A coloured 512×512 canvas rendered at 4 px/cell (128×128 logical cells). Each cell is colour-coded by `space_type`. Hovering shows bass/treble/reverb bars. Clicking fires Web Audio.

**Audio character derivation:**
```js
function audioCharacter(wx, wy, seed) {
  const rs = 128;
  const rx = Math.floor(wx / rs);
  const ry = Math.floor(wy / rs);
  const lx = (wx % rs) / rs;
  const ly = (wy % rs) / rs;
  const regionSeed = positionHash(rx, ry, 0, seed + 9000);
  const fp1 = spectralFingerprint(rx, ry, seed + 9001, 3);
  const fp2 = spectralFingerprint(rx, ry, seed + 9002, 3);
  const bass   = hashToFloat(regionSeed) * 0.5 + evaluateSpectral(lx, ly, fp1) * 0.5;
  const treble = hashToFloat(regionSeed >>> 16) * 0.5 + evaluateSpectral(lx, ly, fp2) * 0.5;
  const reverb = hashToFloat(regionSeed >>> 24);
  let spaceType;
  if      (reverb > 0.7) spaceType = bass > 0.5 ? "cave"         : "cathedral";
  else if (bass   > 0.6) spaceType = treble > 0.4 ? "forest"     : "dense_forest";
  else                   spaceType = "open_plains";
  return { spaceType, bass, treble, reverb };
}
```

**Synthesis recipe per space type (Web Audio API):**
- `cave`: low-freq sine, long reverb tail (convolver / feedback delay), Q-boosted resonance
- `cathedral`: sine + harmonics, hall reverb, slow attack
- `forest`: bandpass filtered noise + mid-range sine chords, short reverb
- `dense_forest`: heavier noise, attenuated highs, fluttery tremolo
- `open_plains`: clean sine ping, no reverb, fast decay

---

## Panel 4 — Signal Interference Patterns

**Goal:** Show how deterministic wave interference from seeded sources produces complex spatial resonance fields — useful for ley lines, RF terrain, magical zones, etc.

**Visual:** A 512×512 canvas rendered with a continuous colour map from interference value −1 → +1. A "source count" slider (2–8 sources) and animated time offset create flowing interference patterns. Hovering shows the interference value at that point.

**Interference field:**
```js
function interferenceField(wx, wy, seed, sourceCount = 4, time = 0) {
  let total = 0;
  for (let i = 0; i < sourceCount; i++) {
    const s    = positionHash(i, 0, 0, seed + 8000);
    const sx   = hashToFloat(s) * 512;
    const sy   = hashToFloat(s >>> 16) * 512;
    const freq = 0.01 + hashToFloat(s >>> 24) * 0.09;
    const dist = Math.hypot(wx - sx, wy - sy);
    const phase = hashToFloat(positionHash(i, 0, 0, seed + 8001)) * 2 * Math.PI;
    total += (1.0 / (1.0 + dist * 0.01)) * Math.sin(2 * Math.PI * freq * dist + phase + time);
  }
  return total / sourceCount;
}
```

**Color map:** Diverging palette — deep violet (−1) → black (0) → electric amber (+1), with contour lines at ±0.3 and ±0.7.

---

## UI Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  ZERO-SPECTRAL  ·  World Seed: [______]  [Randomise]                │
├─────────────────────────────────────────────────────────────────────┤
│  ① ADAPTIVE LOD                                                     │
│  [LOD0][LOD1][LOD2][LOD3][LOD4][LOD5][LOD6][LOD7]  ← 8 canvases   │
├─────────────────────────────────────────────────────────────────────┤
│  ② BIOME FINGERPRINTING                                             │
│  [512×256 world] | [amplitude bar chart for selected biome]         │
├─────────────────────────────────────────────────────────────────────┤
│  ③ AUDIO CHARACTER MAP                                              │
│  [512×512 colour map]  | [bass/treble/reverb bars] [▶ Play tone]    │
├─────────────────────────────────────────────────────────────────────┤
│  ④ INTERFERENCE FIELD                                               │
│  [512×512 field]  | Sources: [2——●——8]  [▶ Animate]                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Design Aesthetic

- **Theme:** Dark-mode scientific instrument / oscilloscope terminal
- **Palette:** Near-black background (#0a0c10), electric teal accents (#00e5c8), amber highlights (#ffb830), muted panel borders
- **Font:** `JetBrains Mono` for labels and values; `Space Mono` for headings
- **Canvas rendering:** Each canvas pixel-perfect at native resolution with `imageSmoothingEnabled = false`
- **No external libraries** — Web Canvas 2D + Web Audio API only; all imports from CDN fonts

---

## Implementation Notes

1. **Performance:** Fingerprint is computed once per region per render pass and cached by `"rx,ry,bandCount"` key — never recomputed per pixel.
2. **Seam blending:** Panel 2 uses `blendedSpectralValue` with a 32-pixel blend zone to prevent visible region edges.
3. **Web Audio context** created lazily on first user gesture to comply with browser autoplay policies.
4. **Interference animation** uses `requestAnimationFrame` with a time uniform; pauses when panel scrolls out of view (`IntersectionObserver`).
5. **Seed input** triggers full redraw of all panels; debounced 150 ms.
