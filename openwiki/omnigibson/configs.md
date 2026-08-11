---
type: wiki-component
title: OmniGibson Configuration
description: Config file structure — default_cfg.yaml, sensor/robot/controller YAML configs, config merging (merge_nested_dicts), config precedence.
tags: [omnigibson, configuration, yaml, config-merging]
---

# OmniGibson Configuration

OmniGibson uses YAML configuration extensively. Every component (scene, robot, sensor, controller, task) can be fully configured via YAML files.

## Config Structure

### Config Directory

```
OmniGibson/omnigibson/configs/
├── default_cfg.yaml          # Default environment config
├── sensors/
│   ├── vision.yaml           # VisionSensor defaults
│   └── scan.yaml             # ScanSensor defaults
├── robots/
│   ├── fetch.yaml            # Fetch robot definition
│   ├── locobot.yaml          # Locobot robot definition
│   ├── turtlebot3.yaml       # TurtleBot3 definition
│   ├── husky.yaml            # Husky definition
│   ├── freight.yaml          # Freight definition
│   ├── r1pro.yaml            # R1Pro definition
│   ├── tiago.yaml            # Tiago definition
│   └── franka.yaml           # Franka definition
├── controllers/
│   ├── dd.yaml               # DifferentialDriveController
│   ├── ik.yaml               # InverseKinematicsController
│   ├── osc.yaml              # OperationalSpaceController
│   ├── joint.yaml            # JointController
│   ├── multi_finger_gripper.yaml  # Multi-finger Gripper
│   └── null_gripper.yaml     # Null/Passive Gripper
├── r1pro_behavior.yaml       # R1Pro behavior task config
├── r1pro_primitives.yaml     # R1Pro action primitives
├── r1_primitives.yaml        # R1 action primitives
├── tiago_primitives.yaml     # Tiago action primitives
├── turtlebot_nav.yaml        # TurtleBot navigation config
└── franka_vector_env.yaml    # Franka + vector env config
```

## Default Config (default_cfg.yaml)

```yaml
# Environment settings
env:
  device: null                  # GPU device (null = auto)
  action_frequency: 30          # Action frequency (Hz)
  rendering_frequency: 30       # Rendering frequency (Hz)
  physics_frequency: 120        # Physics frequency (Hz)
  automatic_reset: true         # Auto-reset on termination
  flatten_obs_space: true       # Flatten observation space
  flatten_action_space: true    # Flatten action space

# Viewer settings
render:
  viewer_width: 1280
  viewer_height: 720

# Scene settings
scene:
  type: Scene                   # Scene type
  model: null                   # Scene model name

# Robot settings (list of robots)
robots:
  - name: Locobot
    model: locobot
    action_type: continuous
    action_normalize: true
    controller_config:
      base:
        name: DifferentialDriveController
      trunk:
        name: JointController
      arm_0:
        name: InverseKinematicsController
        subsume_controllers: [trunk]
      gripper_0:
        name: MultiFingerGripperController
    sensor_config:
      VisionSensor:
        sensor_kwargs:
          image_height: 128
          image_width: 128
      ScanSensor:
        sensor_kwargs:
          min_range: 0.05
          max_range: 10.0

# Object settings (list of objects)
objects: []

# Task settings
task:
  type: DummyTask
  termination_config:
    type: Timeout
    max_steps: 1000
  reward_config:
    type: PointGoalReward
```

## Sensor Configs

### VisionSensor (sensors/vision.yaml)

```yaml
name: VisionSensor
sensor_kwargs:
  image_height: 128
  image_width: 128
  min_distance: 0.01
  max_distance: 10.0
  clip_near: 0.01
  clip_far: 10.0
  semantic_segmentation: true
  instance_segmentation: true
  depth: true
  rgb: true
  normal: false
  fov: 90.0
  position: [0.0, 0.0, 0.5]
  orientation: [0.0, 0.0, 0.0]
```

### ScanSensor (sensors/scan.yaml)

```yaml
name: ScanSensor
sensor_kwargs:
  min_range: 0.05
  max_range: 10.0
  resolution: 360
  horizontal_fov: 360.0
  vertical_fov: 10.0
  position: [0.0, 0.0, 0.1]
  orientation: [0.0, 0.0, 0.0]
```

