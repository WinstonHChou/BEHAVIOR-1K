---
type: wiki-component
title: OmniGibson Scenes
description: Scene, TraversableScene, InteractiveTraversableScene — scene loading, asset handling, object placement, and reset behavior.
tags: [omnibibson, scenes, loading, assets, traversability]
---

# OmniGibson Scenes

**Scenes** define the 3D environment in which robots operate. They manage USD scene loading, object placement, traversability maps, and reset behavior.

## Registry

```python
from omnigibson.scenes import (
    Scene,
    TraversableScene,
    StaticTraversableScene,
    InteractiveTraversableScene,
    REGISTERED_SCENES,
)
```

## Class Hierarchy

```
Scene (omnigibson/scenes/scene_base.py, 51 KB)
│
├── TraversableScene (3 KB)
│   └── StaticTraversableScene (6 KB) — Non-interactive traversable
│   └── InteractiveTraversableScene (11 KB) — Interactive + traversable
```

## Scene Base

**File:** `scenes/scene_base.py` (~51 KB)

**Purpose:** Base class for all scenes. Handles USD loading, object placement, and scene reset.

### Key Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `model` | str | Scene model name (e.g., "Rs_int") |
| `objects` | list[USDObject] | All objects in scene |
| `traversable_map` | TraversableMap | Navigation traversability map |
| `segmentation_map` | SegmentationMap | Semantic segmentation map |

### Key Methods

| Method | Return Type | Description |
|--------|-------------|-------------|
| `load()` | None | Load USD scene, place objects |
| `reset()` | None | Reset scene state |
| `import_scene()` | None | Import USD scene from path |
| `get_traversable_map()` | TraversableMap | Get traversability map |
| `get_segmentation_map()` | SegmentationMap | Get segmentation map |
| `get_occupied_cells()` | np.ndarray | Get occupied cell positions |

## TraversableScene

**File:** `scenes/traversable_scene.py` (~3 KB)

**Purpose:** Scene with traversability information for robot navigation.

```python
class TraversableScene(Scene):
    """Scene where robots can navigate."""
    
    def get_traversable_map(self):
        """Return traversability map for robot navigation."""
        return self.traversable_map
```

## StaticTraversableScene

**File:** `scenes/static_traversable_scene.py` (~6 KB)

**Purpose:** Traversable scene with static (non-interactive) objects. More efficient than InteractiveTraversableScene.

```python
class StaticTraversableScene(TraversableScene):
    """Traversable scene with static (non-interactive) objects."""
    
    @property
    def is_interactive(self):
        return False
```

## InteractiveTraversableScene

**File:** `scenes/interactive_traversable_scene.py` (~11 KB)

**Purpose:** Traversable scene with interactive objects (objects that can be grasped, opened, etc.). Used for BEHAVIOR tasks.

```python
class InteractiveTraversableScene(TraversableScene):
    """Traversable scene with interactive objects."""
    
    @property
    def is_interactive(self):
        return True
    
    def get_interactive_objects(self):
        """Return list of interactive objects in scene."""
        return [obj for obj in self.objects if obj.interactive]
```

## Scene Loading

Scenes are loaded from USD files under `$OMNIGIBSON_DATA_PATH`:

```python
scene = Scene(model="Rs_int")
scene.load()  # Loads USD scene, creates objects
```

### Scene Asset Path

```
$OMNIGIBSON_DATA_PATH/*/models/scenes/<model>/scene.usd
```

The scene model name (e.g., "Rs_int") maps to a scene directory under the models/scenes path.

### Object Placement

Objects defined in the scene USD are automatically placed according to the USD file. Additional objects can be added programmatically:

```python
with og.sim.adding_objects():
    obj = DatasetObject(name="bowl.n.01_1", category="bowl.n.01")
    obj.set_position([0.5, 0.5, 0.0])
    og.sim.add_object(obj)
```

## Scene Reset

When an episode ends (or manually via `scene.reset()`):

1. **Object state reset** — All object states are cleared and re-initialized
2. **Object position reset** — Objects return to initial positions
3. **Robot reset** — Robots return to initial joint positions
4. **Task reset** — Task goals are re-sampled

## Traversable Map

The traversability map is a 2D grid where each cell indicates whether a robot can navigate there:

```python
traversable_map = scene.get_traversable_map()
# traversable_map.data: np.ndarray of shape (height, width)
# Values: 0 = traversable, 1 = occupied
```

The map is computed from the scene's collision geometry and updated when objects move.

## Segmentation Map

The segmentation map provides semantic and instance segmentation for each pixel:

```python
seg_map = scene.get_segmentation_map()
# seg_map.seg_semantic: np.ndarray (H, W) — semantic labels
# seg_map.seg_instance: np.ndarray (H, W) — instance IDs
```

## Scene Graph

Scenes maintain a spatial relationship graph via the `GraphBuilder`:

```python
from omnigibson.scene_graphs.graph_builder import GraphBuilder

graph = GraphBuilder(scene)
graph.build()  # Build spatial relationship graph
```

## Scene Types

| Scene Type | Interactive | Traversable | Use Case |
|-----------|-------------|-------------|----------|
| `Scene` | No | No | Simple scenes |
| `StaticTraversableScene` | No | Yes | Navigation tasks |
| `InteractiveTraversableScene` | Yes | Yes | BEHAVIOR tasks (default) |

## Testing

- `test_scene_graph.py` (~4 KB) — Tests spatial relationship graph building
- `test_examples.py` — Tests scene loading and navigation
- `benchmark/benchmark_interactive_scene.py` — Scene load/step performance

## See Also

<!-- openwiki: broken internal link [omnigibson/robots.md] file "omnigibson/robots.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/robots.md`](omnigibson/robots.md) — Robot navigation in scenes
<!-- openwiki: broken internal link [omnigibson/maps.md] file "omnigibson/maps.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/maps.md`](omnigibson/maps.md) — Traversable and segmentation maps
<!-- openwiki: broken internal link [omnigibson/scene_graphs.md] file "omnigibson/scene_graphs.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/scene_graphs.md`](omnigibson/scene_graphs.md) — Spatial relationship graphs
<!-- openwiki: broken internal link [omnigibson/tasks.md] file "omnigibson/tasks.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/tasks.md`](omnigibson/tasks.md) — Task placement in scenes
