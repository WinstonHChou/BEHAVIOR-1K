---
type: wiki-component
title: OmniGibson Termination Conditions
description: Termination conditions — Falling, GraspGoal, MaxCollision, PointGoal, PredicateGoal, ReachingGoal, Timeout, with registry and success/failure categorization.
tags: [omnigibson, termination, success, failure, episode-control]
---

# OmniGibson Termination Conditions

**Termination conditions** determine when an episode should end. They evaluate after each step and can signal success, failure, or general termination. All conditions are registered in three separate registries.

## Registry

```python
from omnigibson.termination_conditions import (
    BaseTerminationCondition,
    REGISTERED_TERMINATION_CONDITIONS,
    REGISTERED_SUCCESS_CONDITIONS,
    REGISTERED_FAILURE_CONDITIONS,
)
```

## Class Hierarchy

```
BaseTerminationCondition (omnigibson/termination_conditions/base.py, 5 KB, ABC)
│
├── Timeout (1 KB)
├── Falling (2 KB)
├── MaxCollision (2 KB)
├── PointGoal (2 KB)
├── ReachingGoal (1 KB)
├── GraspGoal (600 B)
└── PredicateGoal (1 KB)
```

## BaseTerminationCondition

**Purpose:** Abstract base class for all termination conditions.

```python
class BaseTerminationCondition(ABC):
    """Base class for termination conditions."""
    
    def __init__(self, config):
        self.config = config
        self._state = {}  # Internal state
    
    @abstractmethod
    def evaluate(self, env, scene, robots, task):
        """Return True if condition is met."""
        pass
    
    @property
    @abstractmethod
    def is_success(self):
        """Whether this condition indicates success."""
        pass
    
    @property
    @abstractmethod
    def is_failure(self):
        """Whether this condition indicates failure."""
        pass
```

## Registered Conditions

### General Termination Conditions

```python
REGISTERED_TERMINATION_CONDITIONS = {
    "Timeout": Timeout,
    "Falling": Falling,
    "MaxCollision": MaxCollision,
    "PointGoal": PointGoal,
    "ReachingGoal": ReachingGoal,
    "GraspGoal": GraspGoal,
    "PredicateGoal": PredicateGoal,
}
```

### Success Conditions

```python
REGISTERED_SUCCESS_CONDITIONS = {
    "PointGoal": PointGoal,
    "ReachingGoal": ReachingGoal,
    "GraspGoal": GraspGoal,
    "PredicateGoal": PredicateGoal,
}
```

### Failure Conditions

```python
REGISTERED_FAILURE_CONDITIONS = {
    "Falling": Falling,
    "MaxCollision": MaxCollision,
}
```

## Condition Types

### Timeout

**File:** `termination_conditions/timeout.py` (~1 KB)

**Purpose:** Terminates after a maximum number of steps. Always a general termination (not success or failure).

```python
class Timeout(BaseTerminationCondition):
    def __init__(self, config):
        self.max_steps = config.get("max_steps", 1000)
        self.step_count = 0
    
    def evaluate(self, env, scene, robots, task):
        self.step_count += 1
        return self.step_count >= self.max_steps
    
    @property
    def is_success(self):
        return False
    
    @property
    def is_failure(self):
        return False
```

### Falling

**File:** `termination_conditions/falling.py` (~2 KB)

**Purpose:** Terminates if a specified object falls below a height threshold. Always a failure.

```python
class Falling(BaseTerminationCondition):
    def __init__(self, config):
        self.object_name = config["object_name"]
        self.threshold = config.get("threshold", -0.5)
    
    def evaluate(self, env, scene, robots, task):
        obj = scene.get_object(self.object_name)
        if obj is None:
            return False
        return obj.pos_w[2] < self.threshold
    
    @property
    def is_success(self):
        return False
    
    @property
    def is_failure(self):
        return True
```

### MaxCollision

**File:** `termination_conditions/max_collision.py` (~2 KB)

**Purpose:** Terminates if collision count exceeds a threshold. Always a failure.

```python
class MaxCollision(BaseTerminationCondition):
    def __init__(self, config):
        self.max_collisions = config.get("max_collisions", 100)
        self.collision_count = 0
    
    def evaluate(self, env, scene, robots, task):
        # Count collisions since last step
        self.collision_count += scene.get_collision_count()
        return self.collision_count >= self.max_collisions
    
    @property
    def is_success(self):
        return False
    
    @property
    def is_failure(self):
        return True
```

