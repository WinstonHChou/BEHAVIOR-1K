---
type: wiki-component
title: OmniGibson Registries
description: Complete inventory of all REGISTERED_* dicts — what can be registered, how registration works, config-to-registry mapping, and full entry list.
tags: [omnigibson, registry-pattern, registration, extensibility]
---

# OmniGibson Registries

OmniGibson uses a **registry pattern** for all extensible components. Each registry is a `dict[str, Class]` populated either via explicit decorator registration or automatic discovery from YAML/config files.

## Registry Implementation

The core registry mechanism is defined in `utils/registry_utils.py`:

```python
class Registry:
    """Registry for extensible components."""
    def __init__(self, name):
        self.name = name
        self._registry = {}
    
    def register(self, key):
        """Decorator to register a class."""
        def wrapper(cls):
            self._registry[key] = cls
            return cls
        return wrapper
    
    def __getitem__(self, key):
        return self._registry[key]
    
    def __contains__(self, key):
        return key in self._registry
    
    @property
    def keys(self):
        return list(self._registry.keys())
```

Most components use a simpler pattern: a module-level `REGISTERED_*` dict that is populated via `@Registerable.register()` decorators or direct assignment.

## Complete Registry Inventory

### Objects (`objects/`)

| Registry | Location | Purpose |
|----------|----------|---------|
| `REGISTERED_OBJECTS` | `objects/__init__.py` | All object classes |

**Registered objects:**
- `USDObject` — Base class (ABCMeta, Registerable)
- `DatasetObject` — Pre-sampled BEHAVIOR objects with taxonomy abilities
- `PrimitiveObject` — Simple primitives (boxes, spheres, cylinders)
- `LightObject` — Light emitters (dome, point, spot)

**Registration:** Classes are registered via the `Registerable` metaclass:

```python
@Registerable.register()
class MyObject(USDObject):
    pass
```

### Robots (`robots/`)

| Registry | Location | Purpose |
|----------|----------|---------|
| `REGISTERED_ROBOTS` | `robots/__init__.py` | Auto-discovered from YAML |

**Discovery:** Robot classes are auto-discovered from YAML files under `$OMNIGIBSON_DATA_PATH/*/models/*/`:

```python
# robots/__init__.py scans DATA_PATH for robot YAML files
# and creates a registry entry for each model
```

**Known robots:** fetch, locobot, turtlebot3, husky, freight, r1pro, tiago, panda, franka

### Controllers (`controllers/`)

| Registry | Location | Purpose |
|----------|----------|---------|
| `REGISTERED_CONTROLLERS` | `controllers/__init__.py` | All controller classes |
| `REGISTERED_LOCOMOTION_CONTROLLERS` | `controllers/__init__.py` | Base movement controllers |
| `REGISTERED_MANIPULATION_CONTROLLERS` | `controllers/__init__.py` | Arm/effector controllers |

**Locomotion controllers:**
- `DifferentialDriveController` — Differential drive base (DD)
- `HolonomicBaseJointController` — Holonomic (omnidirectional) base
- `NullJointController` — Passive/read-only base

**Manipulation controllers:**
- `InverseKinematicsController` — Task-space pose control (IK)
- `OperationalSpaceController` — Operational space control (OSC)
- `JointController` — Direct joint position/velocity/effort control
- `MultiFingerGripperController` — Gripper control
- `DDController` — Differential drive (legacy)

**Registration:** Controllers can be registered via decorator or discovered from YAML config files:

```python
# Controllers can be configured via YAML files in configs/controllers/
# e.g., controllers/ik.yaml, controllers/joint.yaml, controllers/dd.yaml
```

### Scenes (`scenes/`)

| Registry | Location | Purpose |
|----------|----------|---------|
| `REGISTERED_SCENES` | `scenes/__init__.py` | All scene classes |

**Registered scenes:**
- `Scene` — Base scene class
- `TraversableScene` — Scene where robots can navigate
- `StaticTraversableScene` — Traversable but static (non-interactive)
- `InteractiveTraversableScene` — Traversable + interactive objects

**Discovery:** Scene classes are auto-discovered from YAML files under `$OMNIGIBSON_DATA_PATH/*/models/scenes/`.

