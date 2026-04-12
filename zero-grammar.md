---
name: zero-grammar
description: ZeroBytes-seeded deterministic grammar expansion methodology. Makes L-Systems, graph grammars, and production rule systems infinite, parallel, and zero-storage by using coordinate hashes to deterministically select which grammar rules fire at each position. Use when a developer needs semantically coherent structures — cities that grow plausibly, organic plant forms, procedural narrative quest graphs, dungeon layouts that feel architecturally authored, road networks with realistic topology — without storing expansion trees or running sequential simulations. Triggers on phrases like "zero-grammar", "deterministic L-system", "infinite grammar", "seeded grammar expansion", "procedural city grammar", "authored feel without storing data", "grammar without sequential expansion", "coordinate-seeded production rules", "ZeroBytes grammar", or when a ZeroBytes system produces content that is *statistically* plausible but *semantically* incoherent (terrain that has no geographical logic, dungeons with no architectural sense). Always use when the user wants content that feels *designed* rather than merely *probable*.
---

# Zero-Grammar: Deterministic Infinite Grammar Expansion

Extends [ZeroBytes](../zerobytes/SKILL.md) from statistical coherence into **semantic coherence**. Instead of hashing coordinates to tile types, Zero-Grammar hashes coordinates to *grammar production rule selections* — making the expansion of any L-System, graph grammar, or rewriting system fully deterministic, parallelisable, and zero-storage.

## The Core Problem ZeroBytes Cannot Solve

ZeroBytes produces content that is *statistically* related to its neighbours. A biome map is coherent because adjacent hashes produce similar elevation values. But it has no **semantic structure** — no concept of "a building should have a door", "a city grows from a centre", "a quest needs a resolution".

Grammar systems encode semantic rules. Zero-Grammar makes them infinite.

```
# ZeroBytes city: statistically plausible, semantically incoherent
(10, 5) = "commercial"    (11, 5) = "residential"
(10, 6) = "park"          (11, 6) = "industrial"
# Adjacent industrial and residential, park next to commercial — no urban logic.

# Zero-Grammar city: semantically authored
CityGrammar: CBD → [Commercial ring → Residential ring → Industrial fringe]
# Every city tile knows its grammar role. The structure is coherent.
```

## Grammar Types Supported

| Grammar Type | Use Case | Zero-Grammar Mechanism |
|---|---|---|
| L-Systems | Plants, coastlines, river deltas, road fractals | Hash selects stochastic production rule at each rewriting step |
| Graph Grammars | City layout, dungeon wings, quest structure | Hash selects which graph transformation rule fires at each node |
| Shape Grammars | Architecture, ruins, building interiors | Hash selects split axis and ratio at each subdivision step |
| String Grammars | Narrative generation, dialogue trees | Hash selects production rule at each symbol expansion |

## Core Pattern: Hash-Driven Rule Selection

```python
import struct, xxhash
from dataclasses import dataclass
from typing import Callable, Any

def position_hash(x: int, y: int, depth: int, salt: int = 0) -> int:
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqq', x, y, depth))
    return h.intdigest()

def hash_to_float(h: int) -> float:
    return (h & 0xFFFFFFFF) / 0x100000000

@dataclass
class Production:
    """A single grammar production rule with a selection weight."""
    symbol:   str            # what symbol this rule rewrites
    result:   Any            # the expansion (string, list, graph transform, etc.)
    weight:   float = 1.0   # relative probability weight

def select_rule(rules: list[Production], x: int, y: int,
                depth: int, world_seed: int) -> Production:
    """
    Deterministically select a production rule for position (x,y)
    at grammar depth `depth`. Same inputs → same rule, always.
    """
    total = sum(r.weight for r in rules)
    h = position_hash(x, y, depth, world_seed)
    v = hash_to_float(h) * total
    cumulative = 0.0
    for rule in rules:
        cumulative += rule.weight
        if v < cumulative:
            return rule
    return rules[-1]  # fallback
```

