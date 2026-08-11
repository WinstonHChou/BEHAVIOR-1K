---
type: wiki-component
title: OmniGibson Utilities
description: Key utility modules — transform_utils, config_utils, bddl_utils (bridge to BDDL3), asset_utils, registry_utils, python_utils, and more.
tags: [omnigibson, utilities, transforms, bddl-bridge, assets, registry]
---

# OmniGibson Utilities

OmniGibson provides a comprehensive set of utility modules under `omnigibson/utils/` (~33 files). These handle transforms, config parsing, BDDL integration, asset management, registry patterns, and more.

## Utility Modules

| Module | Description | Key Symbols |
|--------|-------------|-------------|
| `transform_utils.py` | 3D transform math (4×4, quaternions, rotation) | `pos_quat_4x4_to_pose_mat`, `pose_mat_to_pos_quat`, `quat_multiply`, `quat_conjugate` |
| `transform_utils_np.py` | NumPy-based transform math | `pos_quat_4x4_to_pose_mat_np`, `transform_batch_np` |
| `config_utils.py` | YAML/JSON config parsing | `merge_nested_dicts`, `load_yaml`, `load_json` |
| `bddl_utils.py` | BDDL bridge (74 KB) | `get_behavior_activities`, `BDDLSampler`, `get_knowledge_base`, `og_categories_from_bddl_inst` |
| `asset_utils.py` | Asset download, versioning, hashing | `download_asset`, `hash_asset`, `get_asset_version` |
| `asset_conversion_utils.py` | URDF→USD conversion (113 KB) | `convert_urdf_to_usd`, `process_urdf_mesh` |
| `urdfpy_utils.py` | URDF parsing via urdfpy (133 KB) | `parse_urdf`, `extract_links`, `extract_joints` |
| `usd_utils.py` | USD manipulation primitives (130 KB) | `create_prim`, `set_attr`, `get_prim_at_path` |
| `registry_utils.py` | Registry pattern (21 KB) | `Registry`, `Registerable` metaclass |
| `python_utils.py` | Registerable metaclass, Serializable base | `Registerable`, `Serializable` |
| `object_utils.py` | Object creation from config | `create_object_from_config`, `get_object_class` |
| `object_state_utils.py` | Object state query utilities (19 KB) | `check_state_satisfied`, `find_state_candidates` |
| `control_utils.py` | Control theory helpers | `compute_jacobian`, `pseudoinverse` |
| `geometry_utils.py` | Geometric predicates | `point_in_polyhedron`, `sphere_bbox_overlap` |
| `grasping_planning_utils.py` | Grasping planning (26 KB) | `compute_grasp_pose`, `find_valid_grasps` |
| `motion_planning_utils.py` | Motion planning (25 KB) | `plan_motion`, `check_collision_free` |
| `teleop_utils.py` | Teleoperation utilities (33 KB) | `parse_teleop_config`, `map_teleop_action` |
| `vision_utils.py` | Vision processing | `process_camera_obs`, `extract_depth_from_rgb` |
| `render_utils.py` | Rendering helpers | `render_image`, `render_depth` |
| `processing_utils.py` | Data processing (14 KB) | `process_batch`, `normalize_obs` |
| `data_utils.py` | Data pipeline utilities (9 KB) | `save_dataset`, `load_dataset` |
| `numpy_utils.py` | NumPy helpers | `batch_transform`, `repeat_along_axis` |
| `sampling_utils.py` | Random sampling (60 KB) | `sample_pose`, `sample_object_positions` |
| `ui_utils.py` | UI, logging, disclaimer (50 KB) | `display_disclaimer`, `print_info`, `print_warning` |
| `backend_utils.py` | Backend selection | `set_backend`, `get_backend` |
| `lazy_import_utils.py` | Lazy imports (1.7 KB) | `lazy_import` |
| `deprecated_utils.py` | Deprecated API warnings (61 KB) | `deprecated`, `warn_deprecated` |
| `git_utils.py` | Git operations | `get_git_hash`, `get_git_status` |
| `profiling_utils.py` | Performance profiling | `ProfileTimer`, `benchmark` |
| `coacd_runner.py` | COACD convex decomposition | `run_coacd` |
| `pynvml_utils.py` | NVIDIA GPU monitoring (148 KB) | `GPUStats`, `get_gpu_memory` |
| `physx_utils.py` | PhysX API wrappers | `create_physx_scene`, `apply_force` |
| `gym_utils.py` | Gym observation space utilities (6 KB) | `flatten_obs_space`, `unflatten_obs_space` |
| `constants.py` | Enumerations | `LightingMode`, `ParticleModifyCondition` |
| `lazy.py` | Lazy module importer (120 bytes) | `LazyImporter` replacement |

