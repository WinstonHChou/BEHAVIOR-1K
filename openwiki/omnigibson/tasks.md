---
type: wiki-component
title: OmniGibson Tasks
description: BaseTask, BehaviorTask, GraspTask, PointNavigationTask, PointReachingTask, DummyTask — RL task definitions, goal sampling, and BDDL integration.
tags: [omnigibson, tasks, rl, behavior-task,bddl]
---

# OmniGibson Tasks

**Tasks** define the goal, reward structure, and termination conditions for a simulation episode. They sit between the environment and the underlying scene/robots, providing the high-level objective.

## Registry

```python
from omnigibson.tasks import (
    BaseTask,
    BehaviorTask,
    GraspTask,
    PointNavigationTask,
    PointReachingTask,
    DummyTask,
    REGISTERED_TASKS,
)
```

## Class Hierarchy

```
BaseTask (omnigibson/tasks/base_task.py, 17 KB, ABC)
│
├── BehaviorTask (36 KB) — BEHAVIOR-style task from BDDL (primary task type)
├── GraspTask (10 KB) — Grasping task
├── PointNavigationTask (20 KB) — Navigation to point goal
├── PointReachingTask (7 KB) — Reaching to point goal
└── DummyTask (1 KB) — Minimal no-op task
```

## BaseTask

**File:** `tasks/base_task.py` (~17 KB)

**Purpose:** Abstract base class for all tasks. Defines the common interface and handles reward/termination configuration.

### Key Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `name` | str | Task name |
| `scene` | Scene | Reference to the parent scene |
| `robots` | list[Robot] | Reference to robots in scene |
| `termination_config` | dict | Termination condition configuration |
| `reward_config` | dict | Reward function configuration |
| `metrics_config` | dict | Metric configuration |
| `goal_info` | dict | Task-specific goal information |

### Key Methods

| Method | Return Type | Description |
|--------|-------------|-------------|
| `load()` | None | Initialize task state, sample goals |
| `step()` | None | Update task state each step |
| `reset()` | None | Reset task state, re-sample goals |
| `check_termination()` | bool | Check if episode should end |
| `compute_reward()` | float | Compute reward for current state |
| `get_metrics()` | dict | Get task-level metrics |
| `get_observations()` | dict | Task-specific observations |

### Task Lifecycle

```python
class BaseTask:
    def load(self):
        """Load task state, sample initial goals."""
        self._sample_goals()
        self._setup_termination_conditions()
        self._setup_reward_functions()
    
    def step(self):
        """Update task state each step."""
        self._update_goals()
        self._compute_reward()
        self._check_termination()
    
    def reset(self):
        """Reset task state."""
        self._sample_goals()
        self._reset_metrics()
    
    def check_termination(self):
        """Check if episode should end."""
        for condition in self.termination_conditions:
            if condition.evaluate():
                return True
        return False
    
    def compute_reward(self):
        """Compute reward for current step."""
        total_reward = 0.0
        for reward_fn in self.reward_functions:
            total_reward += reward_fn.compute()
        return total_reward
```

## BehaviorTask

**File:** `tasks/behavior_task.py` (~36 KB)

**Purpose:** The primary task type for BEHAVIOR-1K. Implements BDDL-style tasks with symbolic goal conditions.

### BDDL Integration

BehaviorTask integrates with the BDDL3 knowledge base:

```python
from bddl.knowledge_base import KnowledgeBase

class BehaviorTask(BaseTask):
    def load(self):
        # Load knowledge base
        self.kb = KnowledgeBase(populate=True)
        
        # Sample a task from KB
        self.task = self.kb.sample_task()
        
        # Compile task for current scene
        self.compiled_task = self.task.compile(scene_layout=self._get_scene_layout())
        
        # Set up termination based on BDDL goal conditions
        self.termination_conditions.append(
            PredicateGoal(goal_conditions=self.compiled_task.goal_conditions)
        )
```

### Goal Sampling