## Recipe 1: Infinite Stochastic L-System (Plants, Rivers, Roads)

```python
@dataclass
class LSymbol:
    symbol: str
    x:      int      # world position this symbol was born at
    y:      int
    depth:  int      # rewriting generation

# Define stochastic L-system rules
# Each symbol maps to a list of possible productions with weights
PLANT_GRAMMAR = {
    "F": [
        Production("F", ["F", "+", "F", "-", "-", "F", "+", "F"], weight=0.5),
        Production("F", ["F", "-", "F", "+", "+", "F", "-", "F"], weight=0.3),
        Production("F", ["F", "F"],                                weight=0.2),
    ],
    "+": [Production("+", ["+"], weight=1.0)],
    "-": [Production("-", ["-"], weight=1.0)],
}

def expand_lsystem(axiom: list[LSymbol], grammar: dict,
                   world_seed: int, max_depth: int = 5) -> list[LSymbol]:
    """
    Expand an L-system deterministically. Each symbol's expansion is
    selected by its world position hash — reproducible and parallelisable.
    """
    current = axiom
    for depth in range(max_depth):
        next_gen = []
        for sym in current:
            rules = grammar.get(sym.symbol)
            if not rules:
                next_gen.append(sym)
                continue
            chosen = select_rule(rules, sym.x, sym.y, depth, world_seed)
            # Propagate world position to children, incrementing depth
            for i, child_sym in enumerate(chosen.result):
                next_gen.append(LSymbol(child_sym, sym.x + i, sym.y, depth + 1))
        current = next_gen
    return current

# Usage: grow a plant at world position (100, 200)
axiom = [LSymbol("F", x=100, y=200, depth=0)]
plant = expand_lsystem(axiom, PLANT_GRAMMAR, world_seed=42)
```

## Recipe 2: Shape Grammar (Architecture, Buildings, Ruins)

Shape grammars subdivide shapes recursively. Zero-Grammar seeds each split deterministically by position.

```python
from dataclasses import dataclass

@dataclass
class Box:
    x: float; y: float; w: float; h: float
    symbol: str   # architectural role: "facade", "room", "courtyard", etc.
    depth:  int
    gx:     int   # grid position (integer chunk coords)
    gy:     int

def split_box(box: Box, world_seed: int) -> list[Box]:
    """
    Split a box into sub-boxes using deterministic hash-driven decisions.
    """
    if box.depth >= 5 or box.w < 2.0 or box.h < 2.0:
        return [box]   # terminal — leaf symbol

    ix, iy = int(box.gx), int(box.gy)

    # Hash selects: split axis
    axis_hash  = position_hash(ix, iy, box.depth,     world_seed)
    ratio_hash = position_hash(ix, iy, box.depth + 1, world_seed)
    role_hash  = position_hash(ix, iy, box.depth + 2, world_seed)

    split_horizontal = hash_to_float(axis_hash) > 0.5
    ratio = 0.3 + hash_to_float(ratio_hash) * 0.4   # split between 30%–70%

    SPLIT_ROLES = {
        "facade": [("facade_left", "facade_right"),
                   ("entrance",    "facade_right")],
        "room":   [("room",  "corridor"),
                   ("room",  "room")],
        "courtyard": [("courtyard", "garden"),
                      ("courtyard", "well")],
    }
    role_options = SPLIT_ROLES.get(box.symbol, [("room", "room")])
    idx = int(hash_to_float(role_hash) * len(role_options))
    left_role, right_role = role_options[idx]

    if split_horizontal:
        split_y = box.y + box.h * ratio
        return [
            Box(box.x, box.y,     box.w, box.h * ratio,        left_role,  box.depth+1, ix, iy),
            Box(box.x, split_y,   box.w, box.h * (1 - ratio), right_role, box.depth+1, ix, iy+1),
        ]
    else:
        split_x = box.x + box.w * ratio
        return [
            Box(box.x,    box.y, box.w * ratio,       box.h, left_role,  box.depth+1, ix,   iy),
            Box(split_x,  box.y, box.w * (1 - ratio), box.h, right_role, box.depth+1, ix+1, iy),
        ]

def generate_building(gx: int, gy: int, world_seed: int,
                      width: float = 20.0, height: float = 20.0) -> list[Box]:
    """Generate a complete building floor plan at grid position (gx, gy)."""
    root = Box(0.0, 0.0, width, height, "facade", depth=0, gx=gx, gy=gy)
    queue = [root]
    leaves = []
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

```python
from dataclasses import dataclass, field

