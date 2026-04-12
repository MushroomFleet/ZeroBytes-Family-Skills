---
name: zero-wfc
description: ZeroBytes-seeded Wave Function Collapse hybrid. Infinite O(1) world scale with local structural validity guarantees. Use when a ZeroBytes system produces statistically plausible but structurally broken content — rooms that never connect, corridors to nowhere, rivers that stop mid-chunk. Triggers on "valid dungeon generation", "connected rooms", "guaranteed solvable", "zero-wfc", "infinite WFC world", "constraint-valid procgen", "WFC seeded by coordinates", or any description of content that "looks right but is broken". The skill adds a WFC layer inside each bounded chunk while ZeroBytes handles all infinite-scale O(1) queries; edge constraints bridge the two layers so neighbouring chunks always match at their borders. Always use when the developer needs validity guarantees, not just statistical plausibility.
---

# Zero-WFC: Infinite Scale + Local Structural Validity

Extends [ZeroBytes](../zerobytes/SKILL.md) with Wave Function Collapse constraint solving. ZeroBytes governs the infinite outer world; WFC governs the bounded inner chunk — guaranteeing local structures are *actually valid*, not just statistically plausible.

## The Problem ZeroBytes Cannot Solve

ZeroBytes generates each tile independently. This breaks structural integrity:

```
# ZeroBytes dungeon — statistically plausible, structurally broken
(0,0)=room  (1,0)=wall  (2,0)=room
(0,1)=wall  (1,1)=door  (2,1)=wall   ← door connects to nothing
(0,2)=room  (1,2)=wall  (2,2)=room   ← rooms are islands
```

WFC treats the chunk as constraint-satisfaction: adjacency rules propagate globally before any tile commits, so corridors connect, rivers flow, buildings have entrances.

## Two-Layer Architecture

```
┌─────────────────────────────────────────────────────────┐
│  LAYER 1: ZeroBytes  (Infinite · O(1) · Zero Storage)   │
│  Query: "What KIND of chunk is at (cx, cy)?"            │
│  Output: chunk_seed · biome_type · edge_constraints     │
└───────────────────────────┬─────────────────────────────┘
                            │  chunk_seed + edge profile
┌───────────────────────────▼─────────────────────────────┐
│  LAYER 2: WFC  (Bounded · Coherent · Locally Valid)      │
│  Query: "What is the EXACT layout of this chunk?"        │
│  Input: chunk_seed · adjacency rules · edge constraints  │
│  Output: fully connected, constraint-valid tile grid     │
└─────────────────────────────────────────────────────────┘
```

Layer 1 answers all macro queries O(1) — biome, faction, region — without ever invoking WFC. Layer 2 runs only when a chunk is first loaded, then cached permanently.

## Core Implementation

