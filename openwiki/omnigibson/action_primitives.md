---
type: wiki-component
title: OmniGibson Action Primitives
description: ActionPrimitiveSetBase, Curobo motion planning integration, starter semantic primitives, symbolic primitives — high-level action skills for robots.
tags: [omnigibson, action-primitives, motion-planning, curobo, skill]
---

# OmniGibson Action Primitives

**Action primitives** provide high-level action abstractions for robots. They sit above low-level controllers and provide semantic actions like "pick", "place", "navigate", "slide_to_object".

## Registry

```python
from omnigibson.action_primitives import (
    ActionPrimitiveSetBase,
    starter_semantic_action_primitives,
    symbolic_semantic_action_primitives,
    curobo,
)
```

## Class Hierarchy

```
ActionPrimitiveSetBase (omnigibson/action_primitives/action_primitive_set_base.py, 3 KB, ABC)
│
├── Curobo (51 KB) — NVIDIA Curobo motion planner integration
│
├── starter_semantic_action_primitives (95 KB) — Starter-level semantic actions
│   ├── Pick
│   ├── Place
│   ├── Navigate
│   ├── Open
│   ├── Close
│   ├── Pour
│   └── ... (many more)
│
└── symbolic_semantic_action_primitives (27 KB) — Symbolic planning actions
    ├── SlicingRule
    └── ... (symbolic action wrappers)
```

## ActionPrimitiveSetBase

**File:** `action_primitives/action_primitive_set_base.py` (~3 KB)

**Purpose:** Abstract base class for action primitive sets. Defines the interface that all primitive sets must implement.

```python
class ActionPrimitiveSetBase(ABC):
    """Base class for action primitive sets."""
    
    def __init__(self, robot, scene):
        self.robot = robot
        self.scene = scene
    
    @abstractmethod
    def act(self, action):
        """Execute an action primitive."""
        pass
    
    @abstractmethod
    def get_action_space(self):
        """Return action space."""
        pass
    
    @abstractmethod
    def reset(self):
        """Reset primitive state."""
        pass
```

## Curobo

**File:** `action_primitives/curobo.py` (~51 KB)

**Purpose:** NVIDIA Curobo motion planner integration. Provides high-quality, collision-free motion planning for robot arms.

### Integration

```python
from omnigibson.action_primitives.curobo import CuroboActionPrimitive

curobo = CuroboActionPrimitive(
    robot=robot,
    scene=scene,
    config_path="configs/curobo_config.yaml",
)

# Plan and execute a motion
success = curobo.act(
    target_pos=[0.5, 0.5, 0.3],
    target_quat=[0, 0, 0, 1],
)
```

### Key Features

- **Collision avoidance** — Checks for collisions with scene objects
- **Kinematic planning** — Plans joint trajectories that respect joint limits
- **Multiple objectives** — Supports reaching, grasping, and navigation goals
- **Caching** — Caches previous solutions for faster planning

### Configuration

```yaml
# curobo_config.yaml
curobo:
  num_samples: 100
  horizon: 30
  dt: 0.033
  weight_position: 1.0
  weight_orientation: 0.5
  weight_velocity: 0.1
  weight_torque: 0.01
  collision_check_distance: 0.02
```

## Starter Semantic Action Primitives

**File:** `action_primitives/starter_semantic_action_primitives.py` (~95 KB)

**Purpose:** Starter-level semantic actions that provide robot-level commands. These are higher-level than Curobo but lower-level than symbolic planning.

### Available Primitives

| Primitive | Description | Action Space |
|-----------|-------------|--------------|
| `Pick` | Pick up an object | Target position + orientation |
| `Place` | Place an object at target | Target position + orientation |
| `Navigate` | Move base to target | (x, y, theta) |
| `Open` | Open an articulated object | Joint position target |
| `Close` | Close an articulated object | Joint position target |
| `Pour` | Pour substance from container | Target position + pour rate |
| `SlideToObject` | Slide tool across object | Contact position + force |
| `Push` | Push an object | Direction + force |
| `Pull` | Pull an object | Direction + force |
| `Wipe` | Wipe surface | Sweeping trajectory |
| `Sweep` | Sweep floor | Circular motion trajectory |
| `Mop` | Mop floor | Wet wiping trajectory |
| `Spray` | Spray substance | Target position + spray rate |
| `Stir` | Stir contents | Circular motion trajectory |
| `Cut` | Cut object with tool | Cutting trajectory |

