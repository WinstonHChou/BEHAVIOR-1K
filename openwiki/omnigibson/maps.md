---
type: wiki-component
title: OmniGibson Maps
description: SegmentationMap and TraversableMap — spatial maps for semantic segmentation and navigation traversability.
tags: [omnigibson, maps, segmentation, traversability, navigation]
---

# OmniGibson Maps

**Maps** provide spatial representations of the scene for navigation and perception. OmniGibson supports traversability maps (for robot navigation) and segmentation maps (for semantic perception).

## Classes

```python
from omnigibson.maps import (
    BaseMap,
    SegmentationMap,
    TraversableMap,
)
```

## BaseMap

**File:** `maps/base_map.py` (~2 KB)

**Purpose:** Abstract base class for all spatial maps.

```python
class BaseMap(ABC):
    """Abstract base class for spatial maps."""
    
    def __init__(self, scene, resolution=0.05):
        self.scene = scene
        self.resolution = resolution  # Map resolution in meters
        self.width = 0
        self.height = 0
        self.data = None  # Underlying data array
        self.origin = np.array([0.0, 0.0])  # Map origin in world coordinates
    
    @abstractmethod
    def update(self):
        """Update map from scene state."""
        pass
    
    def get_cell(self, x, y):
        """Get cell value at world coordinates."""
        ix, iy = self.world_to_index(x, y)
        if 0 <= ix < self.width and 0 <= iy < self.height:
            return self.data[iy, ix]
        return None
    
    def world_to_index(self, x, y):
        """Convert world coordinates to map indices."""
        ix = int((x - self.origin[0]) / self.resolution)
        iy = int((y - self.origin[1]) / self.resolution)
        return ix, iy
    
    def index_to_world(self, ix, iy):
        """Convert map indices to world coordinates."""
        x = self.origin[0] + ix * self.resolution
        y = self.origin[1] + iy * self.resolution
        return x, y
```

## SegmentationMap

**File:** `maps/segmentation_map.py` (~8 KB)

**Purpose:** Semantic segmentation map of the scene. Each cell contains the semantic category ID of the object at that position.

### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `data` | np.ndarray | Semantic segmentation data (H, W) |
| `semantic_labels` | dict | ID → category name mapping |
| `instance_labels` | np.ndarray | Instance segmentation data (H, W) |

### Creation

```python
seg_map = SegmentationMap(scene, resolution=0.05)
seg_map.update()  # Populate from scene
```

### Usage

```python
# Get semantic category at position
category_id = seg_map.get_cell(1.0, 1.0)  # Returns semantic ID
category_name = seg_map.semantic_labels.get(category_id, "unknown")

# Get all occupied cells
occupied = np.argwhere(seg_map.data > 0)  # (N, 2) indices
```

## TraversableMap

**File:** `maps/traversable_map.py` (~9 KB)

**Purpose:** Navigation traversability map. Each cell indicates whether a robot can navigate there.

### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `data` | np.ndarray | Traversability grid (H, W), 0=traversable, 1=occupied |
| `occupancy_threshold` | float | Distance threshold for occupancy |
| `robot_radius` | float | Robot base radius (for inflation) |
| `inflation_radius` | float | Inflation radius for obstacle avoidance |

### Creation

```python
trav_map = TraversableMap(scene, resolution=0.05, robot_radius=0.15, inflation_radius=0.2)
trav_map.update()
```

### Usage

```python
# Check if position is traversable
is_traversable = trav_map.get_cell(1.0, 1.0) == 0

# Get occupied cells
occupied = np.argwhere(trav_map.data > 0)

# Visualize map
import matplotlib.pyplot as plt
plt.imshow(trav_map.data, cmap='gray')
plt.colorbar()
plt.show()
```

### Map Inflation

TraversableMaps apply obstacle inflation based on robot radius:

```python
def _inflate_occupancy(self):
    """Inflate occupied cells based on robot radius."""
    from scipy.ndimage import distance_transform_edt
    
    # Compute distance transform
    dist = distance_transform_edt(self.data == 0) * self.resolution
    
    # Cells within inflation radius are marked as occupied
    self.data = (dist < self.robot_radius + self.inflation_radius).astype(int)
```

## Integration with Scenes

Maps are maintained by the scene:

```python
class Scene:
    def __init__(self):
        self.traversable_map = None
        self.segmentation_map = None
    
    def load(self):
        self.traversable_map = TraversableMap(
            scene=self,
            resolution=0.05,
            robot_radius=0.15,
        )
        self.segmentation_map = SegmentationMap(
            scene=self,
            resolution=0.05,
        )
    
    def step(self):
        # Update maps each step
        self.traversable_map.update()
        self.segmentation_map.update()
```

## Testing

- `test_examples.py` — Tests map usage in navigation examples
- `benchmark/benchmark_interactive_scene.py` — Map update timing

## See Also

<!-- openwiki: broken internal link [omnigibson/scenes.md] file "omnigibson/scenes.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/scenes.md`](omnigibson/scenes.md) — Scene management
<!-- openwiki: broken internal link [omnigibson/maps.md] file "omnigibson/maps.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/maps.md`](omnigibson/maps.md) — Maps overview
<!-- openwiki: broken internal link [omnigibson/scene_graphs.md] file "omnigibson/scene_graphs.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/scene_graphs.md`](omnigibson/scene_graphs.md) — Scene graphs (alternative spatial representation)
