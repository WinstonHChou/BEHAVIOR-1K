---
type: wiki-component
title: OmniGibson Robot Class
description: The monolithic Robot class — arm+base+gripper control, proprioception, grasping modes, YAML-config-driven robot definitions, and controller composition.
tags: [omnigibson, robot, control, proprioception, grasping]
---

# OmniGibson Robot Class

**`Robot`** (defined in `robots/robot.py`, ~200 KB) is the monolithic class representing a physical robot in simulation. It composes a mobile base, one or more arms, a gripper, and associated controllers into a unified interface. Robots are registered via YAML configuration files under `$OMNIGIBSON_DATA_PATH/*/models/*/`.

## Entry Point

```python
from omnigibson.robots.robot import Robot
# Robots are auto-discovered from YAML files when the package is imported
```

## Class Hierarchy

```
omnigibson.objects.usd_object.USDObject (ABC)
│
└── Robot
    │
    └── (auto-discovered from YAML configs under DATA_PATH)
        ├── Fetch
        ├── Locobot
        ├── TurtleBot3
        ├── Husky
        ├── Freight
        ├── R1Pro
        ├── Tiago
        ├── Franka
        └── ...
```

## Architecture

A robot instance composes three main subsystems:

```
Robot
├── Base (Locomotion)
│   └── LocomotionController (DifferentialDrive, Holonomic, NullJoint)
├── Trunk/Torso
│   └── JointController
├── Arms (one or more)
│   ├── ArmJointController
│   └── IKController / OSCController
└── Grippers
    └── MultiFingerGripperController
```

## Key Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `name` | str | Robot name |
| `model` | str | Model name (from YAML) |
| `action_type` | str | "continuous" or "discrete" |
| `action_normalize` | bool | Normalize actions to [-1, 1] |
| `grasping_mode` | str | "physical" or "assisted" |
| `controller_config` | dict | Per-DOF controller config |
| `sensor_config` | dict | Sensor placement config |
| `proprio_obs` | list[str] | Proprioceptive observation keys |
| `dof_info` | dict | Mapping of DOF to controller type |
| `base_dofs` | list[str] | Base joint DOF names |
| `trunk_dofs` | list[str] | Trunk/torso joint DOF names |
| `arm_dofs` | list[str] | Arm joint DOF names |
| `gripper_dofs` | list[str] | Gripper joint DOF names |

## Controller Composition

The robot builds its controller hierarchy from the `controller_config` dict:

```python
# From robot.yaml:
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
```

This produces:

| DOF | Controller | Control Mode |
|-----|------------|--------------|
| base | DifferentialDriveController | Linear velocity + angular velocity |
| trunk | JointController | Joint position |
| arm_0 | InverseKinematicsController | End-effector pose (position + quaternion) |
| gripper_0 | MultiFingerGripperController | Gripper position/width |

### `ControllerView`

Each controller is wrapped in a `ControllerView` that exposes a unified interface:

```python
# ControllerView provides:
view.action_space     # Action space for this controller
view.observation_space  # Observation space for this controller
view.step(action)      # Apply action to controller
view.get_obs()         # Get controller state observations
```

## Proprioceptive Observations

The robot exposes a structured observation dictionary:

```python
obs = robot.get_observations()
# Keys include:
# - eef_0_pos, eef_0_quat  # End-effector position and orientation
# - trunk_qpos, arm_0_qpos, arm_0_qpos_sin, arm_0_qpos_cos
# - base_qpos, base_vel
# - gripper_0_qpos, gripper_0_force
# - grasp_main, grasp_center, grasp_success  # Grasp state
# - eef_0_pos, eef_0_quat  # Per-EFF position/orientation
```

The exact keys depend on `proprio_obs` config:

```yaml
robot:
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
```

## Grasping Modes

### Physical Grasping

```yaml
robot:
  grasping_mode: physical
```

Uses PhysX contact detection to determine if the robot is grasping an object. The robot's end-effector must physically contact the object, and collision filters must allow the contact.

### Assisted Grasping

```yaml
robot:
  grasping_mode: assisted
```

Uses a simplified "assisted" grasping mode where the robot can pick up objects without precise collision detection. This is faster but less physically accurate.

## Action Interface

### Continuous Actions

```python
# Action space: Box(low, high, shape=...)
action = env.action_space.sample()
obs, reward, terminated, truncated, info = env.step(action)
```

The action is split by controller:
- Base: `[linear_vel, angular_vel]`
- Trunk: `[joint_position]`
- Arm: `[eef_x, eef_y, eef_z, quat_x, quat_y, quat_z, quat_w]`
- Gripper: `[gripper_position]`

### Discrete Actions

```yaml
robot:
  action_type: discrete
```

Discrete actions use a lookup table defined in the robot's controller config.

## Key Methods

| Method | Return Type | Description |
|--------|-------------|-------------|
| `get_observations()` | dict | Full observation dict (proprio + sensor) |
| `get_proprio_obs()` | dict | Proprioceptive observations only |
| `step(action)` | None | Apply action to controllers |
| `reset()` | None | Reset robot to initial state |
| `compute_jacobians()` | dict | Compute per-arm Jacobians |
| `get_grasp_status()` | dict | Get grasp state (grasped, grasp_center, etc.) |
| `set_position(position)` | None | Set base position directly |
| `set_orientation(orientation)` | None | Set base orientation directly |
| `load(controller_cfg, sensor_cfg)` | None | Load controller/sensor config |
| `compute_joint_limits()` | None | Compute joint limits from URDF |

## YAML Configuration

Robots are defined via YAML files under `$OMNIGIBSON_DATA_PATH/*/models/*/`. The robot schema is defined in `robots/definition_schema.py`.

### Example: fetch.yaml

```yaml
robot:
  name: Fetch
  model: fetch
  action_type: continuous
  action_normalize: true
  proprio_obs:
    - eef_0_pos
    - eef_0_quat
    - trunk_qpos
    - arm_0_qpos_sin
    - arm_0_qpos_cos
    - gripper_0_qpos
    - grasp_main
    - base_qpos
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

## Robot State Caching

The robot maintains cached state for fast lookup (flat cache):

```python
# Cached state (updated in _on_post_physics_step)
self.pos_w                          # World-space position (N, 3)
self.orn_w                          # World-space orientation (N, 4)
self.vel_lin_w                      # Linear velocity (N, 3)
self.vel_ang_w                      # Angular velocity (N, 3)
self.fext_w                         # External forces (N, 3)
self.torques_w                      # External torques (N, 3)
self.qpos                           # Joint positions (N_qpos,)
self.qvel                           # Joint velocities (N_qvel,)
```

## Testing

- `test_controllers.py` — Tests all controller types with robots
- `test_robot_states_flatcache.py` — Tests robot state flat cache accuracy
- `test_robot_teleoperation.py` — Tests robot teleoperation via VR/keyboard

## See Also

<!-- openwiki: broken internal link [omnigibson/robot_schema.md] file "omnigibson/robot_schema.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/robot_schema.md`](omnigibson/robot_schema.md) — Robot YAML definition schema
<!-- openwiki: broken internal link [omnigibson/controllers.md] file "omnigibson/controllers.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/controllers.md`](omnigibson/controllers.md) — Controller hierarchy
<!-- openwiki: broken internal link [omnigibson/objects.md] file "omnigibson/objects.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/objects.md`](omnigibson/objects.md) — USDObject base class
<!-- openwiki: broken internal link [omnigibson/configs.md] file "omnigibson/configs.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/configs.md`](omnigibson/configs.md) — Config structure