### Example: Pick Primitive

```python
from omnigibson.action_primitives.starter_semantic_action_primitives import PickPrimitive

pick = PickPrimitive(robot=robot, scene=scene)

# Execute a pick action
success = pick.act(
    target_object="bowl.n.01_1",
    grasp_type="parallel",
)

# Check success
if success:
    print(f"Picked {pick.picked_object}")
```

### Pick Primitive Flow

```
1. Move to pre-grasp pose
2. Approach target object
3. Close gripper
4. Verify grasp (IsGrasping state)
5. Retreat to safe pose
```

## Symbolic Semantic Action Primitives

**File:** `action_primitives/symbolic_semantic_action_primitives.py` (~27 KB)

**Purpose:** Symbolic action primitives that wrap BDDL predicates into robot actions. These connect symbolic planning (BDDL) to physical actions.

### Key Primitives

| Primitive | BDDL Predicate | Description |
|-----------|---------------|-------------|
| `SlicingRule` | `slicable`, `sliced` | Slice an object with a knife |
| `CookingAction` | `cooked`, `heated` | Heat an object |
| `WashingAction` | `clean`, `stained` | Clean an object |
| `MixingAction` | `mixed`, `stirred` | Mix substances |

### Slicing Rule

```python
from omnigibson.action_primitives.symbolic_semantic_action_primitives import SlicingRule

slice_action = SlicingRule(
    robot=robot,
    scene=scene,
    knife_obj="knife.n.01_1",
    target_obj="apple.n.01_1",
)

# Execute slicing
success = slice_action.act()
# Sets: apple → sliced_apple, knife → on_top(apple)
```

## Action Primitive Selection

Action primitives are selected based on the task type:

```python
class BehaviorTask(BaseTask):
    def get_action_primitives(self):
        """Return appropriate action primitives for this task."""
        if self.task_type == "grasp":
            return [PickPrimitive, PlacePrimitive]
        elif self.task_type == "cooking":
            return [CookingAction, PickPrimitive, PlacePrimitive]
        elif self.task_type == "navigation":
            return [NavigatePrimitive]
        else:
            return [PickPrimitive, PlacePrimitive, NavigatePrimitive]
```

## Primitive State Management

Each primitive maintains state across multiple environment steps:

```python
class PickPrimitive:
    def __init__(self, robot, scene):
        self.state = "IDLE"  # IDLE → APPROACH → GRASP → RETREAT → DONE
        self.robot = robot
        self.scene = scene
        self.target_object = None
    
    def step(self):
        """Execute one step of the primitive."""
        if self.state == "APPROACH":
            self._move_to_approach_pose()
            if self._at_approach_pose():
                self.state = "GRASP"
        elif self.state == "GRASP":
            self._close_gripper()
            if self.robot.grasp_success:
                self.state = "RETREAT"
        elif self.state == "RETREAT":
            self._move_to_retreat_pose()
            if self._at_retreat_pose():
                self.state = "DONE"
    
    def reset(self):
        self.state = "IDLE"
        self.target_object = None
```

## Testing

- `test_primitives.py` (~6 KB) — Tests primitive action execution
- `test_symbolic_primitives.py` (~12 KB) — Tests symbolic action primitives
- `test_curobo.py` (~21 KB) — Tests Curobo motion planning integration (skipped if Curobo unavailable)

## See Also

<!-- openwiki: broken internal link [omnigibson/robots.md] file "omnigibson/robots.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/robots.md`](omnigibson/robots.md) — Robot controllers (called by primitives)
<!-- openwiki: broken internal link [omnigibson/action_primitives.md] file "omnigibson/action_primitives.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/action_primitives.md`](omnigibson/action_primitives.md) — Action primitives overview
<!-- openwiki: broken internal link [omnigibson/tasks.md] file "omnigibson/tasks.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/tasks.md`](omnigibson/tasks.md) — Task primitive integration
