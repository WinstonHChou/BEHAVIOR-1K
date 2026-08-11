---
type: wiki-overview
title: OmniGibson Overview
description: Architecture overview of the OmniGibson simulation engine — registry pattern, config-driven design, lifecycle, lazy imports, and component hierarchy.
tags: [omnigibson, architecture, overview, registry-pattern]
---

# OmniGibson Overview

**OmniGibson** (v3.9.1) is a robotics simulation platform built on NVIDIA's [Omniverse/Isaac Sim](https://developer.nvidia.com/isaac-sim) framework, developed by Stanford's Visual Learning Lab. It accelerates Embodied AI research by providing photorealistic rendering, physical realism, dynamic object states, fluid/cloth simulation, and a clean OpenAI Gymnasium-compatible interface.

## Package Structure

```
OmniGibson/omnigibson/
├── __init__.py              # Main entry: og.launch(), og.sim, og.app, og.gm
├── simulator.py             # Simulator singleton (91 KB)
├── macros.py                # MacroDict config system (11 KB)
├── lazy.py                  # Lazy importer for Isaac Sim (120 B)
├── transition_rules.py      # Runtime transition engine (111 KB)
├── prims/                   # USD primitive layer (11 files)
├── objects/                 # Object abstraction (5 files)
├── robots/                  # Robot definitions (3 files + YAML)
├── controllers/             # Control hierarchy (10 files)
├── scenes/                  # Scene loading (5 files)
├── tasks/                   # RL tasks (6 files)
├── envs/                    # Gymnasium environments (9 files)
├── sensors/                 # Observation layer (6 files)
├── object_states/           # 40+ state classes (38 files)
├── systems/                 # Particle physics (4 files)
├── action_primitives/       # Motion planning/action skills (4 files)
├── termination_conditions/  # Episode end logic (9 files)
├── reward_functions/        # RL rewards (6 files)
├── metrics/                 # Evaluation metrics (3 files)
├── maps/                    # Spatial maps (3 files)
├── scene_graphs/            # Spatial relationships (2 files)
├── materials/               # USD materials (2 files)
├── configs/                 # YAML configuration files
├── eval/                    # Challenge evaluation framework
├── examples/                # Demo scripts
├── utils/                   # 33 utility modules
└── tests/                   # Test suite (separate from package)
```

## Registry Pattern

All extensible components use a **registry pattern** — a dictionary populated at import time via class decorators or automatic discovery. This enables plug-and-play extension.

| Registry | Location | Purpose |
|----------|----------|---------|
| `REGISTERED_OBJECTS` | `objects/` | Object classes (DatasetObject, PrimitiveObject, LightObject, etc.) |
| `REGISTERED_ROBOTS` | `robots/` | Auto-discovered from YAML files under DATA_PATH |
| `REGISTERED_CONTROLLERS` | `controllers/` | Controller classes (IK, OSC, Joint, Gripper, etc.) |
| `REGISTERED_LOCOMOTION_CONTROLLERS` | `controllers/` | Base movement controllers (DD, Holonomic) |
| `REGISTERED_MANIPULATION_CONTROLLERS` | `controllers/` | Arm/effector controllers (IK, OSC, Joint, Gripper) |
| `REGISTERED_SCENES` | `scenes/` | Scene classes |
| `REGISTERED_TASKS` | `tasks/` | RL task classes |
| `REGISTERED_TERMINATION_CONDITIONS` | `termination_conditions/` | Episode termination logic |
| `REGISTERED_SUCCESS_CONDITIONS` | `termination_conditions/` | Success detection |
| `REGISTERED_FAILURE_CONDITIONS` | `termination_conditions/` | Failure detection |
| `REGISTERED_REWARD_FUNCTIONS` | `reward_functions/` | RL reward functions |
| `REGISTERED_ENV_WRAPPERS` | `envs/` | Environment wrappers |
| `REGISTERED_RULES` | `transition_rules.py` | Transition rule recipes |

<!-- openwiki: broken internal link [omnigibson/registries.md] file "omnigibson/registries.md" does not exist. Fix the href or restore the target, then delete this comment. -->
See [`omnigibson/registries.md`](omnigibson/registries.md) for the full registry inventory and how to extend registries.

## Config-Driven Design

