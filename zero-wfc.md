---
name: zero-wfc
description: ZeroBytes-seeded Wave Function Collapse hybrid methodology. Infinite-scale deterministic world generation with local structural validity guarantees. Use when a developer needs both infinite O(1) world access (from ZeroBytes) AND globally coherent local structures — connected dungeons, valid road networks, plausible building interiors, river systems that always reach the sea, cave systems with guaranteed exits. Triggers on phrases like "valid dungeon generation", "connected rooms", "guaranteed solvable", "structurally coherent chunks", "WFC seeded by coordinates", "zero-wfc", "infinite WFC world", "constraint-valid procgen", "locally valid globally infinite", or when a ZeroBytes system produces plausible-but-disconnected content (terrain that never forms rivers, rooms that never connect). Always use this skill when the user's ZeroBytes system needs *validity guarantees*, not just statistical plausibility.
---

# Zero-WFC: Infinite Scale + Local Structural Validity

Extends [ZeroBytes](../zerobytes/SKILL.md) with [Wave Function Collapse](https://github.com/mxgmn/WaveFunctionCollapse) constraint solving. ZeroBytes handles the infinite outer world; WFC handles the bounded inner chunk — guaranteeing that local structures are *actually valid*, not just statistically plausible.

## The Core Problem ZeroBytes Cannot Solve

ZeroBytes generates each tile independently. This means:

```
# ZeroBytes dungeon — statistically plausible, structurally broken
(0,0) = "room"    (1,0) = "wall"   (2,0) = "room"
(0,1) = "wall"    (1,1) = "door"   (2,1) = "wall"
(0,2) = "room"    (1,2) = "wall"   (2,2) = "room"
# The door at (1,1) connects to nothing. The rooms are islands.
```

WFC solves this: it treats the chunk as a constraint-satisfaction problem and propagates adjacency rules globally across the chunk before committing any tile.

## Architecture: Two Layers

```
┌──────────────────────────────────────────────────────────────┐
│  LAYER 1: ZeroBytes (Infinite, O(1), Zero Storage)           │
│  Answers: "What KIND of chunk is at chunk coordinate (cx,cy)?│
│  Outputs: chunk_seed, biome_type, constraint_profile_id      │
└──────────────────────────────┬───────────────────────────────┘
                               │ chunk_seed + profile
┌──────────────────────────────▼───────────────────────────────┐
│  LAYER 2: WFC (Bounded, Coherent, Locally Valid)             │
│  Answers: "What is the EXACT layout of this chunk?"          │
│  Inputs:  chunk_seed, adjacency rules, edge constraints      │
│  Outputs: fully connected, constraint-valid tile grid        │
└──────────────────────────────────────────────────────────────┘
```

**Key insight:** Only Layer 2 runs when the player enters a chunk. Layer 1 answers all macro-scale queries (biome, region type, faction) in O(1) without ever running WFC.

## Core Implementation

```python
import struct, xxhash, random
from enum import IntEnum

# ─── ZeroBytes Layer ───────────────────────────────────────────

def chunk_seed(cx: int, cy: int, world_seed: int) -> int:
    h = xxhash.xxh64(seed=world_seed)
    h.update(struct.pack('<qq', cx, cy))
    return h.intdigest()

def hash_to_float(h: int) -> float:
    return (h & 0xFFFFFFFF) / 0x100000000

class BiomeType(IntEnum):
    DUNGEON   = 0
    CAVE      = 1
    RUINS     = 2
    FOREST    = 3

def chunk_biome(cx: int, cy: int, world_seed: int) -> BiomeType:
    """O(1) macro query. Never runs WFC."""
    seed = chunk_seed(cx, cy, world_seed)
    elevation = hash_to_float(seed)
    moisture  = hash_to_float(seed ^ 0xDEADBEEF)
    if elevation < 0.3:   return BiomeType.CAVE
    if moisture  > 0.7:   return BiomeType.FOREST
    if elevation > 0.75:  return BiomeType.RUINS
    return BiomeType.DUNGEON

# ─── Edge Constraint Negotiation ───────────────────────────────

def chunk_edge_constraints(cx: int, cy: int, world_seed: int) -> dict:
    """
    Deterministically compute what tile types must appear on each
    edge of this chunk — so neighbouring chunks can match borders.
    Called by BOTH the chunk and its neighbours. O(1).
    """
    def edge_seed(ecx, ecy, direction, wseed):
        h = xxhash.xxh64(seed=wseed + direction * 9999)
        h.update(struct.pack('<qq', ecx, ecy))
        return h.intdigest()

    OPEN  = "open"   # corridor / passage must exist on this edge
    WALL  = "wall"   # solid wall on this edge
    ANY   = "any"    # WFC decides freely

    def edge_type(ecx, ecy, d, wseed):
        v = hash_to_float(edge_seed(ecx, ecy, d, wseed))
        return OPEN if v > 0.6 else WALL if v < 0.2 else ANY

    return {
        "north": edge_type(cx, cy-1, 0, world_seed),  # neighbour's south
        "south": edge_type(cx, cy,   1, world_seed),
        "east":  edge_type(cx,   cy, 2, world_seed),
        "west":  edge_type(cx-1, cy, 3, world_seed),  # neighbour's east
    }

# ─── WFC Layer ─────────────────────────────────────────────────

class Tile(IntEnum):
    WALL   = 0
    FLOOR  = 1
    DOOR   = 2
    WATER  = 3

# Adjacency rules: tile → set of tiles allowed in each direction
# Format: {tile: {direction: allowed_set}}
DUNGEON_RULES = {
    Tile.WALL:  {"N": {Tile.WALL, Tile.FLOOR, Tile.DOOR},
                 "S": {Tile.WALL, Tile.FLOOR, Tile.DOOR},
                 "E": {Tile.WALL, Tile.FLOOR, Tile.DOOR},
                 "W": {Tile.WALL, Tile.FLOOR, Tile.DOOR}},
    Tile.FLOOR: {"N": {Tile.FLOOR, Tile.WALL, Tile.DOOR},
                 "S": {Tile.FLOOR, Tile.WALL, Tile.DOOR},
                 "E": {Tile.FLOOR, Tile.WALL, Tile.DOOR},
                 "W": {Tile.FLOOR, Tile.WALL, Tile.DOOR}},
    Tile.DOOR:  {"N": {Tile.FLOOR},  # door must connect floor on both sides
                 "S": {Tile.FLOOR},
                 "E": {Tile.FLOOR},
                 "W": {Tile.FLOOR}},
}

BIOME_RULES = {
    BiomeType.DUNGEON: DUNGEON_RULES,
    BiomeType.CAVE:    DUNGEON_RULES,   # swap for cave-specific rules
    BiomeType.RUINS:   DUNGEON_RULES,   # swap for ruins-specific rules
    BiomeType.FOREST:  DUNGEON_RULES,   # swap for forest-specific rules
}

BIOME_WEIGHTS = {
    BiomeType.DUNGEON: {Tile.WALL: 0.45, Tile.FLOOR: 0.50, Tile.DOOR: 0.05},
    BiomeType.CAVE:    {Tile.WALL: 0.60, Tile.FLOOR: 0.38, Tile.DOOR: 0.02},
    BiomeType.RUINS:   {Tile.WALL: 0.40, Tile.FLOOR: 0.50, Tile.DOOR: 0.10},
    BiomeType.FOREST:  {Tile.WALL: 0.20, Tile.FLOOR: 0.80, Tile.DOOR: 0.00},
}

def generate_chunk(cx: int, cy: int, world_seed: int,
                   chunk_size: int = 16) -> list[list[Tile]]:
    """
    Generate a single chunk using ZeroBytes-seeded WFC.
    Returns a chunk_size x chunk_size tile grid, guaranteed valid.
    """
    cseed  = chunk_seed(cx, cy, world_seed)
    biome  = chunk_biome(cx, cy, world_seed)
    edges  = chunk_edge_constraints(cx, cy, world_seed)
    rules  = BIOME_RULES[biome]
    weights = BIOME_WEIGHTS[biome]

    rng = random.Random(cseed)   # seeded — fully deterministic

    N = chunk_size
    DIRS = {"N": (0,-1), "S": (0,1), "E": (1,0), "W": (-1,0)}
    OPPOSITE = {"N": "S", "S": "N", "E": "W", "W": "E"}

    # Wave: each cell holds set of possible tiles
    wave = [[set(Tile) for _ in range(N)] for _ in range(N)]

    # ── Apply edge constraints from ZeroBytes ──────────────────
    edge_positions = {
        "N": [(x, 0)     for x in range(N)],
        "S": [(x, N-1)   for x in range(N)],
        "E": [(N-1, y)   for y in range(N)],
        "W": [(0,   y)   for y in range(N)],
    }
    for direction, constraint in edges.items():
        if constraint == "open":
            # Force at least one FLOOR or DOOR on this edge
            mid = N // 2
            positions = edge_positions[direction]
            for pos in [positions[mid]]:
                x, y = pos
                wave[y][x] = {Tile.FLOOR}
        elif constraint == "wall":
            for x, y in edge_positions[direction]:
                wave[y][x] = {Tile.WALL}

    # ── WFC collapse loop ───────────────────────────────────────
    def entropy(cell_set):
        return len(cell_set)

    def propagate(wave, x, y):
        stack = [(x, y)]
        while stack:
            cx_, cy_ = stack.pop()
            current = wave[cy_][cx_]
            for d, (dx, dy) in DIRS.items():
                nx_, ny_ = cx_ + dx, cy_ + dy
                if not (0 <= nx_ < N and 0 <= ny_ < N):
                    continue
                neighbor = wave[ny_][nx_]
                allowed = set()
                for tile in current:
                    allowed |= rules.get(tile, {}).get(d, set(Tile))
                new_neighbor = neighbor & allowed
                if new_neighbor != neighbor:
                    if not new_neighbor:
                        return False  # contradiction
                    wave[ny_][nx_] = new_neighbor
                    stack.append((nx_, ny_))
        return True

    max_attempts = 10
    for attempt in range(max_attempts):
        attempt_rng = random.Random(cseed + attempt * 0x1337)
        attempt_wave = [row[:] for row in [
            [set(c) for c in row] for row in wave
        ]]

        collapsed = set()
        contradiction = False

        while len(collapsed) < N * N:
            # Find lowest-entropy uncollapsed cell
            min_ent = float('inf')
            candidates = []
            for y in range(N):
                for x in range(N):
                    if (x, y) in collapsed:
                        continue
                    e = entropy(attempt_wave[y][x])
                    if e < min_ent:
                        min_ent = e
                        candidates = [(x, y)]
                    elif e == min_ent:
                        candidates.append((x, y))

            if not candidates:
                break

            x, y = attempt_rng.choice(candidates)
            options = list(attempt_wave[y][x])
            tile_weights = [weights.get(t, 0.1) for t in options]
            chosen = attempt_rng.choices(options, weights=tile_weights)[0]
            attempt_wave[y][x] = {chosen}
            collapsed.add((x, y))

            if not propagate(attempt_wave, x, y):
                contradiction = True
                break

        if not contradiction:
            return [[list(attempt_wave[y][x])[0] for x in range(N)]
                    for y in range(N)]

    # Fallback: solid wall chunk with guaranteed floor border
    return [[Tile.FLOOR if (x == 1 or y == 1 or x == N-2 or y == N-2)
             else Tile.WALL
             for x in range(N)] for y in range(N)]
```

## Chunk Caching Strategy

WFC is expensive. Cache generated chunks; use ZeroBytes for all non-layout queries.

```python
from functools import lru_cache

@lru_cache(maxsize=256)
def get_chunk(cx: int, cy: int, world_seed: int) -> tuple:
    """Cache generated chunks. LRU evicts cold chunks automatically."""
    grid = generate_chunk(cx, cy, world_seed)
    return tuple(tuple(row) for row in grid)  # hashable for cache

def get_tile(wx: int, wy: int, world_seed: int,
             chunk_size: int = 16) -> Tile:
    """World-space tile lookup. Generates chunk on first access."""
    cx, cy = wx // chunk_size, wy // chunk_size
    lx, ly = wx % chunk_size, wy % chunk_size
    chunk = get_chunk(cx, cy, world_seed)
    return chunk[ly][lx]

def get_biome(wx: int, wy: int, world_seed: int,
              chunk_size: int = 16) -> BiomeType:
    """O(1) biome query. Never touches WFC."""
    return chunk_biome(wx // chunk_size, wy // chunk_size, world_seed)
```

## Constraint Profile System

Different chunk types carry different WFC rulesets, selected by ZeroBytes:

```python
@dataclass
class ConstraintProfile:
    rules:          dict          # adjacency rules
    weights:        dict          # tile spawn weights
    min_open_edges: int           # minimum edges that must be "open"
    required_tiles: list[Tile]    # tiles that MUST appear at least once
    forbidden_tiles: list[Tile]   # tiles that must NOT appear

PROFILES = {
    "dungeon_room":   ConstraintProfile(..., min_open_edges=2, ...),
    "dungeon_corridor": ConstraintProfile(..., min_open_edges=2, ...),
    "cave_deep":      ConstraintProfile(..., min_open_edges=1, ...),
    "ruins_open":     ConstraintProfile(..., min_open_edges=4, ...),
}

def select_profile(cx: int, cy: int, world_seed: int) -> str:
    seed = chunk_seed(cx, cy, world_seed)
    biome = chunk_biome(cx, cy, world_seed)
    v = hash_to_float(seed ^ 0xCAFEBABE)
    if biome == BiomeType.DUNGEON:
        return "dungeon_corridor" if v > 0.5 else "dungeon_room"
    if biome == BiomeType.CAVE:
        return "cave_deep"
    return "ruins_open"
```

## Query Decision Table

| Query | Use | Cost |
|---|---|---|
| "What biome is at world pos X,Y?" | ZeroBytes only | O(1) |
| "What faction controls region?" | ZeroBytes only | O(1) |
| "What tile is at world pos X,Y?" | WFC (cached) | O(chunk²) once |
| "Is this dungeon solvable?" | WFC guarantee | Free — always true |
| "How many rooms in this chunk?" | Post-process WFC output | O(chunk²) |

## The Five Zero-WFC Laws

1. **ZeroBytes answers scale** — Biome, faction, region properties never touch WFC
2. **WFC answers structure** — Tile layout, room connectivity, validity
3. **Edge constraints bridge them** — Computed O(1) by ZeroBytes; consumed by WFC
4. **Cache aggressively** — A generated chunk is permanent; never regenerate it
5. **Fallback is always valid** — If WFC contradicts, emit a safe default chunk

## Anti-Patterns

```python
# BAD: Running WFC for every tile query
def get_tile(wx, wy):
    chunk = generate_chunk(...)   # WFC runs every call — O(N²) per tile
    return chunk[wy % N][wx % N]

# BAD: Non-deterministic WFC seed
def generate_chunk(cx, cy):
    rng = random.Random()         # Different every run — breaks reproducibility

# BAD: No edge constraint negotiation
# Chunks will have mismatched borders — walls facing open corridors

# BAD: Unbounded chunk size
CHUNK_SIZE = 512                  # WFC is O(N² log N) — 512² is very slow

# GOOD: Bounded chunks, seeded RNG, cached output, edge negotiation
chunk = get_chunk(cx, cy, world_seed)   # Cached WFC, ZeroBytes seeded
biome = get_biome(wx, wy, world_seed)   # Pure ZeroBytes — O(1)
```

## Recommended Chunk Sizes

| Use Case | Chunk Size | WFC Time (approx) |
|---|---|---|
| Roguelike rooms | 8×8 – 16×16 | < 1ms |
| Dungeon wings | 16×16 – 32×32 | 1–10ms |
| City blocks | 32×32 | 5–20ms |
| Region terrain | 64×64 | 50–200ms (generate during load) |

## Usage

When implementing a Zero-WFC system:

1. **Design the ZeroBytes layer first** — establish world_seed, biome types, region properties
2. **Define adjacency rules per biome** — what tiles can be next to what
3. **Design edge constraint vocabulary** — what edge states are meaningful (open/wall/water/road)
4. **Set chunk size** — smaller = faster WFC, larger = richer structures
5. **Implement caching** — WFC output is permanent; regenerating wastes time
6. **Test edge matching** — walk across chunk borders and verify no seams

**Core principle:** ZeroBytes makes the world infinite. WFC makes it habitable. Neither works as well alone.
