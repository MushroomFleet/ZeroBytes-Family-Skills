---
name: zero-graph
description: Topology-is-seed procedural graph generation methodology. Extends ZeroBytes and Zero-Quadratic from point and pair properties into complete node-edge graph structures generated deterministically from a region coordinate. Use when a developer asks for procedural dungeon wing layouts, settlement road networks, quest dependency graphs, neural-style connection diagrams, cave system topologies, river delta branching, faction alliance webs, subway or trade network layouts, or any system where a graph structure (nodes + edges) must be generated deterministically without storing an adjacency list or node registry. Triggers on phrases like "procedural graph", "deterministic topology", "dungeon layout graph", "road network generation", "quest dependency structure", "connection diagram", "node-edge generation", "zero-graph", "topology from seed", or when a Zero-Quadratic system needs to also generate the node set itself, not just edge properties between pre-existing nodes.
---

# Zero-Graph: Topology-is-Seed Procedural Graph Generation

Extends [ZeroBytes](../zerobytes/SKILL.md) and [Zero-Quadratic](../zero-quadratic/SKILL.md) from point and pair properties into complete graph structures. The region coordinate IS the seed for both node existence and edge topology. No adjacency list, no node registry, no stored structure — the graph reconstructs on demand from its seed.

**The key distinction:** Zero-Quadratic queries edge properties between *pre-existing* nodes. Zero-Graph generates the node set and the topology simultaneously from a single region seed.

## The Five Extended Laws

1. **O(1) Region Access**: The complete node set and edge structure of a region are computable from the region coordinate — no stored graph required
2. **Internal Consistency**: Edge existence is determined by `pair_hash` on node IDs derived from the *same region seed* — nodes and edges share a coherent generative origin
3. **Boundary Coherence**: Cross-region edges use hashes derived from both region seeds — boundaries are symmetric and seam-free
4. **Hierarchy**: Sub-graphs inherit structure from parent graphs; density character flows downward through scales
5. **Determinism**: Same region coordinate → same graph topology on all machines and query orders

## Core Pattern

```python
import struct
import xxhash

def position_hash(x, y, z, salt=0):
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqq', x, y, z))
    return h.intdigest()

def pair_hash(ax, ay, bx, by, salt=0):
    p1, p2 = sorted([(ax, ay), (bx, by)])
    h = xxhash.xxh64(seed=salt)
    h.update(struct.pack('<qqqq', p1[0], p1[1], p2[0], p2[1]))
    return h.intdigest()

def hash_to_float(h):
    return (h & 0xFFFFFFFF) / 0x100000000

def generate_graph(region_x, region_y, region_seed,
                   max_nodes=16, region_size=100.0, edge_threshold=0.55):
    """
    Generate a complete node-edge graph from a region coordinate.
    Deterministic and fully reconstructable from (region_x, region_y, region_seed).
    """
    region_s = position_hash(region_x, region_y, 0, region_seed)
    n_nodes  = 2 + int(hash_to_float(region_s) * (max_nodes - 2))

    nodes = []
    for i in range(n_nodes):
        node_s = position_hash(region_s, i, 0, 0)
        nodes.append({
            "id":    i,
            "x":     hash_to_float(node_s) * region_size,
            "y":     hash_to_float(node_s >> 16) * region_size,
            "type":  int(hash_to_float(node_s >> 32) * 4),
            "value": hash_to_float(node_s >> 48)
        })

    edges = []
    for a in range(n_nodes):
        for b in range(a + 1, n_nodes):
            edge_seed = pair_hash(a, region_s & 0xFFFF, b, region_s & 0xFFFF, 0)
            strength  = hash_to_float(edge_seed)
            if strength > edge_threshold:
                dx = nodes[a]["x"] - nodes[b]["x"]
                dy = nodes[a]["y"] - nodes[b]["y"]
                edges.append({
                    "from": a, "to": b,
                    "strength": strength,
                    "distance": (dx**2 + dy**2) ** 0.5,
                    "type": int(hash_to_float(edge_seed >> 16) * 3)
                })

    return {"nodes": nodes, "edges": edges, "region": (region_x, region_y)}

def ensure_connectivity(graph, region_s):
    """Guarantee a spanning tree — no isolated nodes. Deterministic nearest-neighbour join."""
    nodes = graph["nodes"]; edges = graph["edges"]; n = len(nodes)
    parent = list(range(n))
    def find(x):
        while parent[x] != x: parent[x] = parent[parent[x]]; x = parent[x]
        return x
    def union(x, y): parent[find(x)] = find(y)
    for e in edges: union(e["from"], e["to"])
    for i in range(n):
        if find(i) != find(0):
            closest, min_dist = 0, float('inf')
            for j in range(n):
                if j != i and find(j) == find(0):
                    d = ((nodes[i]["x"]-nodes[j]["x"])**2 + (nodes[i]["y"]-nodes[j]["y"])**2)**0.5
                    if d < min_dist: min_dist = d; closest = j
            edges.append({"from": min(i,closest), "to": max(i,closest),
                          "strength": 0.5, "distance": min_dist, "type": 0, "forced": True})
            union(i, closest)
    graph["edges"] = edges
    return graph
```