## Robot Configs

### Locobot (robots/locobot.yaml)

```yaml
robot:
  name: Locobot
  model: locobot
  action_type: continuous
  action_normalize: true
  proprio_obs:
    - eef_0_pos
    - eef_0_quat
    - trunk_qpos
    - arm_0_qpos
    - arm_0_qpos_sin
    - arm_0_qpos_cos
    - gripper_0_qpos
    - grasp_main
    - base_qpos
    - base_vel
  grasping_mode: physical
  controller_config:
    base:
      name: DifferentialDriveController
    trunk:
      name: JointController
    arm_0:
      name: InverseKinematicsController
      subsume_controllers: [trunk]
    gripper_0:
      name: MultiFingerGripperController
  sensor_config:
    VisionSensor:
      sensor_kwargs:
        image_height: 128
        image_width: 128
        min_distance: 0.01
        max_distance: 10.0
    ScanSensor:
      sensor_kwargs:
        min_range: 0.05
        max_range: 10.0
```

## Controller Configs

### Inverse Kinematics (controllers/ik.yaml)

```yaml
name: InverseKinematicsController
damping_alpha: 0.5
position_gain: 1.0
orientation_gain: 1.0
max_velocity: 1.0
use_current_as_initial_guess: true
```

### Joint Controller (controllers/joint.yaml)

```yaml
name: JointController
control_type: position
gains:
  P: 100.0
  I: 0.0
  D: 1.0
action_normalize: true
```

### Multi-finger Gripper (controllers/multi_finger_gripper.yaml)

```yaml
name: MultiFingerGripperController
gripper_action_range: [0.0, 0.08]
gripper_velocity_scale: 0.1
force_threshold: 5.0
```

### Differential Drive (controllers/dd.yaml)

```yaml
name: DifferentialDriveController
wheel_radius: 0.0325
wheel_separation: 0.28
action_normalize: true
```

## Config Precedence

Config values are merged in order (later values override earlier):

1. **Default config** (`default_cfg.yaml`) — Baseline values
2. **Robot YAML** (e.g., `locobot.yaml`) — Robot-specific defaults
3. **User config** (passed to `og.Environment`) — User overrides
4. **Environment config** (env.* in user config) — Env-level overrides

```python
def _merge_config(self, config):
    # Start with defaults
    merged = load_yaml("configs/default_cfg.yaml")
    
    # Apply robot-specific configs
    for robot_cfg in merged.get("robots", []):
        robot_yaml = f"configs/robots/{robot_cfg.get('model')}.yaml"
        if os.path.exists(robot_yaml):
            robot_defaults = load_yaml(robot_yaml)
            merged = merge_nested_dicts(merged, robot_defaults)
    
    # Apply user config (overrides all)
    merged = merge_nested_dicts(merged, config)
    
    return merged
```

## merge_nested_dicts Utility

```python
from omnigibson.utils import config_utils

def merge_nested_dicts(base, override):
    """Recursively merge override dict into base dict."""
    result = base.copy()
    for key, value in override.items():
        if key in result and isinstance(result[key], dict) and isinstance(value, dict):
            result[key] = merge_nested_dicts(result[key], value)
        else:
            result[key] = value
    return result
```

## External YAML Files

Configs can reference external YAML files:

```python
# In user config, reference external file
env = og.Environment(configs={
    "task": "configs/r1pro_behavior.yaml",  # Will be loaded and merged
})
```

## Testing

- `test_examples.py` — Tests environment creation from configs
- `test_envs.py` — Tests config parsing and environment creation

## See Also

<!-- openwiki: broken internal link [omnigibson/configs.md] file "omnigibson/configs.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/configs.md`](omnigibson/configs.md) — Config overview
<!-- openwiki: broken internal link [omnigibson/environments.md] file "omnigibson/environments.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/environments.md`](omnigibson/environments.md) — Environment config usage
<!-- openwiki: broken internal link [omnigibson/robots.md] file "omnigibson/robots.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/robots.md`](omnigibson/robots.md) — Robot YAML format
