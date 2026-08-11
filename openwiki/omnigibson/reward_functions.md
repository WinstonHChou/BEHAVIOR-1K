---
type: wiki-component
title: OmniGibson Reward Functions
description: BaseRewardFunction, REGISTERED_REWARD_FUNCTIONS, each reward type (PointGoalReward, CollisionReward, GraspReward, PotentialReward, ReachingGoalReward) with implementation details.
tags: [omnigibson, rewards, reinforcement-learning, shaping]
---

# OmniGibson Reward Functions

**Reward functions** compute step-level rewards for reinforcement learning. They are configurable per-task and registered in a global registry.

## Registry

```python
from omnigibson.reward_functions import (
    BaseRewardFunction,
    REGISTERED_REWARD_FUNCTIONS,
)
```

## Class Hierarchy

```
BaseRewardFunction (omnigibson/reward_functions/reward_function_base.py, 3 KB, Registerable metaclass)
│
├── PointGoalReward (800 B)
├── ReachingGoalReward (1 KB)
├── GraspReward (7 KB)
├── CollisionReward (2 KB)
└── PotentialReward (2 KB)
```

## BaseRewardFunction

**File:** `reward_functions/reward_function_base.py` (~3 KB)

**Purpose:** Abstract base class for all reward functions. Uses `Registerable` metaclass for registration.

```python
class BaseRewardFunction(ABC, metaclass=Registerable):
    """Abstract base for reward functions."""
    
    def __init__(self, config):
        self.config = config
    
    @abstractmethod
    def compute(self, env, scene, robots, task):
        """Compute reward for current state. Returns float."""
        pass
    
    @abstractmethod
    def reset(self):
        """Reset internal state."""
        pass
```

## Registered Rewards

```python
REGISTERED_REWARD_FUNCTIONS = {
    "PointGoalReward": PointGoalReward,
    "ReachingGoalReward": ReachingGoalReward,
    "GraspReward": GraspReward,
    "CollisionReward": CollisionReward,
    "PotentialReward": PotentialReward,
}
```

## Reward Types

### PointGoalReward

**File:** `reward_functions/point_goal_reward.py` (~800 B)

**Purpose:** Reward based on robot base distance to a target point. Higher reward when closer.

```python
class PointGoalReward(BaseRewardFunction):
    def compute(self, env, scene, robots, task):
        robot_base_pos = robots[0].get_base_position()
        target_pos = np.array(self.config["target_position"])
        distance = np.linalg.norm(robot_base_pos - target_pos)
        
        # Linear reward: 1.0 at target, 0.0 beyond threshold
        threshold = self.config.get("distance_threshold", 0.1)
        if distance <= threshold:
            return 1.0
        return max(0.0, 1.0 - distance)
    
    def reset(self):
        self.prev_distance = None
```

### ReachingGoalReward

**File:** `reward_functions/reaching_goal_reward.py` (~1 KB)

**Purpose:** Reward based on end-effector distance to target position.

```python
class ReachingGoalReward(BaseRewardFunction):
    def compute(self, env, scene, robots, task):
        eef_pos = robots[0].get_ee_position()
        target_pos = np.array(self.config["target_position"])
        distance = np.linalg.norm(eef_pos - target_pos)
        
        threshold = self.config.get("distance_threshold", 0.05)
        if distance <= threshold:
            return 1.0
        return max(0.0, 1.0 - distance / threshold)
    
    def reset(self):
        self.prev_distance = None
```

### GraspReward

**File:** `reward_functions/grasp_reward.py` (~7 KB)

**Purpose:** Reward for grasping an object. Provides:
- **Dense reward** based on approach progress
- **Sparse reward** for successful grasp
- **Negative reward** for collision during grasp

```python
class GraspReward(BaseRewardFunction):
    def compute(self, env, scene, robots, task):
        robot = robots[0]
        target_obj = scene.get_object(self.config["target_object"])
        
        # Check grasp status
        if robot.is_grasping(target_obj.name):
            return 1.0  # Successful grasp
        
        # Dense approach reward
        eef_pos = robot.get_ee_position()
        obj_pos = target_obj.pos_w
        distance = np.linalg.norm(eef_pos - obj_pos)
        
        threshold = self.config.get("grasp_threshold", 0.1)
        approach_reward = max(0.0, 1.0 - distance / threshold)
        
        # Collision penalty
        if scene.has_collision(robot, target_obj):
            return -0.1
        
        return approach_reward
    
    def reset(self):
        self.prev_grasp = False
```

### CollisionReward

**File:** `reward_functions/collision_reward.py` (~2 KB)

**Purpose:** Penalizes collisions between robot and scene objects.

```python
class CollisionReward(BaseRewardFunction):
    def compute(self, env, scene, robots, task):
        collision_count = scene.get_collision_count()
        
        if collision_count > 0:
            return -self.config.get("collision_penalty", -0.1)
        return 0.0
    
    def reset(self):
        self.prev_collision_count = 0
```

### PotentialReward

**File:** `reward_functions/potential_reward.py` (~2 KB)

**Purpose:** Potential-based reward shaping. Uses a potential function Φ(s) to shape rewards:

```
r = γΦ(s') - Φ(s) + r_sparse
```

This preserves optimal policies while providing dense feedback during training.

```python
class PotentialReward(BaseRewardFunction):
    def compute(self, env, scene, robots, task):
        # Compute potential at current and previous state
<!-- openwiki: broken internal link [state=env.get_obs(] file "state=env.get_obs(" does not exist. Fix the href or restore the target, then delete this comment. -->
        current_potential = self.config["potential_fn"](state=env.get_obs())
        
        if self.prev_potential is not None:
            reward = self.config.get("gamma", 1.0) * current_potential - self.prev_potential
        else:
            reward = 0.0
        
        self.prev_potential = current_potential
        return reward
    
    def reset(self):
        self.prev_potential = None
```

## Task Integration

Tasks configure reward functions:

```python
class BaseTask:
    def _setup_reward_functions(self):
        config = self.cfg.get("reward_config", {})
        reward_cls = REGISTERED_REWARD_FUNCTIONS[config["type"]]
        self.reward_functions.append(reward_cls(config))
    
    def compute_reward(self):
        total_reward = 0.0
        for reward_fn in self.reward_functions:
            total_reward += reward_fn.compute(self.env, self.scene, self.robots, self)
        return total_reward
```

## Custom Reward Functions

To create a custom reward function:

```python
from omnigibson.reward_functions import BaseRewardFunction, REGISTERED_REWARD_FUNCTIONS

@BaseRewardFunction.register()
class MyCustomReward(BaseRewardFunction):
    def compute(self, env, scene, robots, task):
        # Custom reward computation
        return 0.0
    
    def reset(self):
        pass
```

## Testing

- `test_envs.py` — Tests reward computation
- `test_behavior_task.py` — Tests BehaviorTask reward computation

## See Also

<!-- openwiki: broken internal link [omnigibson/tasks.md] file "omnigibson/tasks.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/tasks.md`](omnigibson/tasks.md) — Task reward integration
<!-- openwiki: broken internal link [omnigibson/termination.md] file "omnigibson/termination.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/termination.md`](omnigibson/termination.md) — Termination conditions
<!-- openwiki: broken internal link [omnigibson/registries.md] file "omnigibson/registries.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/registries.md`](omnigibson/registries.md) — Reward registry