### PointGoal

**File:** `termination_conditions/point_goal.py` (~2 KB)

**Purpose:** Terminates successfully when the robot reaches a target point. Always a success.

```python
class PointGoal(BaseTerminationCondition):
    def __init__(self, config):
        self.target_position = np.array(config["target_position"])
        self.distance_threshold = config.get("distance_threshold", 0.1)
    
    def evaluate(self, env, scene, robots, task):
        robot_base_pos = robots[0].get_base_position()
        distance = np.linalg.norm(robot_base_pos - self.target_position)
        return distance < self.distance_threshold
    
    @property
    def is_success(self):
        return True
    
    @property
    def is_failure(self):
        return False
```

### ReachingGoal

**File:** `termination_conditions/reaching_goal.py` (~1 KB)

**Purpose:** Terminates successfully when the robot's end-effector reaches a target position.

```python
class ReachingGoal(BaseTerminationCondition):
    def __init__(self, config):
        self.target_position = np.array(config["target_position"])
        self.distance_threshold = config.get("distance_threshold", 0.05)
    
    def evaluate(self, env, scene, robots, task):
        eef_pos = robots[0].get_ee_position()
        distance = np.linalg.norm(eef_pos - self.target_position)
        return distance < self.distance_threshold
    
    @property
    def is_success(self):
        return True
    
    @property
    def is_failure(self):
        return False
```

### GraspGoal

**File:** `termination_conditions/grasp_goal.py` (~600 B)

**Purpose:** Terminates successfully when the robot grasps a target object.

```python
class GraspGoal(BaseTerminationCondition):
    def __init__(self, config):
        self.object_name = config["object_name"]
    
    def evaluate(self, env, scene, robots, task):
        robot = robots[0]
        # Check if robot is grasping the target object
        return robot.is_grasping(self.object_name)
    
    @property
    def is_success(self):
        return True
    
    @property
    def is_failure(self):
        return False
```

### PredicateGoal

**File:** `termination_conditions/predicate_goal.py` (~1 KB)

**Purpose:** Terminates successfully when BDDL goal predicates are satisfied. Used by BehaviorTask.

```python
class PredicateGoal(BaseTerminationCondition):
    def __init__(self, config):
        self.goal_conditions = config.get("goal_conditions", [])
    
    def evaluate(self, env, scene, robots, task):
        # Evaluate BDDL goal conditions against scene state
        success, _ = evaluate_state(
            self.goal_conditions,
            lambda pred_cls, *args: scene.check_predicate(pred_cls, *args)
        )
        return success
    
    @property
    def is_success(self):
        return True
    
    @property
    def is_failure(self):
        return False
```

## Integration with Task

Tasks configure termination conditions:

```python
class BaseTask:
    def _setup_termination_conditions(self):
        config = self.cfg.get("termination_config", {})
        term_cls = REGISTERED_TERMINATION_CONDITIONS[config["type"]]
        self.termination_conditions.append(term_cls(config))
    
    def check_termination(self):
        for condition in self.termination_conditions:
            if condition.evaluate(self.env, self.scene, self.robots, self):
                return condition.is_success, condition.is_failure
        return False, False
```

## Environment Integration

The environment calls termination checks each step:

```python
class Environment:
    def step(self, action):
        obs, reward, terminated, truncated, info = self.base_env.step(action)
        
        # Check task termination
        success, failure = self.task.check_termination()
        if success:
            terminated = True
            info["success"] = True
        elif failure:
            terminated = True
            info["failure"] = True
        
        return obs, reward, terminated, truncated, info
```

## Testing

- `test_envs.py` — Tests environment termination
- `test_behavior_task.py` — Tests predicate goal termination
- `test_examples.py` — Tests various termination conditions in examples

## See Also

<!-- openwiki: broken internal link [omnigibson/tasks.md] file "omnigibson/tasks.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/tasks.md`](omnigibson/tasks.md) — Task termination integration
<!-- openwiki: broken internal link [omnigibson/reward_functions.md] file "omnigibson/reward_functions.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/reward_functions.md`](omnigibson/reward_functions.md) — Reward function integration
<!-- openwiki: broken internal link [omnigibson/registries.md] file "omnigibson/registries.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/registries.md`](omnigibson/registries.md) — Termination registry
