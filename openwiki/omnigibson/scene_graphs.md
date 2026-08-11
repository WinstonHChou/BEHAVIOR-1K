---
type: wiki-component
title: OmniGibson Scene Graphs
description: GraphBuilder — spatial relationship graphs for objects in a scene, used for task planning and state reasoning.
tags: [omnigibson, scene-graph, spatial-relationships, planning]
---

# OmniGibson Scene Graphs

**Scene graphs** represent spatial relationships between objects in a scene as a directed graph. They are used for task planning, state reasoning, and BDDL predicate evaluation.

## Class

```python
from omnigibson.scene_graphs.graph_builder import GraphBuilder
```

## GraphBuilder

**File:** `scene_graphs/graph_builder.py` (~14 KB)

**Purpose:** Constructs a directed graph where nodes are objects and edges represent spatial relationships.

### Graph Structure

```
GraphBuilder(scene)
├── nodes: dict[str, Node]  # Object name → Node
└── edges: list[Edge]       # (source, target, label)
```

### Node Structure

```python
class Node:
    name: str              # Object name (e.g., "bowl.n.01_1")
    category: str          # Object category (e.g., "bowl")
    position: np.ndarray   # Object position (N, 3)
    attributes: dict       # Additional attributes (color, material, etc.)
    neighbors: list[Node]  # Connected nodes
```

### Edge Structure

```python
class Edge:
    source: str            # Source object name
    target: str            # Target object name
    label: str             # Relationship type (e.g., "ontop", "nextto", "inside")
    attributes: dict       # Edge attributes (distance, offset, etc.)
```

## Building the Graph

```python
builder = GraphBuilder(scene)
builder.build()

# Access the graph
print(f"Objects: {len(builder.nodes)}")
print(f"Relationships: {len(builder.edges)}")

# Query relationships
for edge in builder.edges:
    print(f"{edge.source} --({edge.label})-> {edge.target}")
```

## Edge Types

Common relationship labels:

| Label | Predicate | Description |
|-------|-----------|-------------|
| `ontop` | OnTop | Object is on top of target |
| `nextto` | NextTo | Object is next to target |
| `under` | Under | Object is under target |
| `inside` | Inside | Object is inside target |
| `touching` | Touching | Objects are touching |
| `attached` | AttachedTo | Object is attached to target |
| `draped` | Draped | Cloth is draped over target |
| `adjacent_horizontal` | HorizontalAdjacency | Horizontal adjacency |
| `adjacent_vertical` | VerticalAdjacency | Vertical adjacency |

## Usage Examples

### Finding Objects on a Table

```python
builder = GraphBuilder(scene)
builder.build()

# Find all objects on top of the table
for edge in builder.edges:
    if edge.label == "ontop" and edge.target == "table.n.02_1":
        print(f"{edge.source} is on the table")
```

### Checking Reachability

```python
# Check if object is reachable from robot base
def is_reachable(graph, obj_name, robot_pos, threshold=1.0):
    for node in graph.nodes.values():
        if node.name == obj_name:
            distance = np.linalg.norm(node.position[:2] - robot_pos[:2])
            return distance < threshold
    return False
```

### BDDL Predicate Evaluation

Scene graphs can be used to efficiently evaluate BDDL predicates:

```python
def evaluate_predicate(graph, pred_cls, obj1_name, obj2_name=None):
    """Evaluate a predicate using the scene graph."""
    if pred_cls == OnTop:
        for edge in graph.edges:
            if edge.label == "ontop" and edge.source == obj1_name and edge.target == obj2_name:
                return True
    elif pred_cls == NextTo:
        for edge in graph.edges:
            if edge.label == "nextto" and (edge.source == obj1_name and edge.target == obj2_name or
                                            edge.source == obj2_name and edge.target == obj1_name):
                return True
    return False
```

## Testing

- `test_scene_graph.py` (~4 KB) — Tests graph building and query operations

## See Also

<!-- openwiki: broken internal link [omnigibson/scenes.md] file "omnigibson/scenes.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/scenes.md`](omnigibson/scenes.md) — Scene management
<!-- openwiki: broken internal link [omnigibson/scene_graphs.md] file "omnigibson/scene_graphs.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/scene_graphs.md`](omnigibson/scene_graphs.md) — Scene graphs overview
<!-- openwiki: broken internal link [../cross-system/omnigibson-bddl-integration.md] file "../cross-system/omnigibson-bddl-integration.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`cross-system/omnigibson-bddl-integration.md`](../cross-system/omnigibson-bddl-integration.md) — BDDL integration
