---
name: zero-graph
description: Topology-is-seed procedural graph generation methodology. Extends ZeroBytes and Zero-Quadratic from point and pair properties into complete node-edge graph structures generated deterministically from a region coordinate. Use when a developer asks for procedural dungeon wing layouts, settlement road networks, quest dependency graphs, neural-style connection diagrams, cave system topologies, river delta branching, faction alliance webs, subway or trade network layouts, or any system where a graph structure (nodes + edges) must be generated deterministically without storing an adjacency list or node registry. Triggers on phrases like "procedural graph", "deterministic topology", "dungeon layout graph", "road network generation", "quest dependency structure", "connection diagram", "node-edge generation", "zero-graph", "topology from seed", or when a Zero-Quadratic system needs to also generate the node set itself, not just edge properties between pre-existing nodes.
---

# Zero-Graph: Topology-is-Seed Procedural Graph Generation

Extends [ZeroBytes](zerobytes/SKILL.md) and [Zero-Quadratic](zero-quadratic/SKILL.md) from point and pair properties into complete graph structures. The region coordinate IS the seed for both node existence and edge topology. No adjacency list, no node registry, no stored structure — the graph is reconstructed on demand from its seed.

## The Conceptual Leap from Zero-Quadratic

Zero-Quadratic asks: *What is the relationship between two pre-existing entities?*
Zero-Graph asks: *What nodes exist in this region, and which of them connect?*

Zero-Quadratic assumes you already know which entities exist and queries the edge between them. Zero-Graph generates the entity set and the graph topology simultaneously from a single region seed. This is the distinction between *edge properties* (Zero-Quadratic's domain) and *graph structure* (Zero-Graph's domain).

**The key insight:** A graph's topology — how many nodes, where they are, which connect — is a property of the region that contains it, not a property of individual pairs. Seeding the graph at the region level, then deriving nodes and edges from that seed, produces complete deterministic graph structures with zero storage.

## The Five Extended Laws

Every procedural graph system must satisfy:

1. **O(1) Region Access**: The complete node set and edge structure of a region are computable from the region coordinate without loading or iterating any stored graph
2. **Internal Consistency**: Edge existence is determined by pair_hash on node IDs derived from the *same region seed* — nodes and edges share a coherent generative origin
3. **Boundary Coherence**: Nodes near region boundaries that should logically connect to adjacent regions use inter-region edge hashing derived from both region seeds
4. **Hierarchy**: Sub-graphs inherit structure from parent graphs; a dungeon wing graph inherits from the dungeon graph; a trade route network within a city inherits from the regional road network
5. **Determinism**: Same region coordinate → same graph topology (same nodes, same edges) across all machines and query orders

## Core Pattern

```python
import struct
import math
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
                   max_nodes=16, region_size=100.0,
                   edge_threshold=0.55):
    """
    Generate a complete node-edge graph from a region coordinate.
    Returns: {"nodes": [...], "edges": [...]}
    Both are deterministic and fully reconstructable from (region_x, region_y, region_seed).
    """
    region_s = position_hash(region_x, region_y, 0, region_seed)

    # Step 1: Generate node count and positions
    n_nodes = 2 + int(hash_to_float(region_s) * (max_nodes - 2))
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

    # Step 2: Generate edges via pair_hash on node IDs (Zero-Quadratic layer)
    edges = []
    for a in range(n_nodes):
        for b in range(a + 1, n_nodes):
            edge_seed = pair_hash(a, region_s & 0xFFFF, b, region_s & 0xFFFF, 0)
            strength  = hash_to_float(edge_seed)
            if strength > edge_threshold:
                # Ensure connectivity: check spatial distance
                dx = nodes[a]["x"] - nodes[b]["x"]
                dy = nodes[a]["y"] - nodes[b]["y"]
                dist = (dx**2 + dy**2) ** 0.5
                edges.append({
                    "from":     a,
                    "to":       b,
                    "strength": strength,
                    "distance": dist,
                    "type":     int(hash_to_float(edge_seed >> 16) * 3)
                })

    return {"nodes": nodes, "edges": edges, "region": (region_x, region_y)}

def ensure_connectivity(graph, region_s):
    """
    Guarantee at least a spanning tree — no isolated nodes.
    Uses deterministic minimum spanning tree via sorted edge weights.
    """
    nodes = graph["nodes"]
    edges = graph["edges"]
    n = len(nodes)

    # Find connected components via union-find
    parent = list(range(n))
    def find(x):
        while parent[x] != x: parent[x] = parent[parent[x]]; x = parent[x]
        return x
    def union(x, y): parent[find(x)] = find(y)

    for e in edges: union(e["from"], e["to"])

    # For any isolated node, deterministically connect to nearest existing node
    for i in range(n):
        if find(i) != find(0):
            # Find closest already-connected node
            min_dist = float('inf')
            closest  = 0
            for j in range(n):
                if j != i and find(j) == find(0):
                    dx = nodes[i]["x"] - nodes[j]["x"]
                    dy = nodes[i]["y"] - nodes[j]["y"]
                    d  = (dx**2 + dy**2) ** 0.5
                    if d < min_dist:
                        min_dist = d; closest = j
            edges.append({
                "from": min(i, closest), "to": max(i, closest),
                "strength": 0.5, "distance": min_dist, "type": 0, "forced": True
            })
            union(i, closest)

    graph["edges"] = edges
    return graph
```

