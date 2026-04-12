---
name: zero-grammar
description: ZeroBytes-seeded deterministic grammar expansion. Makes L-Systems, shape grammars, and graph grammars infinite, parallel, and zero-storage by hashing coordinates to select production rules. Use when ZeroBytes content feels statistically plausible but semantically incoherent — cities with no urban logic, dungeons that feel random not designed, plants that look identical, quests with no narrative arc. Triggers on "zero-grammar", "deterministic L-system", "seeded grammar", "authored feel procedurally", "infinite grammar expansion", "coordinate-seeded production rules", "grammar without storing trees", "procedural city layout", "quest graph generation", "building floor plan procgen", or when a developer says their world "looks random instead of designed". The single mechanism that unlocks all grammar types is select_rule(rules, x, y, depth, world_seed) — position replaces random.choice(). Always use when content needs to feel designed, not merely probable.
---

# Zero-Grammar: Deterministic Infinite Grammar Expansion

Extends [ZeroBytes](../zerobytes/SKILL.md) from *statistical* coherence into *semantic* coherence. ZeroBytes hashes coordinates to values. Zero-Grammar hashes coordinates to **grammar rule selections** — making any L-System, shape grammar, or graph grammar fully deterministic, parallelisable, and zero-storage.

## The Problem ZeroBytes Cannot Solve

ZeroBytes produces content that is statistically related to its neighbours but has no concept of meaning:

```
# ZeroBytes city — statistically plausible, semantically broken
(10,5)="commercial"  (11,5)="residential"
(10,6)="park"        (11,6)="industrial"
# Industrial next to residential next to park. No urban logic.

# Zero-Grammar city — semantically authored
CBD → Commercial ring → Residential ring → Industrial fringe
# Every node knows its grammar role. Structure feels designed.
```

Grammar systems encode *what things mean*. Zero-Grammar makes them infinite by replacing `random.choice(rules)` with `select_rule(rules, x, y, depth, world_seed)`.

## Grammar Types Supported

| Type | Use Case | Mechanism |
|---|---|---|
| L-System | Plants, coastlines, road fractals | Hash selects production rule per symbol per rewrite step |
| Shape Grammar | Architecture, buildings, ruins | Hash selects split axis, ratio, and child roles |
| Graph Grammar | Cities, dungeon wings, quest graphs | Hash selects which graph transform fires at each node |
| String Grammar | Narrative, dialogue trees | Hash selects production rule at each symbol expansion |

## Core Pattern: The One Change That Unlocks Everything

```python
import struct, xxhash
from dataclasses import dataclass
from typing import Any

def position_hash(x: int, y: int, depth: int, salt: int = 0) -> int:
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqq', x, y, depth))
    return h.intdigest()

def hash_to_float(h: int) -> float:
    return (h & 0xFFFFFFFF) / 0x100000000

@dataclass
class Production:
    symbol: str       # the symbol this rule rewrites
    result: Any       # expansion: list of symbols, transform name, etc.
    weight: float = 1.0

def select_rule(rules: list[Production], x: int, y: int,
                depth: int, world_seed: int) -> Production:
    """
    The core primitive. Replaces random.choice() in every grammar type.
    Same (x, y, depth, world_seed) → same rule, always, on every machine.
    """
    total = sum(r.weight for r in rules)
    v = hash_to_float(position_hash(x, y, depth, world_seed)) * total
    cumulative = 0.0
    for rule in rules:
        cumulative += rule.weight
        if v < cumulative:
            return rule
    return rules[-1]
```

This one function is the entire Zero-Grammar contribution. Every recipe below is just a different grammar type using it.

## Recipe 1: L-System (Plants, Coastlines, Road Fractals)

```python
@dataclass
class LSymbol:
    symbol: str
    x: int      # world position this symbol was born at
    y: int
    depth: int

PLANT_GRAMMAR = {
    "F": [
        Production("F", ["F","+","F","-","-","F","+","F"], weight=0.5),
        Production("F", ["F","-","F","+"  ,"+","F","-","F"], weight=0.3),
        Production("F", ["F","F"],                           weight=0.2),
    ],
    "+": [Production("+", ["+"])],
    "-": [Production("-", ["-"])],
}

MAX_LSYSTEM_DEPTH = 6   # declare before writing rules; controls output size

def expand_lsystem(axiom: list[LSymbol], grammar: dict,
                   world_seed: int) -> list[LSymbol]:
    """
    Expand deterministically. Each symbol's rule is selected by its
    world position — reproducible, parallelisable, no shared state.
    """
    current = axiom
    for depth in range(MAX_LSYSTEM_DEPTH):
        next_gen = []
        for sym in current:
            rules = grammar.get(sym.symbol)
            if not rules:
                next_gen.append(sym)
                continue
            chosen = select_rule(rules, sym.x, sym.y, depth, world_seed)
            # Children inherit parent x,y offset by sibling index
            for i, child in enumerate(chosen.result):
                next_gen.append(LSymbol(child, sym.x + i, sym.y, depth + 1))
        current = next_gen
    return current

# Grow a unique plant at any world coordinate:
axiom = [LSymbol("F", x=100, y=200, depth=0)]
plant = expand_lsystem(axiom, PLANT_GRAMMAR, world_seed=42)
# plant at (101, 200) will be different — same grammar, different position hash
```