### Tasks (`tasks/`)

| Registry | Location | Purpose |
|----------|----------|---------|
| `REGISTERED_TASKS` | `tasks/__init__.py` | All task classes |

**Registered tasks:**
- `BaseTask` — Abstract task base
- `BehaviorTask` — BEHAVIOR-style task from BDDL (primary task type)
- `GraspTask` — Grasping task
- `PointNavigationTask` — Navigation to point goal
- `PointReachingTask` — Reaching to point goal
- `DummyTask` — Minimal no-op task

### Termination Conditions (`termination_conditions/`)

| Registry | Location | Purpose |
|----------|----------|---------|
| `REGISTERED_TERMINATION_CONDITIONS` | `termination_conditions/__init__.py` | Termination logic |
| `REGISTERED_SUCCESS_CONDITIONS` | `termination_conditions/__init__.py` | Success detection |
| `REGISTERED_FAILURE_CONDITIONS` | `termination_conditions/__init__.py` | Failure detection |

**Termination conditions:**
- `Timeout` — Episode step limit
- `Falling` — Object fell below threshold
- `MaxCollision` — Too many collisions
- `PointGoal` — Reached point goal
- `ReachingGoal` — Reached reaching goal
- `GraspGoal` — Grasping goal achieved
- `PredicateGoal` — Predicate-based termination

### Reward Functions (`reward_functions/`)

| Registry | Location | Purpose |
|----------|----------|---------|
| `REGISTERED_REWARD_FUNCTIONS` | `reward_functions/__init__.py` | All reward types |

**Registered rewards:**
- `BaseRewardFunction` — Abstract base (Registerable metaclass)
- `PointGoalReward` — Point-goal proximity reward
- `ReachingGoalReward` — Reaching-goal reward
- `GraspReward` — Grasping success reward
- `CollisionReward` — Collision penalty reward
- `PotentialReward` — Potential-based shaping reward

### Environment Wrappers (`envs/`)

| Registry | Location | Purpose |
|----------|----------|---------|
| `REGISTERED_ENV_WRAPPERS` | `envs/env_wrapper.py` | All wrapper types |

**Registered wrappers:**
- `EnvironmentWrapper` — Base wrapper class
- `DataWrapper` — Data logging/recording
- `HDF5DataWrapper` — HDF5 dataset collection
- `LeRobotDataWrapper` — LeRobot dataset format
- `MetricsWrapper` — Task metric collection

### Transition Rules (`transition_rules.py`)

| Registry | Location | Purpose |
|----------|----------|---------|
| `REGISTERED_RULES` | `transition_rules.py` | All transition recipe rules |

**Registered rules:** Cooking, Washing, Mixing, Machine, SubstanceCooking, Washer, Dicing, Melting, Slicing rules.

## How Registration Works

### Decorator Pattern (Static Registration)

```python
from omnigibson.utils.python_utils import Registerable

@Registerable.register()
class MyCustomRobot(Robot):
    """My custom robot implementation."""
    pass
# MyCustomRobot is now in REGISTERED_ROBOTS
```

### Config-Based Discovery (Dynamic Registration)

```python
# Controllers are registered from YAML files
# configs/controllers/my_controller.yaml → REGISTERED_CONTROLLERS["my_controller"]
```

### How Config Maps to Registry

When an environment is created from config:

```python
env = og.Environment(configs={
    "robots": [{"controller_config": {"name": "InverseKinematicsController"}}]
})
```

The config key (`"InverseKinematicsController"`) is looked up in the appropriate registry (`REGISTERED_CONTROLLERS`) to instantiate the class.

## Extending Registries

To register a new component type:

1. Create your class inheriting from the base class
2. Use the `@Registerable.register()` decorator
3. Ensure your config key matches the registry lookup

For config-based registration, create a YAML file in the appropriate config directory:

```yaml
# configs/robots/my_robot.yaml
robot:
  name: MyRobot
  model: my_robot
  controller_config:
    arm_0:
      name: InverseKinematicsController
```

## Testing

Registry integrity is tested via `test_examples.py` which validates that all registered components can be instantiated and used in a simulated environment.
