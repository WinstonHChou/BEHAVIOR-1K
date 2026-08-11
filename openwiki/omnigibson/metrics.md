---
type: wiki-component
title: OmniGibson Metrics
description: MetricBase, AgentMetric, TaskMetric — evaluation metrics for agent and task performance during episodes.
tags: [omnigibson, metrics, evaluation, performance]
---

# OmniGibson Metrics

**Metrics** provide structured evaluation of agent and task performance during episodes. They are accumulated over episode steps and reported at episode end.

## Registry

```python
from omnigibson.metrics import (
    MetricBase,
    AgentMetric,
    TaskMetric,
)
```

## Class Hierarchy

```
MetricBase (omnigibson/metrics/metric_base.py, 5 KB, ABC)
│
├── AgentMetric (2 KB) — Per-episode agent metrics
└── TaskMetric (2 KB) — Task-level metrics
```

## MetricBase

**File:** `metrics/metric_base.py` (~5 KB)

**Purpose:** Abstract base class for all metrics.

```python
class MetricBase(ABC):
    """Base class for metrics."""
    
    def __init__(self, name):
        self.name = name
        self.value = 0.0
        self.steps = 0
    
    @abstractmethod
    def update(self, env, scene, robots, task, obs, action, reward, info):
        """Update metric value based on current step."""
        pass
    
    @abstractmethod
    def get_value(self):
        """Return current metric value."""
        pass
    
    def reset(self):
        """Reset metric for new episode."""
        self.value = 0.0
        self.steps = 0
```

## AgentMetric

**File:** `metrics/agent_metric.py` (~2 KB)

**Purpose:** Metrics that measure agent performance (e.g., distance traveled, time spent, energy used).

```python
class AgentMetric(MetricBase):
    """Metric that measures agent performance."""
    
    def update(self, env, scene, robots, task, obs, action, reward, info):
        self.steps += 1
        self._compute_step_metric(obs, action, reward, info)
    
    def _compute_step_metric(self, obs, action, reward, info):
        """Override to compute per-step metric contribution."""
        pass
```

### Common Agent Metrics

| Metric | Description | Computation |
|--------|-------------|-------------|
| `steps` | Total steps in episode | Counter increment |
| `distance_traveled` | Total distance moved by robot base | Sum of base position deltas |
| `time_spent` | Total simulation time | Step count × step dt |
| `energy_used` | Estimated energy consumption | Sum of joint effort × movement |
| `grasp_attempts` | Number of grasp attempts | Count of gripper closure events |
| `grasp_successes` | Number of successful grasps | Count of IsGrasping state changes |

## TaskMetric

**File:** `metrics/task_metric.py` (~2 KB)

**Purpose:** Metrics that measure task-specific performance (e.g., task completion rate, predicate satisfaction).

```python
class TaskMetric(MetricBase):
    """Metric that measures task-specific performance."""
    
    def update(self, env, scene, robots, task, obs, action, reward, info):
        self.steps += 1
        self._compute_task_metric(task, scene, robots)
```

### Common Task Metrics

| Metric | Description | Computation |
|--------|-------------|-------------|
| `predicates_satisfied` | Number of goal predicates satisfied | Count of TRUE BDDL predicates |
| `predicates_total` | Total number of goal predicates | Count of all goal predicates |
| `progress` | Task progress percentage | predicates_satisfied / predicates_total |
| `success` | Whether episode succeeded | Boolean from termination check |
| `task_time` | Time to complete task | Steps × step dt if success, else NaN |

## MetricsWrapper Integration

Metrics are collected via `MetricsWrapper`:

```python
from omnigibson.envs import MetricsWrapper

env = MetricsWrapper(base_env)

# Metrics are added to env.metrics dict
env.metrics["steps"] = StepMetric("steps")
env.metrics["distance"] = AgentMetric("distance_traveled")
env.metrics["task_progress"] = TaskMetric("task_progress")

# During stepping
def step(self, action):
    obs, reward, terminated, truncated, info = self.env.step(action)
    
    # Update all metrics
    for metric in self.metrics.values():
        metric.update(self.env, self.scene, self.robots, self.env.task, obs, action, reward, info)
    
    # Add metrics to info
    if terminated or truncated:
        info["metrics"] = {name: m.get_value() for name, m in self.metrics.items()}
    
    return obs, reward, terminated, truncated, info
```

## Testing

- `test_envs.py` — Tests metric collection during episodes
- `test_examples.py` — Tests metric output in example runs

## See Also

<!-- openwiki: broken internal link [omnigibson/env_wrappers.md] file "omnigibson/env_wrappers.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/env_wrappers.md`](omnigibson/env_wrappers.md) — MetricsWrapper
<!-- openwiki: broken internal link [omnigibson/tasks.md] file "omnigibson/tasks.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/tasks.md`](omnigibson/tasks.md) — Task integration
<!-- openwiki: broken internal link [omnigibson/eval/overview.md] file "omnigibson/eval/overview.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/eval/overview.md`](omnigibson/eval/overview.md) — Evaluation metrics