@dataclass
class GNode:
    id:     int
    symbol: str   # "CBD", "residential", "industrial", "park", "port"
    x:      int
    y:      int

@dataclass
class GEdge:
    a: int; b: int   # node IDs
    kind: str        # "road", "rail", "pedestrian"

@dataclass
class CityGraph:
    nodes: list[GNode] = field(default_factory=list)
    edges: list[GEdge] = field(default_factory=list)

# Graph grammar rules: each matches a node symbol and transforms it
CITY_GRAPH_RULES = {
    "CBD": [
        # CBD expands to: keep CBD, add commercial ring nodes, connect them
        Production("CBD", "expand_cbd_ring",  weight=0.6),
        Production("CBD", "expand_cbd_dense", weight=0.4),
    ],
    "residential": [
        Production("residential", "subdivide_residential", weight=0.7),
        Production("residential", "add_park",              weight=0.3),
    ],
}

CITY_TRANSFORMS = {
    "expand_cbd_ring": lambda node, world_seed: _expand_cbd_ring(node, world_seed),
    "add_park":        lambda node, world_seed: _add_park(node, world_seed),
    # ... etc
}

def _expand_cbd_ring(node: GNode, world_seed: int) -> tuple[list[GNode], list[GEdge]]:
    """Add residential ring around CBD node. Position-seeded."""
    new_nodes, new_edges = [], []
    for i in range(4):   # N, S, E, W neighbours
        dx, dy = [(0,1),(0,-1),(1,0),(-1,0)][i]
        child_x, child_y = node.x + dx, node.y + dy
        nid = len(new_nodes) + node.id * 100 + i
        h = position_hash(child_x, child_y, 0, world_seed)
        sym = "residential" if hash_to_float(h) > 0.3 else "industrial"
        new_nodes.append(GNode(nid, sym, child_x, child_y))
        new_edges.append(GEdge(node.id, nid, "road"))
    return new_nodes, new_edges

def generate_city(cx: int, cy: int, world_seed: int,
                  generations: int = 3) -> CityGraph:
    """Generate a city graph rooted at chunk coordinate (cx, cy)."""
    root = GNode(0, "CBD", cx, cy)
    graph = CityGraph(nodes=[root])

    for gen in range(generations):
        new_nodes, new_edges = [], []
        for node in graph.nodes:
            rules = CITY_GRAPH_RULES.get(node.symbol, [])
            if not rules:
                continue
            rule = select_rule(rules, node.x, node.y, gen, world_seed)
            transform = CITY_TRANSFORMS.get(rule.result)
            if transform:
                nn, ne = transform(node, world_seed)
                new_nodes.extend(nn)
                new_edges.extend(ne)
        graph.nodes.extend(new_nodes)
        graph.edges.extend(new_edges)

    return graph