## Recipe 2: Shape Grammar (Architecture, Buildings, Ruins)

Shape grammars subdivide bounding boxes recursively. Each split decision is driven by position hash.

```python
@dataclass
class Box:
    x: float; y: float; w: float; h: float
    symbol: str   # architectural role: "facade", "room", "courtyard"
    depth:  int
    gx: int; gy: int   # integer grid position for hashing

MAX_SHAPE_DEPTH = 5

SPLIT_ROLES = {
    "facade":    [("entrance", "facade_wing"), ("facade_wing", "facade_wing")],
    "room":      [("room", "corridor"),        ("room", "room")],
    "courtyard": [("courtyard", "garden"),     ("courtyard", "well")],
}

def split_box(box: Box, world_seed: int) -> list[Box]:
    if box.depth >= MAX_SHAPE_DEPTH or box.w < 2.0 or box.h < 2.0:
        return [box]   # terminal leaf

    # Three independent hashes — axis, ratio, child roles
    axis_h  = position_hash(box.gx, box.gy, box.depth,     world_seed)
    ratio_h = position_hash(box.gx, box.gy, box.depth + 1, world_seed)
    role_h  = position_hash(box.gx, box.gy, box.depth + 2, world_seed)

    horizontal = hash_to_float(axis_h) > 0.5
    ratio = 0.3 + hash_to_float(ratio_h) * 0.4   # split 30%–70%

    options  = SPLIT_ROLES.get(box.symbol, [("room", "room")])
    left_sym, right_sym = options[int(hash_to_float(role_h) * len(options))]

    if horizontal:
        sy = box.y + box.h * ratio
        return [Box(box.x, box.y, box.w, box.h*ratio,       left_sym,  box.depth+1, box.gx, box.gy),
                Box(box.x, sy,    box.w, box.h*(1-ratio),   right_sym, box.depth+1, box.gx, box.gy+1)]
    else:
        sx = box.x + box.w * ratio
        return [Box(box.x, box.y, box.w*ratio,     box.h,   left_sym,  box.depth+1, box.gx,   box.gy),
                Box(sx,    box.y, box.w*(1-ratio),  box.h,   right_sym, box.depth+1, box.gx+1, box.gy)]

def generate_building(gx: int, gy: int, world_seed: int,
                      w: float = 20.0, h: float = 20.0) -> list[Box]:
    """
    Unique floor plan at every grid position. No two buildings alike.
    Spatial completeness is guaranteed: leaf boxes exactly tile the
    original bounding box with no gaps and no overlaps — the subdivision
    approach makes floating walls structurally impossible.
    """
    root = Box(0.0, 0.0, w, h, "facade", depth=0, gx=gx, gy=gy)
    queue, leaves = [root], []
    while queue:
        box = queue.pop()
        children = split_box(box, world_seed)
        if len(children) == 1 and children[0] is box:
            leaves.append(box)
        else:
            queue.extend(children)
    return leaves
```

## Recipe 3: Graph Grammar (City Layout, Quest Graphs)

Graph grammars rewrite nodes into sub-graphs. Zero-Grammar seeds the transform choice at each node.

```python
from dataclasses import dataclass, field

@dataclass
class GNode:
    id: int; symbol: str; x: int; y: int

@dataclass
class GEdge:
    a: int; b: int; kind: str

@dataclass
class CityGraph:
    nodes: list[GNode] = field(default_factory=list)
    edges: list[GEdge] = field(default_factory=list)

CITY_RULES = {
    "CBD": [
        Production("CBD", "ring_expand",  weight=0.6),
        Production("CBD", "dense_expand", weight=0.4),
    ],
    "residential": [
        Production("residential", "subdivide", weight=0.7),
        Production("residential", "add_park",  weight=0.3),
    ],
}

MAX_CITY_GENERATIONS = 3

def expand_city_node(node: GNode, transform: str,
                     world_seed: int) -> tuple[list[GNode], list[GEdge]]:
    """Expand one node using the chosen transform. Position-seeded."""
    new_nodes, new_edges = [], []
    if transform in ("ring_expand", "dense_expand"):
        for i, (dx, dy) in enumerate([(0,1),(0,-1),(1,0),(-1,0)]):
            child_x, child_y = node.x + dx, node.y + dy
            nid = node.id * 100 + i
            h   = position_hash(child_x, child_y, 0, world_seed)
            sym = "residential" if hash_to_float(h) > 0.3 else "industrial"
            new_nodes.append(GNode(nid, sym, child_x, child_y))
            new_edges.append(GEdge(node.id, nid, "road"))
    # add more transforms here — "subdivide", "add_park", etc.
    return new_nodes, new_edges

def generate_city(cx: int, cy: int, world_seed: int) -> CityGraph:
    """City graph rooted at (cx, cy). Unique per world coordinate."""
    graph = CityGraph(nodes=[GNode(0, "CBD", cx, cy)])
    for gen in range(MAX_CITY_GENERATIONS):
        additions_n, additions_e = [], []
        for node in graph.nodes:
            rules = CITY_RULES.get(node.symbol, [])
            if not rules: continue
            rule = select_rule(rules, node.x, node.y, gen, world_seed)
            nn, ne = expand_city_node(node, rule.result, world_seed)
            additions_n.extend(nn); additions_e.extend(ne)
        graph.nodes.extend(additions_n); graph.edges.extend(additions_e)
    return graph
```