## Quick Recipes

### Dungeon Wing Layout
```python
def dungeon_wing(wing_x, wing_y, floor, dungeon_seed, max_rooms=12):
    """Each node is a room; each edge is a corridor. Guaranteed connected."""
    wing_seed = position_hash(wing_x, wing_y, floor, dungeon_seed)
    graph     = generate_graph(wing_x, wing_y, wing_seed,
                               max_nodes=max_rooms, region_size=50.0, edge_threshold=0.6)
    graph     = ensure_connectivity(graph, wing_seed)
    TYPES     = ["empty","combat","treasure","boss","shop","puzzle"]
    for node in graph["nodes"]:
        node_s = position_hash(wing_seed, node["id"], floor, 0)
        node["room_type"]   = TYPES[node["type"] % len(TYPES)]
        node["difficulty"]  = 1.0 + floor * 0.3 + node["value"] * 0.5
        node["loot_tier"]   = int(hash_to_float(node_s >> 32) * 5)
        node["is_entrance"] = node["x"] == min(n["x"] for n in graph["nodes"])
    for edge in graph["edges"]:
        edge_s = pair_hash(edge["from"], wing_seed & 0xFFFF, edge["to"], wing_seed & 0xFFFF, 1)
        edge["has_door"] = hash_to_float(edge_s) > 0.4
        edge["locked"]   = hash_to_float(edge_s >> 16) > 0.8
    return graph
```

### Settlement Road Network
```python
def road_network(region_x, region_y, world_seed, max_settlements=10):
    """Nodes are settlements; edges are roads."""
    graph = generate_graph(region_x, region_y, world_seed,
                           max_nodes=max_settlements, region_size=200.0, edge_threshold=0.5)
    TYPES = ["hamlet","village","town","city","fortress"]
    for node in graph["nodes"]:
        node_s = position_hash(region_x, region_y, node["id"], world_seed + 1)
        node["settlement_type"] = TYPES[int(hash_to_float(node_s) * 5)]
        node["population"]      = int(hash_to_float(node_s >> 16) * 5000)
        node["fortified"]       = hash_to_float(node_s >> 32) > 0.7
    for edge in graph["edges"]:
        edge_s = pair_hash(edge["from"], region_x, edge["to"], region_y, world_seed + 2)
        edge["road_quality"] = hash_to_float(edge_s)
        edge["bandit_risk"]  = hash_to_float(edge_s >> 32)
    return graph
```