```

## Integrating with ZeroBytes

Zero-Grammar is most powerful when ZeroBytes determines *which grammar* to use:

```python
def world_grammar(wx: int, wy: int, world_seed: int) -> str:
    """O(1) ZeroBytes query: which grammar governs this world position?"""
    h = position_hash(wx // 64, wy // 64, 0, world_seed)
    elevation = hash_to_float(h)
    moisture  = hash_to_float(h ^ 0xDEADBEEF)

    if elevation < 0.2:   return "coastal_city"
    if elevation > 0.8:   return "mountain_ruins"
    if moisture  > 0.7:   return "forest_organic"
    return "plains_settlement"

GRAMMAR_MAP = {
    "coastal_city":      CITY_GRAPH_RULES,
    "mountain_ruins":    RUINS_SHAPE_GRAMMAR,
    "forest_organic":    PLANT_GRAMMAR,
    "plains_settlement": SETTLEMENT_GRAMMAR,
}

def generate_structure(wx: int, wy: int, world_seed: int):
    grammar_id = world_grammar(wx, wy, world_seed)
    grammar    = GRAMMAR_MAP[grammar_id]
    # Pass to appropriate generator...
```

## Depth Budget

Grammar expansion depth must be bounded at design time to prevent combinatorial explosion:

| Grammar | Max Depth | Output Size | Use Case |
|---|---|---|---|
| L-System (plant) | 6–8 | ~500 symbols | Single tree or shrub |
| Shape grammar (building) | 4–6 | 20–100 leaf boxes | Building floor plan |
| Graph grammar (city district) | 3–4 | 50–200 nodes | City district |
| String grammar (quest) | 3–5 | 10–50 steps | Quest outline |

**Rule:** `max_depth` is declared at design time, never grows at runtime.

## The Five Zero-Grammar Laws

1. **Position seeds rule selection** — `select_rule(rules, x, y, depth, world_seed)` is the only randomness source
2. **Depth is declared, not emergent** — `max_depth` is a constant; the grammar cannot run forever
3. **Children inherit parent position** — world position flows down the expansion tree; siblings get offset positions
4. **Grammar is pure data** — rules, weights, and transforms are stored once; expansions are not
5. **Determinism is verifiable** — `generate(x, y, seed) == generate(x, y, seed)` always

## Anti-Patterns

```python
# BAD: Global random state in grammar expansion
import random
def expand(symbol):
    return random.choice(grammar[symbol])   # Non-deterministic!

# BAD: Unbounded depth
def expand(sym, depth=0):
    return expand(chosen_rule, depth + 1)   # Stack overflow

# BAD: Storing the expansion tree
tree = {}
def expand(sym, x, y, depth):
    tree[(x,y,depth)] = chosen             # Defeats zero-storage purpose

# BAD: Sequential dependency between sibling nodes
def expand_city(nodes):
    for node in nodes:
        node.symbol = depends_on(previous_node.symbol)  # Order-dependent!

# GOOD: Position-seeded, bounded, stateless
rule = select_rule(grammar[symbol], x, y, depth, world_seed)
children = rule.result if depth < MAX_DEPTH else [terminal(symbol)]
```

## Debugging Checklist

When grammar output differs across runs:
1. Check for `random.random()` or `random.choice()` without `Random(seed)`
2. Check for `time.time()` or `id(obj)` as hash inputs
3. Check that child positions are deterministically derived from parent position

When grammar produces same output everywhere:
- Verify `depth` parameter increments correctly through recursion
- Check that sibling positions differ (offset by `i` or similar)

When expansion runs too long:
- Add `if depth >= MAX_DEPTH: return [terminal_symbol]` at every rule
- Hard-cap `MAX_DEPTH` as a module constant, not a parameter

## Usage

When implementing a Zero-Grammar system:

1. **Choose grammar type** — L-System for organic forms, shape grammar for architecture, graph grammar for networks, string grammar for narrative
2. **Write production rules with weights** — be explicit about what each symbol means semantically
3. **Bind rule selection to `select_rule`** — this is the only change from a traditional grammar
4. **Set `MAX_DEPTH`** — before writing any rules; depth determines output size budget
5. **Compose with ZeroBytes** — use ZeroBytes to select which grammar governs each region
6. **Verify** — `assert generate(x,y,seed) == generate(x,y,seed)` in your test suite

**Core principle:** Grammar systems encode what things *mean*. ZeroBytes encodes where things *are*. Zero-Grammar unifies them: meaning springs from coordinates, infinitely, without storing a single expansion.