## Quick Recipes

### Dungeon Wing Layout
```python
def dungeon_wing(wing_x, wing_y, floor, dungeon_seed, max_rooms=12):
    """
    Generate a dungeon wing as a room-connection graph.
    Each node is a room; each edge is a doorway/corridor.
    """
    wing_seed = position_hash(wing_x, wing_y, floor, dungeon_seed)
    graph = generate_graph(wing_x, wing_y, wing_seed,
                           max_nodes=max_rooms, region_size=50.0, edge_threshold=0.6)
    graph = ensure_connectivity(graph, wing_seed)

    ROOM_TYPES = ["empty", "combat", "treasure", "boss", "shop", "puzzle"]

    # Enrich nodes with room-specific properties
    for node in graph["nodes"]:
        node_s = position_hash(wing_seed, node["id"], floor, 0)
        node["room_type"]  = ROOM_TYPES[node["type"] % len(ROOM_TYPES)]
        node["difficulty"] = 1.0 + floor * 0.3 + node["value"] * 0.5
        node["loot_tier"]  = int(hash_to_float(node_s >> 32) * 5)
        # Special room: entrance is the leftmost node
        node["is_entrance"] = node["x"] == min(n["x"] for n in graph["nodes"])

    # Enrich edges with corridor properties
    for edge in graph["edges"]:
        edge_s = pair_hash(edge["from"], wing_seed & 0xFFFF,
                           edge["to"],   wing_seed & 0xFFFF, 1)
        edge["has_door"] = hash_to_float(edge_s) > 0.4
        edge["locked"]   = hash_to_float(edge_s >> 16) > 0.8

    return graph
```

### Settlement Road Network
```python
def road_network(region_x, region_y, world_seed, max_settlements=10):
    """
    Generate a settlement-road network for a map region.
    Nodes are settlements; edges are roads.
    """
    graph = generate_graph(region_x, region_y, world_seed,
                           max_nodes=max_settlements, region_size=200.0, edge_threshold=0.5)

    SETTLEMENT_TYPES = ["hamlet", "village", "town", "city", "fortress"]

    for node in graph["nodes"]:
        node_s = position_hash(region_x, region_y, node["id"], world_seed + 1)
        # Settlement type influenced by position and region character
        terrain_val = hash_to_float(node_s)
        node["settlement_type"] = SETTLEMENT_TYPES[int(terrain_val * 5)]
        node["population"]      = int(hash_to_float(node_s >> 16) * 5000)
        node["fortified"]       = hash_to_float(node_s >> 32) > 0.7

    for edge in graph["edges"]:
        edge_s = pair_hash(edge["from"], region_x, edge["to"], region_y, world_seed + 2)
        edge["road_quality"] = hash_to_float(edge_s)           # 0 = dirt track, 1 = paved road
        edge["has_bridge"]   = hash_to_float(edge_s >> 16) > 0.7
        edge["bandit_risk"]  = hash_to_float(edge_s >> 32)

    return graph
```

### Quest Dependency Graph
```python
def quest_graph(quest_origin_x, quest_origin_y, world_seed, max_objectives=8):
    """
    Generate a quest as a directed dependency graph.
    Nodes are objectives; directed edges are "must complete A before B".
    Edge directionality makes this a DAG (directed acyclic graph).
    """
    region_s = position_hash(quest_origin_x, quest_origin_y, 0, world_seed + 5000)
    n_obj = 2 + int(hash_to_float(region_s) * (max_objectives - 2))

    OBJECTIVE_TYPES = ["collect", "kill", "escort", "investigate", "deliver", "protect"]
    objectives = []
    for i in range(n_obj):
        obj_s = position_hash(region_s, i, 0, 0)
        objectives.append({
            "id":   i,
            "type": OBJECTIVE_TYPES[int(hash_to_float(obj_s) * len(OBJECTIVE_TYPES))],
            "difficulty": hash_to_float(obj_s >> 16),
            "optional":   hash_to_float(obj_s >> 32) > 0.75
        })

    # Directed dependencies: lower-index objectives tend to be prerequisites
    # Use asymmetric hash to determine direction; threshold to create DAG
    dependencies = []
    for a in range(n_obj):
        for b in range(a + 1, n_obj):
            dep_seed = position_hash(a, region_s & 0xFFFF, b, 0)
            if hash_to_float(dep_seed) > 0.65:
                # a must be completed before b (forward dependency, no cycle)
                dependencies.append({
                    "prerequisite": a,
                    "unlocks":      b,
                    "strength":     hash_to_float(dep_seed >> 16)
                })

    return {"objectives": objectives, "dependencies": dependencies,
            "origin": (quest_origin_x, quest_origin_y)}
```