```python
import struct, xxhash, random
from enum import IntEnum

# ── Layer 1: ZeroBytes ────────────────────────────────────────

def chunk_seed(cx: int, cy: int, world_seed: int) -> int:
    h = xxhash.xxh64(seed=world_seed)
    h.update(struct.pack('<qq', cx, cy))
    return h.intdigest()

def hash_to_float(h: int) -> float:
    return (h & 0xFFFFFFFF) / 0x100000000

class BiomeType(IntEnum):
    DUNGEON = 0; CAVE = 1; RUINS = 2; FOREST = 3

def chunk_biome(cx: int, cy: int, world_seed: int) -> BiomeType:
    """O(1) — never runs WFC."""
    s = chunk_seed(cx, cy, world_seed)
    elev = hash_to_float(s); mois = hash_to_float(s ^ 0xDEADBEEF)
    if elev < 0.3: return BiomeType.CAVE
    if mois > 0.7: return BiomeType.FOREST
    if elev > 0.75: return BiomeType.RUINS
    return BiomeType.DUNGEON

# ── Edge Constraint Negotiation ───────────────────────────────
# The bridge between layers. Both a chunk AND its neighbours call
# this with the same arguments — guaranteeing matching borders.
# O(1) — pure hash, no WFC.

def chunk_edge_constraints(cx: int, cy: int, world_seed: int) -> dict:
    OPEN, WALL, ANY = "open", "wall", "any"
    def ev(ecx, ecy, d):
        h = xxhash.xxh64(seed=world_seed + d * 9999)
        h.update(struct.pack('<qq', ecx, ecy))
        v = hash_to_float(h.intdigest())
        return OPEN if v > 0.6 else WALL if v < 0.2 else ANY
    return {
        "north": ev(cx, cy-1, 0),   # matches neighbour's south
        "south": ev(cx,  cy,  1),
        "east":  ev(cx,  cy,  2),
        "west":  ev(cx-1, cy, 3),   # matches neighbour's east
    }

# ── Layer 2: WFC ──────────────────────────────────────────────

class Tile(IntEnum):
    WALL = 0; FLOOR = 1; DOOR = 2; WATER = 3

# Adjacency rules: what tiles are legal neighbours in each direction.
# Extend this dict per biome — cave rules differ from city rules.
RULES = {
    Tile.WALL:  {d: {Tile.WALL, Tile.FLOOR, Tile.DOOR} for d in "NSEW"},
    Tile.FLOOR: {d: {Tile.WALL, Tile.FLOOR, Tile.DOOR, Tile.WATER} for d in "NSEW"},
    Tile.DOOR:  {d: {Tile.FLOOR} for d in "NSEW"},    # door requires floor both sides
    Tile.WATER: {d: {Tile.WATER, Tile.FLOOR} for d in "NSEW"},  # water flows or shores
}

WEIGHTS = {Tile.WALL: 0.42, Tile.FLOOR: 0.48, Tile.DOOR: 0.05, Tile.WATER: 0.05}

DIRS = {"N": (0,-1), "S": (0,1), "E": (1,0), "W": (-1,0)}

def generate_chunk(cx: int, cy: int, world_seed: int,
                   N: int = 16) -> list[list[Tile]]:
    """
    WFC-generate one NxN chunk. Seeded by ZeroBytes chunk_seed.
    Edge constraints from chunk_edge_constraints are applied first,
    so neighbouring chunks always match at their shared border.
    """
    cseed = chunk_seed(cx, cy, world_seed)
    edges = chunk_edge_constraints(cx, cy, world_seed)

    edge_cells = {
        "N": [(x, 0)   for x in range(N)],
        "S": [(x, N-1) for x in range(N)],
        "E": [(N-1, y) for y in range(N)],
        "W": [(0,   y) for y in range(N)],
    }

    for attempt in range(10):
        rng  = random.Random(cseed + attempt * 0x1337)
        wave = [[set(Tile) for _ in range(N)] for _ in range(N)]

        # Apply edge constraints from ZeroBytes before WFC starts
        for side, constraint in edges.items():
            if constraint == "open":
                mid = N // 2
                x, y = edge_cells[side][mid]
                wave[y][x] = {Tile.FLOOR}
            elif constraint == "wall":
                for x, y in edge_cells[side]:
                    wave[y][x] = {Tile.WALL}

        def propagate(wave, sx, sy):
            stack = [(sx, sy)]
            while stack:
                cx_, cy_ = stack.pop()
                for d, (dx, dy) in DIRS.items():
                    nx_, ny_ = cx_+dx, cy_+dy
                    if not (0 <= nx_ < N and 0 <= ny_ < N): continue
                    allowed = set()
                    for t in wave[cy_][cx_]: allowed |= RULES[t][d]
                    new = wave[ny_][nx_] & allowed
                    if not new: return False
                    if new != wave[ny_][nx_]:
                        wave[ny_][nx_] = new
                        stack.append((nx_, ny_))
            return True

        collapsed, contradiction = set(), False
        while len(collapsed) < N * N and not contradiction:
            best, cands = N*N+1, []
            for y in range(N):
                for x in range(N):
                    if (x,y) in collapsed: continue
                    e = len(wave[y][x])
                    if e < best: best, cands = e, [(x,y)]
                    elif e == best: cands.append((x,y))
            if not cands: break
            x, y = rng.choice(cands)
            opts = list(wave[y][x])
            wave[y][x] = {rng.choices(opts, weights=[WEIGHTS.get(t,.1) for t in opts])[0]}
            collapsed.add((x,y))
            if not propagate(wave, x, y): contradiction = True

        if not contradiction:
            return [[list(wave[y][x])[0] for x in range(N)] for y in range(N)]

    # Safe fallback: always a valid (if boring) chunk
    return [[Tile.FLOOR if x==1 or y==1 or x==N-2 or y==N-2 else Tile.WALL
             for x in range(N)] for y in range(N)]
```

## Caching — Essential, Not Optional

WFC is O(N² log N). Cache every generated chunk; never regenerate.

```python
from functools import lru_cache

@lru_cache(maxsize=256)
def get_chunk(cx: int, cy: int, world_seed: int) -> tuple:
    grid = generate_chunk(cx, cy, world_seed)
    return tuple(tuple(row) for row in grid)

def get_tile(wx: int, wy: int, world_seed: int, N: int = 16) -> Tile:
    """World-to-tile. Generates chunk on first access, cached thereafter."""
    chunk = get_chunk(wx // N, wy // N, world_seed)
    return chunk[wy % N][wx % N]

def get_biome(wx: int, wy: int, world_seed: int, N: int = 16) -> BiomeType:
    """O(1) — never touches WFC."""
    return chunk_biome(wx // N, wy // N, world_seed)
```

