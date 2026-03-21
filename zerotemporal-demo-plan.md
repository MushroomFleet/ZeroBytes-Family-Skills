# Zero-Temporal Demo Plan
## Deterministic Weather · Seasonal Terrain · NPC Schedules

---

## Overview

An interactive browser demo implementing three Zero-Temporal systems side-by-side, all sharing a single **world epoch** slider. Every value is computed in O(1) — no simulation, no stored state — by hashing `(x, y, z, epoch)` as the seed.

**World Tick Size:** 1 tick = 1 in-game hour  
**Ticks Per Day:** 24  
**Ticks Per Year:** 8760 (365 × 24)  
**World Seed:** 42 (arbitrary; baked into demo)

---

## System 1 — Deterministic Weather Grid

### What it shows
A 12×8 tile map where each tile independently queries its weather state at the current epoch. No weather "spreads" — each tile is computed in isolation.

### Data model
```js
function temporalHash(x, y, z, epoch, salt = 0) {
  // xxHash64 equivalent via FNV-style mixing in pure JS
  let h = BigInt(salt) ^ 0xcbf29ce484222325n;
  for (const v of [x, y, z, epoch]) {
    h ^= BigInt(v);
    h = BigInt.asUintN(64, h * 0x100000001b3n);
  }
  return Number(h & 0xFFFFFFFFn) / 0x100000000;
}

function coherentValue(x, y, seed, octaves = 4) {
  // Smooth fractional noise from position hash — static spatial property
}

function weather(x, y, epoch) {
  const baseTemp     = coherentValue(x * 0.12, y * 0.12, WORLD_SEED);
  const baseMoisture = coherentValue(x * 0.12, y * 0.12, WORLD_SEED + 500);
  const seasonPhase  = Math.sin((epoch % TICKS_PER_YEAR) / TICKS_PER_YEAR * 2 * Math.PI);
  const stormVal     = temporalHash(Math.floor(x / 4), Math.floor(y / 4), 0,
                                    Math.floor(epoch / (TICKS_PER_DAY * 7)), WORLD_SEED + 100);
  const dailyVal     = temporalHash(Math.floor(x / 2), Math.floor(y / 2), 0,
                                    Math.floor(epoch / TICKS_PER_DAY), WORLD_SEED + 200);
  return {
    temperature:   baseTemp + 0.3 * seasonPhase,
    cloudCover:    (dailyVal * 0.5 + 0.25) + baseMoisture * 0.3,
    stormActive:   stormVal > 0.85 && baseMoisture > 0.4,
    precipitation: Math.max(0, dailyVal - 0.5) * baseMoisture,
  };
}
```

### Visual encoding
| Property | Visual |
|----------|--------|
| Temperature | Tile background: blue (cold) → amber (warm) |
| Cloud cover | White opacity overlay |
| Storm active | Lightning ⚡ icon + dark tint |
| Precipitation | Rain 🌧 icon + blue tint |

---

## System 2 — Seasonal Terrain State

### What it shows
A 16×10 terrain map. Each cell has a **permanent static biome** (ZeroBytes layer) overlaid with a **seasonal state** (Zero-Temporal layer). Tiles can transition: plains → snow_plains, ocean → frozen_sea, lowland → flood_plain.

### Data model
```js
function terrainState(x, y, epoch) {
  const elevation = coherentValue(x * 0.08, y * 0.08, WORLD_SEED + 1000);
  const moisture  = coherentValue(x * 0.08, y * 0.08, WORLD_SEED + 2000);
  const seasonVal = Math.sin((epoch % TICKS_PER_YEAR) / TICKS_PER_YEAR * 2 * Math.PI);
  const isWinter  = seasonVal < -0.5;
  const isSpring  = seasonVal > 0.3 && moisture > 0.6;

  let biome;
  if      (elevation < -0.3)                          biome = isWinter ? "frozen_sea"    : "ocean";
  else if (elevation < -0.1 && isSpring)              biome = "flood_plain";
  else if (elevation < 0.15)                          biome = isWinter ? "snow_plains"   : (moisture > 0.4 ? "forest" : "plains");
  else if (elevation < 0.4)                           biome = isWinter ? "snow_hills"    : "hills";
  else                                                biome = isWinter ? "snow_mountain" : "mountain";

  return { biome, elevation, moisture, seasonVal };
}
```

### Biome colour palette
| Biome | Colour |
|-------|--------|
| ocean | #1a6fa8 |
| frozen_sea | #a8d8f0 |
| plains | #7ab648 |
| snow_plains | #e8f4f8 |
| forest | #2d6a2d |
| flood_plain | #5b8fa8 |
| hills | #9aaf6a |
| snow_hills | #ccd8c8 |
| mountain | #8a7a6a |
| snow_mountain | #f0ece8 |