### Faction Alliance Web
```python
def faction_web(world_x, world_y, world_seed, region_size=500, max_factions=8):
    """
    Generate a faction alliance/rivalry web for a world region.
    Positive edge strength = alliance; negative = rivalry.
    """
    rx = world_x // region_size
    ry = world_y // region_size
    graph = generate_graph(rx, ry, world_seed,
                           max_nodes=max_factions, region_size=float(region_size), edge_threshold=0.45)

    FACTION_TYPES = ["military", "mercantile", "religious", "scholarly", "criminal", "noble"]

    for node in graph["nodes"]:
        node_s = position_hash(rx, ry, node["id"], world_seed + 3000)
        node["faction_type"]  = FACTION_TYPES[int(hash_to_float(node_s) * len(FACTION_TYPES))]
        node["power_level"]   = hash_to_float(node_s >> 16)
        node["stability"]     = hash_to_float(node_s >> 32)

    for edge in graph["edges"]:
        edge_s = pair_hash(edge["from"], world_seed & 0xFFFF,
                           edge["to"],   world_seed & 0xFFFF, 1)
        # Strength > 0.5 = alliance; < 0.5 = rivalry
        raw = hash_to_float(edge_s)
        edge["relationship"]  = "alliance" if raw > 0.5 else "rivalry"
        edge["intensity"]     = abs(raw - 0.5) * 2   # 0 = neutral, 1 = intense
        edge["is_secret"]     = hash_to_float(edge_s >> 16) > 0.8

    return graph
```

### Inter-Region Graph (Boundary Crossing Edges)
```python
def inter_region_edges(region_a, region_b, world_seed, nodes_a, nodes_b,
                       boundary_threshold=0.7):
    """
    Generate edges that cross region boundaries.
    Uses both region seeds to produce boundary-coherent cross-region connections.
    """
    rax, ray = region_a; rbx, rby = region_b
    cross_edges = []

    # Only consider boundary nodes (those near the shared edge)
    for node_a in nodes_a:
        for node_b in nodes_b:
            # Cross-region pair hash: uses both region positions as context
            cross_seed = pair_hash(
                node_a["id"] + rax * 1000, ray,
                node_b["id"] + rbx * 1000, rby,
                world_seed + 9999
            )
            if hash_to_float(cross_seed) > boundary_threshold:
                dist = ((node_a["x"] - node_b["x"])**2 +
                        (node_a["y"] - node_b["y"])**2) ** 0.5
                cross_edges.append({
                    "from_region": region_a, "from_node": node_a["id"],
                    "to_region":   region_b, "to_node":   node_b["id"],
                    "strength":    hash_to_float(cross_seed),
                    "distance":    dist
                })
    return cross_edges
```

## Hierarchy Pattern

ZeroBytes hierarchy: `parent_seed → child_seed via position`
Zero-Graph adds: `parent_graph → child_graph via subregion coordinate`

```python
def hierarchical_subgraph(parent_region_x, parent_region_y, parent_seed,
                          child_offset_x, child_offset_y,
                          max_nodes=8, region_size=25.0):
    """
    A sub-graph whose node count and connectivity inherit from the parent graph's character.
    Dense parent graphs produce denser child graphs.
    """
    parent_graph = generate_graph(parent_region_x, parent_region_y, parent_seed)
    parent_density = len(parent_graph["edges"]) / max(1, len(parent_graph["nodes"]))

    # Dense parent → lower edge threshold in child (more connections)
    child_threshold = max(0.3, 0.7 - parent_density * 0.05)

    child_seed = position_hash(parent_region_x + child_offset_x,
                               parent_region_y + child_offset_y,
                               int(parent_density * 100), parent_seed)

    return generate_graph(parent_region_x + child_offset_x,
                          parent_region_y + child_offset_y,
                          child_seed, max_nodes, region_size, child_threshold)
```

## Anti-Patterns

