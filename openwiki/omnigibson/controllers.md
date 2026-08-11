---
type: wiki-component
title: OmniGibson Controllers
description: Controller hierarchy — IK, OSC, Joint, Gripper, Differential Drive, Holonomic controllers, ControllerView, and registered controller registry.
tags: [omnigibson, controllers, control, ik, osc, trajectory]
---

# OmniGibson Controllers

**Controllers** provide the interface between high-level actions (e.g., "move arm to position X") and low-level joint commands (e.g., "set joint 0 to angle 0.5 rad). They are organized in a hierarchy with a base registry, and each controller type can be configured via YAML.

## Registry

```python
from omnigibson.controllers import (
    REGISTERED_CONTROLLERS,
    REGISTERED_LOCOMOTION_CONTROLLERS,
    REGISTERED_MANIPULATION_CONTROLLERS,
)
```

## Controller Hierarchy

```
BaseController (omnigibson/controllers/base_controller.py, 35 KB)
│
├── ControllerView (omnigibson/controllers/controller_view.py, 13 KB)
│   └── Wrapper that exposes controller state to robots
│
├── Locomotion Controllers:
│   ├── DifferentialDriveController (dd_controller.py)
│   ├── HolonomicBaseJointController (holonomic_base_joint_controller.py)
│   └── NullJointController (null_gripper.py)
│
├── Manipulation Controllers:
│   ├── InverseKinematicsController (ik_controller.py, 25 KB)
│   ├── OperationalSpaceController (osc.py, 31 KB)
│   ├── JointController (joint_controller.py, 22 KB)
│   └── MultiFingerGripperController (multi_finger_gripper.py, 21 KB)
│
└── Legacy:
    └── DDController (dd_controller.py, 7 KB)
```

## Base Controller

All controllers inherit from `BaseController`:

```python
class BaseController(ABC):
    """Abstract base class for all controllers."""
    
    # Abstract methods
    @abstractmethod
    def step(self, action):
        """Apply action to controller."""
        pass
    
    @abstractmethod
    def get_obs(self):
        """Get controller state observations."""
        pass
    
    @abstractmethod
    @property
    def action_space(self):
        """Return action space."""
        pass
    
    @abstractmethod
    @property
    def observation_space(self):
        """Return observation space."""
        pass
```

The base controller also handles:
- **Config parsing** — Extracts controller parameters from YAML
- **Action normalization** — Maps actions to controller-specific ranges
- **Observation space construction** — Builds gymnasium spaces from config

## Controller Types

### 1. InverseKinematicsController (IK)

**File:** `controllers/ik_controller.py` (~25 KB)

**Purpose:** Task-space pose control. Computes joint angles to place an end-effector at a target position and orientation.

**Action space:**
```
[dx, dy, dz, dquat_x, dquat_y, dquat_z, dquat_w]
```
Each action is a *delta* from the current pose (not an absolute target).

**Key parameters:**
- `damping_alpha` — Damping factor for Jacobian pseudo-inverse (default: 0.5)
- `position_gain` — Position error gain (default: 1.0)
- `orientation_gain` — Orientation error gain (default: 1.0)
- `max_velocity` — Maximum joint velocity (default: 1.0)
- `use_current_as_initial_guess` — Use current pose as IK initial guess

**Implementation:** Uses damped least-squares inverse kinematics:
```
q_new = q_current + J^T * (J * J^T + αI)^(-1) * (target_pose - current_pose)
```

### 2. OperationalSpaceController (OSC)

**File:** `controllers/osc.py` (~31 KB)

**Purpose:** Operational space control for high-precision task-space manipulation. More sophisticated than IK with force/torque control.

**Action space:**
```
[fx, fy, fz, tx, ty, tz]  # Forces and torques in operational space
```

**Key features:**
- Handles kinematic constraints
- Supports both position and force control modes
- Computes operational space inertia matrix
- Supports null-space projection for secondary tasks

### 3. JointController

**File:** `controllers/joint_controller.py` (~22 KB)

**Purpose:** Direct joint position/velocity/effort control.

**Action space:**
```
[joint_positions]  # or [joint_velocities] or [joint_torques]
```

**Key parameters:**
- `control_type` — "position", "velocity", or "effort"
- `gains` — P, I, D gains per joint
- `action_normalize` — Whether to normalize actions

### 4. MultiFingerGripperController

**File:** `controllers/multi_finger_gripper.py` (~21 KB)

**Purpose:** Gripper control.

**Action space:**
```
[gripper_position]  # Normalized to [0, 1]
```

**Key parameters:**
- `gripper_action_range` — [open_position, close_position]
- `gripper_velocity_scale` — Maximum gripper velocity
- `force_threshold` — Grasp detection threshold

### 5. DifferentialDriveController

**File:** `controllers/dd_controller.py` (~22 KB)

**Purpose:** Differential drive base control.

**Action space:**
```
[linear_velocity, angular_velocity]
```

**Key parameters:**
- `wheel_radius` — Wheel radius
- `wheel_separation` — Distance between wheels
- `action_normalize` — Whether to normalize actions

### 6. HolonomicBaseJointController

**File:** `controllers/holonomic_base_joint_controller.py` (~10 KB)

**Purpose:** Holonomic (omnidirectional) base control.

**Action space:**
```
[linear_x, linear_y, angular_z]
```

## ControllerView

**File:** `controllers/controller_view.py` (~13 KB)

`ControllerView` wraps a `BaseController` instance to expose a unified interface:

```python
class ControllerView:
    """Wrapper that exposes controller state to robots."""
    
    def __init__(self, controller):
        self.controller = controller
    
    @property
    def action_space(self):
        return self.controller.action_space
    
    @property
    def observation_space(self):
        return self.controller.observation_space
    
    def step(self, action):
        self.controller.step(action)
    
    def get_obs(self):
        return self.controller.get_obs()
```

Each robot's DOFs are mapped to controller views:

```python
robot.controller_views["arm_0"]     # ControllerView for arm_0 IK
robot.controller_views["base"]      # ControllerView for base DD
robot.controller_views["gripper_0"] # ControllerView for gripper
```

## YAML Configuration

Controllers are configured via YAML files under `configs/controllers/`:

### IK Controller Config (ik.yaml)

```yaml
name: InverseKinematicsController
damping_alpha: 0.5
position_gain: 1.0
orientation_gain: 1.0
max_velocity: 1.0
use_current_as_initial_guess: true
```

### Joint Controller Config (joint.yaml)

```yaml
name: JointController
control_type: position
gains:
  P: 100.0
  I: 0.0
  D: 1.0
action_normalize: true
```

### Gripper Config (multi_finger_gripper.yaml)

```yaml
name: MultiFingerGripperController
gripper_action_range: [0.0, 0.08]  # [open, close] meters
gripper_velocity_scale: 0.1
force_threshold: 5.0
```

## Controller Selection in Robot Config

The robot's `controller_config` maps DOF keys to controller names:

```yaml
robot:
  controller_config:
    base:
      name: DifferentialDriveController
    trunk:
      name: JointController
    arm_0:
      name: InverseKinematicsController
      subsume_controllers: [trunk]  # IK replaces trunk control
    gripper_0:
      name: MultiFingerGripperController
```

## Action Flow

1. **Action received** — `Environment.step(action)` receives action array
2. **Split by controller** — Action is split according to robot's action space structure
3. **Controller.step(action_chunk)** — Each controller processes its action chunk
4. **Joint commands** — Controllers compute target joint positions/velocities/efforts
5. **Apply to simulation** — Commands are applied in `simulator.py`'s `_on_pre_physics_step()`

## Testing

- `test_controllers.py` (~24 KB) — Comprehensive tests for all controller types:
  - `test_ik_controller` — Tests IK solution accuracy
  - `test_osc_controller` — Tests OSC force control
  - `test_joint_controller` — Tests joint position/velocity/effort control
  - `test_gripper_controller` — Tests gripper opening/closing
  - `test_dd_controller` — Tests differential drive movement
  - `test_holonomic_controller` — Tests holonomic base movement
  - Controller observation space tests
  - Controller action normalization tests

## See Also

<!-- openwiki: broken internal link [omnigibson/robots.md] file "omnigibson/robots.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/robots.md`](omnigibson/robots.md) — Robot controller composition
<!-- openwiki: broken internal link [omnigibson/configs.md] file "omnigibson/configs.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/configs.md`](omnigibson/configs.md) — Config structure
<!-- openwiki: broken internal link [omnigibson/registries.md] file "omnigibson/registries.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/registries.md`](omnigibson/registries.md) — Controller registry
