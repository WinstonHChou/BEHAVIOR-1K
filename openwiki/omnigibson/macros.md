---
type: wiki-component
title: OmniGibson Macros & Global Configuration
description: MacroDict system — immutable, lazy-evaluated configuration with module-level scoping, read-locking, and all runtime configuration knobs.
tags: [omnigibson, configuration, macros, global-config]
---

# OmniGibson Macros & Global Configuration

**`MacroDict`** (in `macros.py`, ~11 KB) is OmniGibson's immutable, lazy-evaluated configuration system. All runtime configuration flows through the global macro singleton `gm`, which is imported via `from omnigibson.macros import gm`.

## Core Design

`MacroDict` implements a **read-tracking immutability** pattern:

1. **Lazy evaluation** — Values are computed on first access via callable factories
2. **Read tracking** — Each value tracks which modules have read it via the `_read` set
3. **Read-locking** — Once all expected modules have read a value, it becomes immutable (locked)
4. **Module scoping** — Sub-macros are auto-created per module via `create_module_macros(module_path)`

```python
from omnigibson.macros import gm

# Access global macros
print(gm.PHYSICS_FREQUENCY)  # 120

# Modify only within unlock context
with gm.unlock():
    gm.PHYSICS_FREQUENCY = 60
```

## Entry Point

The global `gm` singleton is created in `omnigibson/__init__.py` and initialized with default values:

```python
from omnigibson.macros import gm
```

## Configuration Categories

### Physics & Simulation

| Macro | Default | Description |
|-------|---------|-------------|
| `PHYSICS_FREQUENCY` | 120 | Physics simulation Hz |
| `RENDERING_FREQUENCY` | 30 | Rendering Hz |
| `ACTION_FREQUENCY` | 30 | Action execution Hz |
| `SIM_STEP_DT` | 1/30 | Simulation timestep |
| `PHYSICS_DT` | 1/120 | Physics timestep |
| `GRAVITY` | 9.81 | Gravity acceleration |
| `ENABLE_DYNAMICS` | True | Enable physics simulation |
| `ENABLE_INTERACTION` | True | Enable object interaction |
| `ENABLE_GRIPPING` | True | Enable robot grasping |

### Object States

| Macro | Default | Description |
|-------|---------|-------------|
| `ENABLE_OBJECT_STATES` | True | Enable dynamic object states |
| `ENABLE_SYSTEMS` | True | Enable particle systems |
| `ENABLE_TRANSITION_RULES` | True | Enable rule-based transitions |

### GPU & Performance

| Macro | Default | Description |
|-------|---------|-------------|
| `USE_GPU_DYNAMICS` | False | Enable GPU particle dynamics |
| `USE_GPU_PHYSX` | True | Enable GPU PhysX |
| `USE_GPU_RENDERING` | True | Enable GPU rendering |
| `CUDA_VISIBLE_DEVICES` | (empty) | GPU selection |
| `HEADLESS` | False | Headless mode |
| `USE_NUMPY_CONTROLLER_BACKEND` | False | Use numpy instead of torch for controllers |

### Rendering

| Macro | Default | Description |
|-------|---------|-------------|
| `VIEWER_WIDTH` | 1280 | Viewer window width |
| `VIEWER_HEIGHT` | 720 | Viewer window height |
| `ENABLE_POST_PROCESSING` | True | Enable post-processing effects |
| `ENABLE_SHADOWS` | True | Enable shadows |
| `LIGHTING_MODE` | "default" | Lighting mode |

### Object State & Transition Rules

| Macro | Default | Description |
|-------|---------|-------------|
| `ENABLE_OBJECT_STATES` | True | Enable dynamic object states |
| `ENABLE_TRANSITION_RULES` | True | Enable rule-based transitions |
| `DISABLED_TRANSITION_RULES` | [] | Transition rules to skip (used by JoyLo, eval) |
| `ENABLE_FILLED` | True | Enable "Filled" state |
| `ENABLE_COOKED` | True | Enable "Cooked" state |
| `ENABLE_CONTAINS` | True | Enable "Contains" state |

### Paths & Data

| Macro | Default | Description |
|-------|---------|-------------|
| `OMNIGIBSON_DATA_PATH` | (set by env var) | Main data directory |
| `OMNIGIBSON_APPDATA_PATH` | (set by env var) | App data directory |
| `DATA_PATH` | (derived) | Resolved data path |
| `DEFAULT_SCENE_PATH` | (path) | Default scene loading path |

### Profiling & Debug

| Macro | Default | Description |
|-------|---------|-------------|
| `ENABLE_PROFILING` | False | Enable performance profiling |
| `LOG_LEVEL` | "INFO" | Logging level |
| `ENABLE_DEBUG` | False | Enable debug mode |
| `PRINT_TIMINGS` | False | Print step timings |

## Module-Level Macros

Each submodule can create its own scoped macros:

```python
from omnigibson.macros import create_module_macros

# Creates gm.OMNIGIBSON.SIMULATOR, gm.OMNIGIBSON.OBJECTS, etc.
my_macros = create_module_macros(__file__)
```

Module-level macros are independently locked — reading a module-level macro doesn't lock global macros.

## Configuration Loading

Configuration flows through multiple layers:

1. **Default macros** — Built into `macros.py` as defaults
2. **Environment config** — YAML config from `og.Environment(configs=...)`
3. **Robot config** — Per-robot YAML files under DATA_PATH
4. **Controller config** — Per-controller YAML files under configs/

Config merging uses `merge_nested_dicts()` from `utils/config_utils.py`:

```python
from omnigibson.utils.config_utils import merge_nested_dicts

final_config = merge_nested_dicts(default_config, user_config)
```

## Runtime Modification

Macros can be modified before they are first read, or within an unlock context:

```python
from omnigibson.macros import gm

# Modify before first read (all modules share the same lock)
with gm.unlock():
    gm.ENABLE_OBJECT_STATES = False

# Or modify a specific macro independently
gm.set("ENABLE_OBJECT_STATES", False)
```

Once a macro is read by all expected modules, it becomes immutable. Attempts to modify locked macros will raise errors.

## Cross-Dependencies

Some macros have implicit dependencies:

- `ENABLE_OBJECT_STATES` must be True for `ENABLE_TRANSITION_RULES` to have effect
- `USE_GPU_DYNAMICS` requires `USE_GPU_PHYSX` to be True
- `ENABLE_FILLED`, `ENABLE_COOKED`, `ENABLE_CONTAINS` are all sub-flags of `ENABLE_OBJECT_STATES`
- `HEADLESS` affects rendering frequency and viewer settings

## Testing

<!-- openwiki: broken internal link [omnigibson/tests.md] file "omnigibson/tests.md" does not exist. Fix the href or restore the target, then delete this comment. -->
See [`omnigibson/tests.md`](omnigibson/tests.md) for the test strategy. Macros are primarily tested indirectly through environment tests and the full integration tests.