## Composing with ZeroBytes: Grammar Selection

ZeroBytes determines *which grammar* governs each region in O(1). Zero-Grammar handles the expansion.

```python
def world_grammar_type(wx: int, wy: int, world_seed: int) -> str:
    """O(1) — pure ZeroBytes, never expands a grammar."""
    h = position_hash(wx // 64, wy // 64, 0, world_seed)
    elev = hash_to_float(h)
    mois = hash_to_float(h ^ 0xDEADBEEF)
    if elev < 0.2: return "coastal_city"
    if elev > 0.8: return "mountain_ruins"
    if mois > 0.7: return "forest_organic"
    return "plains_settlement"

# Then dispatch to the right generator:
# "coastal_city"    → generate_city(wx, wy, world_seed)
# "mountain_ruins"  → generate_building(..., symbol="ruin")
# "forest_organic"  → expand_lsystem(axiom, PLANT_GRAMMAR, world_seed)
```

## Depth Budget

Declare `MAX_DEPTH` before writing rules. It controls output size and prevents runaway expansion.

| Grammar | Max Depth | Typical Output |
|---|---|---|
| L-System (plant) | 6–8 | ~200–500 symbols |
| Shape (building) | 4–6 | 20–100 leaf boxes |
| Graph (city) | 3–4 | 50–200 nodes |
| String (quest) | 3–5 | 10–50 steps |

## The Five Zero-Grammar Laws

1. **Position seeds rule selection** — `select_rule(rules, x, y, depth, world_seed)` is the only source of choice
2. **Depth is declared, not emergent** — `MAX_DEPTH` is a constant; grammar cannot run forever
3. **Children inherit parent position** — world position flows down the tree; siblings are offset by index
4. **Grammar is pure data** — rules and weights stored once; no expansion trees stored
5. **Determinism is verifiable** — `generate(x, y, seed) == generate(x, y, seed)` always

## Anti-Patterns

```python
# BAD: Platform random — non-deterministic
random.choice(grammar["F"])

# BAD: Unbounded depth — stack overflow
def expand(sym, depth=0): return expand(chosen, depth+1)

# BAD: Storing expansions — defeats zero-storage
tree[(x,y,depth)] = chosen_rule

# BAD: Sibling order dependency
for node in nodes:
    node.role = depends_on(previous_node.role)   # order-dependent

# GOOD: Position-seeded, bounded, stateless
rule = select_rule(grammar[sym], x, y, depth, world_seed)
children = rule.result if depth < MAX_DEPTH else [terminal]
```

## Debugging Checklist

**Same output everywhere?**
- Verify `depth` increments through each rewrite generation
- Verify sibling children get distinct `x` values (offset by index `i`)
- Check `select_rule` is receiving `depth`, not always `0`

**Output differs across machines?**
- Replace `random.choice()` with `select_rule()`
- Replace Python `hash()` (platform-dependent) with `xxhash`
- Replace `time.time()` or `id(obj)` seeds with `position_hash()`

**Expansion too slow / too large?**
- Reduce `MAX_DEPTH` — output grows exponentially with depth
- Add terminal conditions: `if w < MIN_SIZE: return [box]`
- For graph grammar: add proximity guard — only expand nodes within `max_radius`

## Composing with the Zero Family

```
ZeroBytes        → world properties at a point             O(1)
Zero-Quadratic   → relationships between points            O(N²)
Zero-Grammar     → semantic structure grown from a point   O(branching^depth)
Zero-WFC         → tile-level validity within a chunk      O(chunk²) cached
```

Zero-Grammar produces structure; Zero-WFC makes it walkable. They compose naturally: grammar defines room roles, WFC resolves the tile grid within each room.

**Core principle:** ZeroBytes encodes where things *are*. Zero-Grammar encodes what they *mean*. Meaning springs from coordinates — infinitely, without storing a single expansion tree.