## Query Decision Table

| Question | Call | Cost |
|---|---|---|
| What biome is at world pos X,Y? | `get_biome()` | O(1) — ZeroBytes only |
| What faction controls region? | ZeroBytes directly | O(1) |
| What tile is at world pos X,Y? | `get_tile()` | O(N²) once, then O(1) cached |
| Is this dungeon solvable? | — | Free — WFC guarantees it |

## Recommended Chunk Sizes

| Use Case | N | WFC Time |
|---|---|---|
| Roguelike rooms | 8–16 | < 1ms |
| Dungeon wings | 16–32 | 1–10ms |
| City blocks | 32 | 5–20ms |
| Region terrain | 64 | 50–200ms (load screen) |

Keep N ≤ 32 for real-time generation. For N > 32, generate during loading.

## The Five Zero-WFC Laws

1. **ZeroBytes answers scale** — biome, faction, region never invoke WFC
2. **WFC answers structure** — tile layout, connectivity, room validity
3. **Edge constraints bridge them** — computed O(1) by ZeroBytes; consumed as WFC pre-constraints
4. **Cache is permanent** — a generated chunk never regenerates; evict only on memory pressure
5. **Fallback is always valid** — if WFC contradicts after 10 attempts, emit a safe default

## Anti-Patterns

```python
# BAD: Running WFC per tile query — O(N²) every call
def get_tile(wx, wy): return generate_chunk(wx//N, wy//N, seed)[wy%N][wx%N]

# BAD: Unseeded WFC RNG — breaks determinism
rng = random.Random()   # must be random.Random(chunk_seed(...))

# BAD: No edge negotiation — chunk borders will mismatch

# BAD: Oversized chunks — N=512 is 262K cells, WFC takes seconds

# GOOD: Cache + ZeroBytes edge constraints + bounded N
biome = get_biome(wx, wy, world_seed)   # O(1)
tile  = get_tile(wx, wy, world_seed)    # WFC once, cached forever
```

## Recipe: River Continuity Across Chunks

Rivers that stop mid-chunk are a classic ZeroBytes failure. Zero-WFC fixes this via edge constraints: if both a chunk and its neighbour see `"open_water"` on their shared border, WFC pre-seeds that border cell to WATER, and propagation ensures the river continues.

```python
def chunk_edge_constraints_river(cx, cy, world_seed):
    """Extended edge negotiation that uses WATER for river edges."""
    def ev(ecx, ecy, d):
        h = xxhash.xxh64(seed=world_seed + d * 9999)
        h.update(struct.pack('<qq', ecx, ecy))
        v = hash_to_float(h.intdigest())
        return "open_water" if v > 0.75 else "wall" if v < 0.2 else "any"
    return {
        "north": ev(cx, cy-1, 0), "south": ev(cx, cy, 1),
        "east":  ev(cx, cy,   2), "west":  ev(cx-1, cy, 3),
    }
# In generate_chunk: when constraint == "open_water", seed border cell to Tile.WATER
# WFC propagation ensures the river body flows coherently to that edge.
# The key: both chunks sharing a border call the same hash → always agree on river presence.
```

## Debugging Checklist

**Chunks don't match at borders?**
- Verify both chunks call `chunk_edge_constraints` with the same canonical arguments
- Check "north" uses `cy-1` (neighbour's coord), not `cy`

**WFC hits contradiction constantly?**
- Loosen adjacency rules — DOOR requiring FLOOR on both sides is strict
- Reduce N — smaller grids contradict less often
- Add weight to the most-flexible tile (FLOOR)

**Different output across machines?**
- Replace `random.random()` with `random.Random(chunk_seed(...))`
- Replace Python `hash()` (platform-dependent) with `xxhash`

## Composing with the Zero Family

```
ZeroBytes        → biome, elevation, moisture, faction    O(1)
Zero-Quadratic   → trade routes, faction tension          O(N²)
Zero-Temporal    → seasonal changes to biome              O(1)
Zero-WFC         → tile layout, room connectivity         O(chunk²) cached
```

Zero-WFC sits at the *leaf* of the hierarchy — it converts abstract world properties into concrete, walkable, guaranteed-valid tile grids.

**Core principle:** ZeroBytes makes the world infinite. WFC makes it habitable. The edge constraint layer makes them speak the same language.