```python
# BAD: Storing the graph — defeats the purpose
graph_db = {}
def get_graph(rx, ry):
    if (rx,ry) not in graph_db:
        graph_db[(rx,ry)] = generate_graph(rx, ry, seed)   # accumulates forever
    return graph_db[(rx,ry)]

# BAD: Non-deterministic node count
import random
n_nodes = random.randint(3, 12)   # changes every call

# BAD: Platform-dependent hash for edge existence
def bad_edge(a, b): return hash((a["id"], b["id"])) > threshold  # hash() is platform-dependent

# BAD: Generating nodes and edges independently with different seeds
nodes = generate_nodes(region_x, region_y, seed_1)
edges = generate_edges(region_x, region_y, seed_2)  # no coherent relationship

# GOOD: Single region seed drives both nodes and edges coherently
graph = generate_graph(region_x, region_y, world_seed)
graph = ensure_connectivity(graph, position_hash(region_x, region_y, 0, world_seed))
```

## Debugging Checklist

When graph structure differs across machines:
1. Check `pair_hash` uses sorted coordinates — `pair_hash(a, region_s, b, region_s)` must sort `[(a, region_s), (b, region_s)]` correctly
2. Check node ID stability — node IDs must be stable integers, not object references or hashes of mutable state
3. Check `ensure_connectivity` union-find for platform-dependent tie-breaking in minimum distance selection

When graphs feel "too connected" or "too sparse":
- Adjust `edge_threshold` — higher = sparser graph, lower = denser
- For sparse graphs (dungeon corridors), use 0.65–0.75
- For dense graphs (social networks, faction webs), use 0.40–0.55

When inter-region connections feel wrong:
- Verify that both region seeds are included in the `inter_region_edges` hash — missing one makes boundaries non-symmetric
- Check that boundary node selection is spatially correct for the shared edge direction

## Determinism Verification

```python
def verify_graph(region_x, region_y, world_seed, max_nodes=16):
    """Verify graph structure is deterministic and internally consistent."""
    # Multiple calls must produce identical graphs
    g1 = generate_graph(region_x, region_y, world_seed, max_nodes)
    g2 = generate_graph(region_x, region_y, world_seed, max_nodes)

    assert len(g1["nodes"]) == len(g2["nodes"]), "Node count non-deterministic!"
    assert len(g1["edges"]) == len(g2["edges"]), "Edge count non-deterministic!"

    for n1, n2 in zip(g1["nodes"], g2["nodes"]):
        assert abs(n1["x"] - n2["x"]) < 1e-9, f"Node position non-deterministic: {n1['id']}"
        assert n1["type"] == n2["type"], f"Node type non-deterministic: {n1['id']}"

    for e1, e2 in zip(sorted(g1["edges"], key=lambda e: (e["from"], e["to"])),
                      sorted(g2["edges"], key=lambda e: (e["from"], e["to"]))):
        assert e1["from"] == e2["from"] and e1["to"] == e2["to"], "Edge non-deterministic!"

    # Internal consistency: all edge node IDs must reference existing nodes
    node_ids = {n["id"] for n in g1["nodes"]}
    for edge in g1["edges"]:
        assert edge["from"] in node_ids, f"Edge references non-existent node {edge['from']}"
        assert edge["to"]   in node_ids, f"Edge references non-existent node {edge['to']}"

    # Connectivity
    g_conn = ensure_connectivity(g1, position_hash(region_x, region_y, 0, world_seed))
    parent = list(range(len(g_conn["nodes"])))
    def find(x):
        while parent[x] != x: x = parent[x]; return x
    for e in g_conn["edges"]:
        p = find(e["from"]); q = find(e["to"]); parent[p] = q
    roots = {find(i) for i in range(len(g_conn["nodes"]))}
    assert len(roots) == 1, f"Graph not fully connected: {len(roots)} components"

    print(f"Zero-Graph verification passed: {len(g1['nodes'])} nodes, {len(g1['edges'])} edges.")
```

## Usage

When implementing a Zero-Graph system:

1. **Define the region** — the spatial unit that contains one graph; size depends on game scale (dungeon wing, city district, world region)
2. **Set max_nodes** — hard-cap N at design time; this controls O(N²) edge generation cost
3. **Choose edge_threshold** — higher = sparser (dungeon corridors); lower = denser (social webs)
4. **Apply ensure_connectivity** — if isolated nodes are unacceptable (all dungeon rooms must be reachable), always run the connectivity pass
5. **Enrich nodes and edges** — layer game-specific properties onto the structural graph using position_hash on node IDs
6. **Handle boundaries** — use `inter_region_edges` for cross-region connections; include both region seeds
7. **Compose with hierarchy** — child sub-graphs inherit density character from parent graph structure
8. **Verify** — run `verify_graph` confirming determinism, internal consistency, and connectivity

**Core principle:** A graph is not a data structure — it is a property of a region. Zero-Graph generates the structure and the topology together from a single seed. No adjacency lists. No node registries. No stored edges. The graph springs complete from its coordinate.
