---
type: wiki-component
title: OmniGibson Simulator
description: The monolithic Simulator singleton — physics engine, scene loading, USD manipulation, step lifecycle, and Isaac Sim integration.
tags: [omnigibson, simulator, physics, lifecycle]
---

# OmniGibson Simulator

**`Simulator`** (defined in `simulator.py`, ~91 KB) is the monolithic singleton that wraps NVIDIA Isaac Sim's physics engine. All physics stepping, scene management, USD manipulation, and lifecycle coordination go through this class, accessible globally via `og.sim`.

## Entry Point

```python
import omnigibson as og
og.launch(...)  # Creates Simulator singleton → og.sim
og.sim          # Access the simulator instance
```

## Step Lifecycle

The simulator executes a tightly-coupled loop of three phases per step:

```
sim.step()
  │
  ├─► _on_pre_physics_step()
  │    ├─► Update all object states (position-dependent)
  │    ├─► Execute transition rules (TransitionRuleAPI)
  │    └─► Apply controller commands (position/velocity/effort)
  │
  ├─► sim_context.step()       # PhysX simulation step
  │
  └─► _on_post_physics_step()
       ├─► Read sensor data (VisionSensor, ScanSensor)
       └─► Update flat caches for robot states
```

### Pre-Physics Phase
1. **Object state updates** — Updates position-dependent object states (e.g., `OnTop`, `Inside`) by iterating over all scene objects
2. **Transition rules** — `TransitionRuleAPI` scans all registered rules, finds matching object candidates, and returns `TransitionResults` for object creation/removal
3. **Controller commands** — Controller outputs (position, velocity, effort targets) are applied to robot joints

### Physics Step
Calls Isaac Sim's native `sim_context.step()` which advances the PhysX simulation by one physics timestep.

### Post-Physics Phase
1. **Sensor data reading** — Vision sensors read RGB/depth/segmentation frames; Scan sensors read LiDAR point clouds
2. **Flat cache updates** — Robot joint positions, velocities, end-effector transforms are cached for fast lookup

## Key Methods

| Method | Purpose |
|--------|---------|
| `__init__()` | Creates Isaac Sim `SimulationContext`, sets up callbacks for pre/post step |
| `play()` | Start the simulation loop |
| `stop()` | Stop the simulation loop |
| `step()` | Execute one full step (pre → physics → post) |
| `import_scene(scene)` | Load a USD scene into the simulator |
| `add_object(obj)` | Add an object via `adding_objects()` context manager |
| `batch_remove_objects()` | Remove multiple objects efficiently |
| `_on_pre_physics_step()` | Callback: update object states, run transition rules, apply controllers |
| `_on_post_physics_step()` | Callback: read sensor data, update robot state caches |
| `edit_usd()` | Context manager for safe USD editing |

## Scene Management

The simulator owns the scene graph and manages scene loading:

```python
og.sim.import_scene(scene)  # Add scene to simulator
og.sim.scenes               # List of loaded scenes
```

Scenes are loaded through `scene_base.py` which handles USD parsing, object placement, and robot initialization.

## Object Lifecycle

Objects are managed through the `adding_objects()` context manager which ensures proper ordering of USD primitive creation:

```python
with og.sim.adding_objects():
    og.sim.add_object(my_obj)
```

Removal uses `batch_remove_objects()` for efficiency.

## Isaac Sim Integration

The simulator wraps Isaac Sim's `SimulationContext`:

```python
# Access Isaac Sim context through the simulator
og.sim.sim_context      # Isaac Sim SimulationContext
og.sim.renderer         # Rendering backend
```

All Isaac Sim imports go through `omnigibson.lazy` to avoid circular dependencies.

## Physics Parameters

Configurable via macros or environment config:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `physics_frequency` | 120 Hz | Physics simulation frequency |
| `rendering_frequency` | 30 Hz | Rendering frequency |
| `action_frequency` | 30 Hz | Action execution frequency |
| `gravity` | 9.81 | Gravity acceleration |
| `sim_step_dt` | 1/30 | Simulation timestep |

## State Save/Load

The simulator supports full state persistence:

```python
og.sim.save_state()  # Save full simulation state
og.sim.load_state()  # Restore saved state
```

This is used by the data collection wrappers for checkpoint/rollback functionality.