## Key Utility Categories

### Transform Utilities

**File:** `transform_utils.py` (~52 KB)

Core 3D math for positions, orientations, and transformations:

```python
from omnigibson.utils.transform_utils import (
    pos_quat_4x4_to_pose_mat,
    pose_mat_to_pos_quat,
    quat_multiply,
    quat_conjugate,
    rotation_matrix_to_6d,
    pose_transform,
)

# Convert position + quaternion to 4×4 matrix
pose = pos_quat_4x4_to_pose_mat(pos, quat)

# Convert 4×4 matrix back to position + quaternion
pos, quat = pose_mat_to_pos_quat(pose)

# Compose transformations
combined = pose_transform(pose1, pose2)
```

### BDDL Bridge

**File:** `bddl_utils.py` (~74 KB)

**Critical bridge between BDDL3 and OmniGibson.** This is the primary integration point between the BDDL domain language and OmniGibson's simulation runtime.

```python
from omnigibson.utils.bddl_utils import (
    get_behavior_activities,
    BDDLSampler,
    get_knowledge_base,
    og_categories_from_bddl_inst,
    translate_bddl_recipe_to_og_recipe,
)

# Get all BDDL activity names
activities = get_behavior_activities()

# Create a sampler for behavior tasks
sampler = BDDLSampler(kb=knowledge_base)
task = sampler.sample_task()

# Get knowledge base from BDDL3
kb = get_knowledge_base()

# Map BDDL instances to OmniGibson categories
og_cats = og_categories_from_bddl_inst(bddl_instances)

# Translate BDDL recipe to OmniGibson recipe
og_recipe = translate_bddl_recipe_to_og_recipe(bddl_recipe)
```

### Asset Utilities

**File:** `asset_utils.py`

Handles downloading, versioning, and caching of 3D assets:

```python
from omnigibson.utils.asset_utils import (
    download_asset,
    hash_asset,
    get_asset_version,
    ASSET_CACHE_PATH,
)

# Download asset to cache
asset_path = download_asset(asset_url, asset_id)

# Hash asset for integrity check
asset_hash = hash_asset(asset_path)
```

### Registry Utilities

**File:** `registry_utils.py` (~21 KB)

Implements the `Registry` class and `Registerable` metaclass:

```python
from omnigibson.utils.registry_utils import Registry, Registerable

# Create a registry
my_registry = Registry()

# Register a class
@my_registry.register()
class MyClass:
    pass

# Get registered class
cls = my_registry["my_class"]

# List all registered names
names = list(my_registry.keys())
```

### Serializable

**File:** `python_utils.py`

Base class for serializable objects (config to/from dict):

```python
from omnigibson.utils.python_utils import Serializable

class MyConfig(Serializable):
    def __init__(self, value=0):
        self.value = value
    
    @classmethod
    def from_dict(cls, d):
        return cls(value=d["value"])
    
    def to_dict(self):
        return {"value": self.value}
```

## Testing

- `test_transform_utils.py` (~20 KB) — Tests transform math operations
- `test_examples.py` — Tests utility usage in examples

## See Also

<!-- openwiki: broken internal link [omnigibson/utils.md] file "omnigibson/utils.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/utils.md`](omnigibson/utils.md) — Overview
<!-- openwiki: broken internal link [../cross-system/omnigibson-bddl-integration.md] file "../cross-system/omnigibson-bddl-integration.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`cross-system/omnigibson-bddl-integration.md`](../cross-system/omnigibson-bddl-integration.md) — BDDL integration
<!-- openwiki: broken internal link [omnigibson/registries.md] file "omnigibson/registries.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/registries.md`](omnigibson/registries.md) — Registry pattern