### Season indicator
A radial progress arc around the epoch slider labelled Spring / Summer / Autumn / Winter — computed from `epoch % TICKS_PER_YEAR`.

---

## System 3 — NPC Schedules & Patrol Routes

### What it shows
A small 20×15 town grid. Six named NPCs each have a **deterministic home** (ZeroBytes static) and a **daily destination** (Zero-Temporal). Their current map position is interpolated based on time-of-day within the epoch.

### Data model
```js
function positionHash(id, salt = 0) {
  // Static — never changes regardless of epoch
}

function npcSchedule(npcId, epoch) {
  const tod        = epoch % TICKS_PER_DAY;               // 0–23
  const dayIndex   = Math.floor(epoch / TICKS_PER_DAY);
  const homeSeed   = positionHash(npcId, WORLD_SEED + 300);
  const home       = { x: Math.floor(hashToFloat(homeSeed) * 18) + 1,
                       y: Math.floor(hashToFloat(homeSeed >>> 16) * 13) + 1 };
  const daySeed    = temporalHash(npcId, 0, 0, dayIndex, WORLD_SEED + 400);
  const dest       = { x: Math.floor(hashToFloat(daySeed) * 18) + 1,
                       y: Math.floor(hashToFloat(daySeed >>> 16) * 13) + 1 };
  const errand     = ["Market","Temple","Tavern","Fields","Workshop"][
                       Math.floor(hashToFloat(daySeed >>> 32) * 5)];

  let activity, location;
  if      (tod < 6)  { activity = "Sleeping";        location = home; }
  else if (tod < 8)  { activity = "Morning Routine"; location = home; }
  else if (tod < 12) { activity = errand;             location = interpolate(home, dest, (tod-8)/4); }
  else if (tod < 17) { activity = errand;             location = dest; }
  else if (tod < 20) { activity = "Returning";        location = interpolate(dest, home, (tod-17)/3); }
  else               { activity = "Evening";          location = home; }

  return { activity, location, home, dest, errand };
}
```

### NPCs
| ID | Name | Role |
|----|------|------|
| 1 | Aldric | Blacksmith |
| 2 | Mira | Herbalist |
| 3 | Tor | Town Guard |
| 4 | Elise | Merchant |
| 5 | Brennan | Farmer |
| 6 | Syla | Priestess |

Each NPC is rendered as a coloured dot on the grid. Hovering shows their name, current activity, and destination.

---

## UI Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  ZERO-TEMPORAL DEMO                               [WORLD SEED: 42] │
│  ─────────────────────────────────────────────────────────────── │
│  EPOCH:  [━━━━━━●━━━━━━━━━━]  Day 214  Hour 14  Year 1  Summer   │
│                                                                    │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────┐  │
│  │  WEATHER GRID    │  │  TERRAIN STATE   │  │  NPC TOWN MAP  │  │
│  │  12×8 tiles      │  │  16×10 tiles     │  │  20×15 grid    │  │
│  │                  │  │                  │  │                │  │
│  │  [tile grid]     │  │  [terrain map]   │  │  [town grid]   │  │
│  │                  │  │                  │  │  ● Aldric      │  │
│  └──────────────────┘  └──────────────────┘  └────────────────┘  │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  NPC SCHEDULE TABLE  (updates live with epoch)             │  │
│  │  Name      │ Location   │ Activity   │ Destination         │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Zero-Temporal Properties Demonstrated

| Property | Demonstrated By |
|----------|----------------|
| O(1) epoch access | Drag slider to epoch 50,000 — instant result |
| World-time isolation | Same epoch always gives same map regardless of when you load the page |
| Temporal coherence | Adjacent epochs produce smooth transitions |
| Spatial-temporal hierarchy | Storms span 4×4 tile chunks; daily weather spans 2×2 chunks |
| Static + dynamic separation | Terrain elevation never changes; biome overlay does |
| Historical query | Slider left extreme = epoch 0, past is freely queryable |

---

## Implementation Notes

- **No xxhash dependency**: Use a 64-bit FNV-1a adapted to JS `BigInt` — output range and distribution are indistinguishable for demo purposes.
- **Coherent noise**: Implement bilinear interpolation between four position-hashed corners for smooth terrain and weather fields.
- **Animation mode**: Auto-advance button increments epoch by 1 every 80ms — shows weather and NPC movement as smooth animated simulation with zero stored state.
- **Epoch scrubbing**: Slider range 0–17520 (2 in-game years × 8760 ticks/year). Display as `Year X · Day Y · Hour Z`.