```python
class BehaviorTask(BaseTask):
    def _sample_goals(self):
        """Sample BDDL goal conditions."""
        self.goal_conditions = self.compiled_task.goal_conditions
        
        # Compute ground goal state options (all possible solutions)
        self.ground_goal_options = get_ground_goal_state_options(
            self.compiled_task, self.scene.objects
        )
        
        # Compute reward based on progress
        self.reward_functions.append(
            ProgressiveReward(
                ground_goal_options=self.ground_goal_options,
                evaluate_fn=self._evaluate_predicate
            )
        )
    
    def _evaluate_predicate(self, predicate_cls, *entity_names):
        """Evaluate a BDDL predicate against current state."""
        state = self.scene.get_state(predicate_cls, *entity_names)
        return state is not None
```

### Scene Layout

BehaviorTask extracts scene layout from the scene:

```python
def _get_scene_layout(self):
    """Extract scene layout for wildcard expansion."""
    layout = {}
    for obj in self.scene.objects:
        category = obj.category
        count = sum(1 for o in self.scene.objects if o.category == category)
        layout[category] = max(layout.get(category, 0), count)
    return layout
```

## GraspTask

**File:** `tasks/grasp_task.py` (~10 KB)

**Purpose:** Simple grasping task. Robot must grasp a specified object.

### Configuration

```yaml
task:
  type: GraspTask
  target_object: "bowl.n.01_1"
  termination_config:
    type: GraspGoal
  reward_config:
    type: GraspReward
```

## PointNavigationTask

**File:** `tasks/point_navigation_task.py` (~20 KB)

**Purpose:** Robot must navigate to a target point.

### Configuration

```yaml
task:
  type: PointNavigationTask
  target_position: [1.0, 1.0, 0.0]
  termination_config:
    type: PointGoal
    distance_threshold: 0.1
  reward_config:
    type: PointGoalReward
```

## PointReachingTask

**File:** `tasks/point_reaching_task.py` (~7 KB)

**Purpose:** Robot must reach a target point with its end-effector.

```yaml
task:
  type: PointReachingTask
  target_position: [0.5, 0.5, 0.5]
  termination_config:
    type: ReachingGoal
    distance_threshold: 0.05
  reward_config:
    type: ReachingGoalReward
```

## DummyTask

**File:** `tasks/dummy_task.py` (~1 KB)

**Purpose:** Minimal no-op task for testing. Always returns zero reward and never terminates.

## Termination & Reward Integration

Tasks configure termination conditions and reward functions:

```python
# Task creates termination conditions from config
def _setup_termination_conditions(self):
    config = self.cfg.get("termination_config", {})
    term_cls = REGISTERED_TERMINATION_CONDITIONS[config["type"]]
    self.termination_conditions.append(term_cls(config))

# Task creates reward functions from config
def _setup_reward_functions(self):
    config = self.cfg.get("reward_config", {})
    reward_cls = REGISTERED_REWARD_FUNCTIONS[config["type"]]
    self.reward_functions.append(reward_cls(config))
```

## Task Observations

Tasks can provide additional observations beyond sensor data:

```python
def get_observations(self):
    obs = super().get_observations()
    obs["task_info"] = {
        "goal": self.goal_info,
        "reward": self._current_reward,
    }
    return obs
```

## Testing

- `test_behavior_task.py` — Tests BehaviorTask sampling and evaluation
- `test_examples.py` — Tests task creation in environments
- `benchmark/benchmark_interactive_scene.py` — Task load/step timing

## See Also

<!-- openwiki: broken internal link [omnigibson/termination.md] file "omnigibson/termination.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/termination.md`](omnigibson/termination.md) — Termination conditions
<!-- openwiki: broken internal link [omnigibson/reward_functions.md] file "omnigibson/reward_functions.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/reward_functions.md`](omnigibson/reward_functions.md) — Reward functions
<!-- openwiki: broken internal link [../cross-system/omnigibson-bddl-integration.md] file "../cross-system/omnigibson-bddl-integration.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`cross-system/omnigibson-bddl-integration.md`](../cross-system/omnigibson-bddl-integration.md) — BDDL integration
