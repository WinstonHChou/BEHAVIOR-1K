---
type: wiki-component
title: OmniGibson Examples
description: Example ecosystem — environments, learning, objects, robots, scenes, sensors, object_states, teleoperation, and WIP demos.
tags: [omnigibson, examples, demos, tutorials]
---

# OmniGibson Examples

Examples are the primary way users interact with OmniGibson. They demonstrate API usage, feature workflows, and provide templates for building custom applications.

## Example Directory Structure

```
OmniGibson/omnigibson/examples/
├── environments/
│   ├── behavior_env_demo.py         # BEHAVIOR environment demo
│   ├── navigation_env_demo.py       # Navigation environment demo
│   └── vector_env_demo.py           # Vectorized environment demo
├── learning/
│   └── navigation_policy_demo.py    # SB3 PPO policy demo
├── object_states/
│   ├── attachment_demo.py           # Attached state demo
│   ├── dicing_demo.py               # Dicing/slicing demo
│   ├── heated_state_demo.py         # Heated/cooked state demo
│   ├── object_state_texture_demo.py # State-based texture demo
│   ├── onfire_demo.py               # OnFire state demo
│   ├── overlaid_demo.py             # Overlaid state demo
│   ├── particle_applier_remover_demo.py  # Particle demo
│   ├── sample_kinematics_demo.py    # Kinematics demo
│   ├── slicing_demo.py              # Slicing demo
│   ├── temperature_demo.py          # Temperature demo
│   └── folded_unfolded_state_demo.py # Cloth demo (WIP)
├── objects/
│   ├── draw_bounding_box.py         # Object bounding boxes
│   ├── highlight_objects.py         # Object highlighting
│   ├── import_custom_object.py      # Custom object import
│   ├── load_object_selector.py      # Object selection GUI
│   └── visualize_object.py          # Object visualization
├── robots/
│   ├── all_robots_visualizer.py     # Robot visualization
│   ├── grasping_mode_example.py     # Grasping modes demo
│   ├── import_custom_robot.py       # Custom robot import
│   ├── robot_control_example.py     # Robot control demo
│   └── sample_kinematics_demo.py    # Robot kinematics demo
├── scenes/
│   ├── scene_selector.py            # Scene selection GUI
│   ├── scene_tour_demo.py           # Scene navigation demo
│   └── traversability_map_example.py # Traversable map demo
├── sensors/
│   ├── sensor_data_visualization.py # Sensor data viz (if exists)
├── simulator/
│   └── sim_save_load_example.py     # Save/load simulation state
├── teleoperation/
│   ├── robot_teleoperate_demo.py    # Robot teleoperation demo
│   └── vr_scene_tour_demo.py        # VR scene tour demo
└── wip/                             # Work-in-progress demos
    ├── curobo_example.py            # Curobo motion planner
    ├── heat_source_or_sink_demo.py  # Heat source/sink demo
    ├── particle_source_sink_demo.py # Particle source/sink demo
    ├── rs_int_primitives_example.py # R1Pro primitives
    ├── solve_behavior_task.py       # Solve BDDL task
    ├── solve_simple_task.py         # Solve simple task
    └── view_cloth_configurations.py # Cloth config viewer
```

## Running Examples

Examples are run from the `OmniGibson/` directory:

```bash
cd OmniGibson

# Run an example (requires OMNIGIBSON_HEADLESS=1 if no display)
OMNIGIBSON_HEADLESS=1 python omniGibson/examples/environments/behavior_env_demo.py

# With GPU and display
python omniGibson/examples/robots/all_robots_visualizer.py
```

## Example Categories

### Environment Demos

Show how to create and interact with OmniGibson environments:

```python
# behavior_env_demo.py
import omnigibson as og

og.launch()
env = og.Environment(configs="configs/default_cfg.yaml")

obs, info = env.reset()
for _ in range(1000):
    action = env.action_space.sample()
    obs, reward, terminated, truncated, info = env.step(action)
    if terminated or truncated:
        obs, info = env.reset()

env.close()
og.clear()
```

### Object State Demos

Demonstrate dynamic object states and transitions:

```python
# temperature_demo.py
# Demonstrates: Temperature, Heated, Cooked states
# Cooking transitions: raw → heated → cooked → burnt

obj = DatasetObject(name="apple.n.01_1", category="apple.n.01")
og.sim.add_object(obj)

# Add cooking states
obj.add_state(Temperature, 293.0)  # Room temperature
obj.add_state(Heated, 373.0)       # Heated
obj.add_state(Cooked, {})          # Cooked state

# Transition rules update states during simulation
og.sim.step()
```

### Robot Demos

Show robot control, visualization, and custom robot import:

```python
# robot_control_example.py
# Demonstrates: Robot control via IK, OSC, Joint controllers

robot = og.robot_factory("Locobot")
og.sim.add_object(robot)

# Control via IK controller
robot.controller_views["arm_0"].step([0.1, 0.0, 0.0, 0.0, 0.0, 0.0, 1.0])
```

### Teleoperation Demos

Show remote control of robots via keyboard/VR:

```python
# robot_teleoperate_demo.py
# Demonstrates: Keyboard and VR teleoperation

from omnigibson.utils.teleop_utils import setup_teleoperation

teleop = setup_teleoperation(robot, env)
while teleop.running:
    teleop.update()
    og.sim.step()
```

### Simulator Demos

Show save/load simulation state:

```python
# sim_save_load_example.py
# Demonstrates: Saving and loading simulation state

# Save state
og.sim.save_state("checkpoint.pkl")

# Load state
og.sim.load_state("checkpoint.pkl")
```

## Testing

Examples are tested via `test_examples.py`:

```python
# test_examples.py runs all examples with pytest
pytest tests/test_examples.py -k "behavior_env_demo"
```

## See Also

<!-- openwiki: broken internal link [omnigibson/examples.md] file "omnigibson/examples.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/examples.md`](omnigibson/examples.md) — Overview
<!-- openwiki: broken internal link [omnigibson/scripts.md] file "omnigibson/scripts.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/scripts.md`](omnigibson/scripts.md) — CLI scripts
<!-- openwiki: broken internal link [omnigibson/environments.md] file "omnigibson/environments.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/environments.md`](omnigibson/environments.md) — Environment API