### Quest Dependency Graph
```python
def quest_graph(origin_x, origin_y, world_seed, max_objectives=8):
    """Directed DAG — lower-index objectives are prerequisites for higher-index ones."""
    region_s = position_hash(origin_x, origin_y, 0, world_seed + 5000)
    n_obj    = 2 + int(hash_to_float(region_s) * (max_objectives - 2))
    TYPES    = ["collect","kill","escort","investigate","deliver","protect"]
    objectives = []
    for i in range(n_obj):
        obj_s = position_hash(region_s, i, 0, 0)
        objectives.append({
            "id": i, "type": TYPES[int(hash_to_float(obj_s) * len(TYPES))],
            "difficulty": hash_to_float(obj_s >> 16),
            "optional":   hash_to_float(obj_s >> 32) > 0.75
        })
    dependencies = []
    for a in range(n_obj):
        for b in range(a + 1, n_obj):
            dep_s = position_hash(a, region_s & 0xFFFF, b, 0)
            if hash_to_float(dep_s) > 0.65:
                dependencies.append({"prerequisite": a, "unlocks": b,
                                     "strength": hash_to_float(dep_s >> 16)})
    return {"objectives": objectives, "dependencies": dependencies}
```

### Faction Alliance Web
```python
def faction_web(world_x, world_y, world_seed, region_size=500, max_factions=8):
    """Edges > 0.5 = alliance; < 0.5 = rivalry."""
    rx    = world_x // region_size;  ry = world_y // region_size
    graph = generate_graph(rx, ry, world_seed,
                           max_nodes=max_factions, region_size=float(region_size), edge_threshold=0.45)
    TYPES = ["military","mercantile","religious","scholarly","criminal","noble"]
    for node in graph["nodes"]:
        node_s = position_hash(rx, ry, node["id"], world_seed + 3000)
        node["faction_type"] = TYPES[int(hash_to_float(node_s) * len(TYPES))]
        node["power_level"]  = hash_to_float(node_s >> 16)
    for edge in graph["edges"]:
        edge_s = pair_hash(edge["from"], world_seed & 0xFFFF, edge["to"], world_seed & 0xFFFF, 1)
        raw = hash_to_float(edge_s)
        edge["relationship"] = "alliance" if raw > 0.5 else "rivalry"
        edge["intensity"]    = abs(raw - 0.5) * 2
        edge["is_secret"]    = hash_to_float(edge_s >> 16) > 0.8
    return graph
```

### Inter-Region Boundary Edges
```python
def inter_region_edges(region_a, region_b, world_seed, nodes_a, nodes_b, threshold=0.7):
    """Cross-region connections using both region seeds — symmetric at boundaries."""
    rax, ray = region_a;  rbx, rby = region_b
    cross = []
    for na in nodes_a:
        for nb in nodes_b:
            cross_seed = pair_hash(na["id"] + rax*1000, ray,
                                   nb["id"] + rbx*1000, rby, world_seed + 9999)
            if hash_to_float(cross_seed) > threshold:
                dist = ((na["x"]-nb["x"])**2 + (na["y"]-nb["y"])**2)**0.5
                cross.append({"from_region": region_a, "from_node": na["id"],
                              "to_region": region_b,   "to_node":   nb["id"],
                              "strength": hash_to_float(cross_seed), "distance": dist})
    return cross
```

## Hierarchy Pattern

```python
def hierarchical_subgraph(parent_rx, parent_ry, parent_seed,
                           child_offset_x, child_offset_y, max_nodes=8, region_size=25.0):
    """Child graph density inherits from parent graph density."""
    parent_graph    = generate_graph(parent_rx, parent_ry, parent_seed)
    parent_density  = len(parent_graph["edges"]) / max(1, len(parent_graph["nodes"]))
    child_threshold = max(0.3, 0.7 - parent_density * 0.05)  # dense parent → denser child
    child_seed      = position_hash(parent_rx + child_offset_x,
                                    parent_ry + child_offset_y,
                                    int(parent_density * 100), parent_seed)
    return generate_graph(parent_rx + child_offset_x, parent_ry + child_offset_y,
                          child_seed, max_nodes, region_size, child_threshold)
```