Every component (robot, scene, sensor, controller, task) is fully configurable via YAML files. Multiple configs can be stacked and merged procedurally via `merge_nested_dicts()` in `utils/config_utils.py`.

### Main config: `default_cfg.yaml`
```yaml
env:
  device: null
  action_frequency: 30
  rendering_frequency: 30
  physics_frequency: 120
  automatic_reset: true
scene:
  type: Scene
  model: null
robots: []
objects: []
task:
  type: DummyTask
```

### Robot config example (fetch.yaml)
```yaml
robot:
  name: Fetch
  model: fetch
  action_type: continuous
  controller_config:
    base:
      name: DifferentialDriveController
    arm_0:
      name: InverseKinematicsController
  sensor_config:
    VisionSensor:
      sensor_kwargs: {image_height: 128, image_width: 128}
```

<!-- openwiki: broken internal link [omnigibson/configs.md] file "omnigibson/configs.md" does not exist. Fix the href or restore the target, then delete this comment. -->
See [`omnigibson/configs.md`](omnigibson/configs.md) for full config structure.

## Lifecycle

```python
import omnigibson as og

# 1. Launch the simulator (singleton, once)
og.launch(
    gravity=9.81,
    physics_dt=1/120,    # 120 Hz physics
    rendering_dt=1/30,   # 30 Hz rendering
    sim_step_dt=1/30,    # 30 Hz simulation steps
    device="cuda:0",
)

# 2. Create an environment from config dict
env = og.Environment(configs={
    "scene": {"type": "Scene", "model": "Rs_int"},
    "robots": [{"name": "Locobot", "model": "locobot"}],
    "objects": [],
    "task": {"type": "PointNavigationTask"},
})

# 3. Gymnasium interface
obs, info = env.reset()
obs, reward, terminated, truncated, info = env.step(action)

# 4. Access global objects
og.sim      # Current Simulator singleton
og.app      # Isaac Sim SimulationApp
og.gm       # Global macros/configs

# 5. Close
env.close()
og.clear()  # Clean shutdown
```

Lifecycle phases:
1. `_launch_app()` → Launches Isaac Sim (once, singleton)
2. `_launch_simulator()` → Creates Simulator instance → `og.sim`
3. `Environment.__init__` → `_load_variables()` → scene + robots + task creation
4. `load()` → Scene.load() + task.load() + sim.play()
5. `env.reset()` → `_reset_scene()` + `_reset_agent()` + `get_obs()`
6. `env.step()` → `sim.step()` + `task.step()` + `get_obs()`
7. `env.close()` → `og.sim.stop()` + cleanup

<!-- openwiki: broken internal link [omnigibson/simulator.md] file "omnigibson/simulator.md" does not exist. Fix the href or restore the target, then delete this comment. -->
See [`omnigibson/simulator.md`](omnigibson/simulator.md) for the full step lifecycle.

## Lazy Imports

The `omnigibson.lazy` module provides a lazy importer for Isaac Sim dependencies to avoid circular dependencies and import errors. All imports from `isaacsim` must go through this mechanism since they are only available after the simulation app has launched (via `simulator.py`'s `launch_app`).

```python
from omnigibson.lazy import lazy_isaacsim
# Access Isaac Sim modules only after og.sim has been initialized
```

Follow the lazy imports to the source code by finding the appropriate extension inside the `isaacsim` directory in the conda env's site packages. Especially relevant extensions start with `isaacsim.core`.

## Global Configuration System

OmniGibson uses a **MacroDict** (`gm.globals`) for runtime configuration. Key flags include:

| Macro | Description |
|-------|-------------|
| `ENABLE_OBJECT_STATES` | Enable dynamic object states |
| `ENABLE_TRANSITION_RULES` | Enable rule-based transitions |
| `USE_GPU_DYNAMICS` | Enable GPU particle dynamics |
| `HEADLESS` | Headless mode |
| `USE_NUMPY_CONTROLLER_BACKEND` | Numpy vs Torch backend |
| `PHYSICS_FREQUENCY` | Physics step frequency |
| `RENDERING_FREQUENCY` | Rendering frequency |

<!-- openwiki: broken internal link [omnigibson/macros.md] file "omnigibson/macros.md" does not exist. Fix the href or restore the target, then delete this comment. -->
See [`omnigibson/macros.md`](omnigibson/macros.md) for the full configuration catalog.