## Anti-Patterns

```python
# BAD: Storing the graph — accumulates forever
graph_db = {}
def get_graph(rx, ry):
    if (rx,ry) not in graph_db: graph_db[(rx,ry)] = generate_graph(rx, ry, seed)
    return graph_db[(rx,ry)]

# BAD: Non-deterministic node count
n_nodes = random.randint(3, 12)

# BAD: Platform-dependent hash for edges
def bad_edge(a, b): return hash((a["id"], b["id"])) > threshold

# BAD: Nodes and edges generated with independent seeds — incoherent
nodes = generate_nodes(region_x, region_y, seed_1)
edges = generate_edges(region_x, region_y, seed_2)

# GOOD: Single region seed drives both coherently
graph = generate_graph(region_x, region_y, world_seed)
graph = ensure_connectivity(graph, position_hash(region_x, region_y, 0, world_seed))
```

## Debugging Checklist

When graph structure differs across machines:
1. `pair_hash` must sort `[(a, region_s), (b, region_s)]` — verify sort is on full tuples
2. Node IDs must be stable integers derived from loop index, not object references
3. `ensure_connectivity` tie-breaking: nearest node selection is deterministic only if node order is stable

When graphs are too connected or too sparse:
- Sparse (dungeon corridors): `edge_threshold = 0.65–0.75`
- Dense (social webs, faction alliances): `edge_threshold = 0.40–0.55`

When inter-region connections feel asymmetric:
- Both region seeds must be present in the `inter_region_edges` hash — omitting one breaks symmetry

## Determinism Verification

```python
def verify_graph(region_x, region_y, world_seed, max_nodes=16):
    g1 = generate_graph(region_x, region_y, world_seed, max_nodes)
    g2 = generate_graph(region_x, region_y, world_seed, max_nodes)
    assert len(g1["nodes"]) == len(g2["nodes"]), "Node count non-deterministic!"
    assert len(g1["edges"]) == len(g2["edges"]), "Edge count non-deterministic!"
    for n1, n2 in zip(g1["nodes"], g2["nodes"]):
        assert abs(n1["x"] - n2["x"]) < 1e-9, f"Node position non-deterministic: {n1['id']}"
        assert n1["type"] == n2["type"],        f"Node type non-deterministic: {n1['id']}"
    node_ids = {n["id"] for n in g1["nodes"]}
    for e in g1["edges"]:
        assert e["from"] in node_ids and e["to"] in node_ids, "Edge references missing node!"
    g_conn = ensure_connectivity(g1, position_hash(region_x, region_y, 0, world_seed))
    parent = list(range(len(g_conn["nodes"])))
    def find(x):
        while parent[x] != x: x = parent[x]
        return x
    for e in g_conn["edges"]:
        p = find(e["from"]); parent[p] = find(e["to"])
    assert len({find(i) for i in range(len(g_conn["nodes"]))}) == 1, "Graph not fully connected!"
```

## Usage

1. **Define the region** — the spatial unit that contains one graph; scale to dungeon wing, city district, or world region
2. **Set max_nodes** — hard-cap N at design time; controls O(N²) edge generation cost
3. **Choose edge_threshold** — higher = sparser; lower = denser
4. **Apply ensure_connectivity** — run if isolated nodes are unacceptable (e.g. all rooms reachable)
5. **Enrich nodes and edges** — layer game-specific properties via `position_hash` on node IDs
6. **Handle boundaries** — `inter_region_edges` for cross-region connections; include both region seeds
7. **Compose with hierarchy** — child sub-graphs inherit density character from parent
8. **Verify** — run `verify_graph` confirming determinism, consistency, and connectivity

**Core principle:** A graph is not a data structure — it is a property of a region. Zero-Graph generates structure and topology together from a single seed. No adjacency lists. No node registries. No stored edges. The graph springs complete from its coordinate.
