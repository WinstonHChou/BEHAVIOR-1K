---
type: wiki-component
title: OmniGibson Environment Wrappers
description: EnvironmentWrapper, DataWrapper family, MetricsWrapper, HDF5/LeRobot wrappers, wrapper factory pattern for environment composition.
tags: [omnigibson, wrappers, environment, data-collection, metrics]
---

# OmniGibson Environment Wrappers

**Environment wrappers** compose additional behavior around a base `Environment` (or other wrapper). They enable data collection, metric tracking, and multi-environment management without modifying the base environment.

## Registry

```python
from omnigibson.envs import (
    EnvironmentWrapper,
    DataWrapper,
    HDF5DataWrapper,
    LeRobotDataWrapper,
    MetricsWrapper,
    REGISTERED_ENV_WRAPPERS,
)
```

## Class Hierarchy

```
EnvironmentWrapper (omnigibson/envs/env_wrapper.py, 3 KB, ABC)
│
├── DataWrapper (39 KB) — Base for data collection wrappers
│   ├── HDF5DataWrapper (36 KB) — HDF5 dataset collection
│   └── LeRobotDataWrapper (20 KB) — LeRobot dataset format
│
├── MetricsWrapper (3 KB) — Task metric collection
│
└── SB3VecEnv (5 KB) — Stable-Baselines3 VecEnv wrapper
```

## EnvironmentWrapper

**File:** `envs/env_wrapper.py` (~3 KB)

**Purpose:** Base wrapper class. All wrappers inherit from this and implement `reset()` and `step()`.

```python
class EnvironmentWrapper:
    """Base class for environment wrappers."""
    
    def __init__(self, env):
        self.env = env  # Wrapped environment
    
    def reset(self, **kwargs):
        return self.env.reset(**kwargs)
    
    def step(self, action):
        return self.env.step(action)
    
    def close(self):
        self.env.close()
    
    @property
    def action_space(self):
        return self.env.action_space
    
    @property
    def observation_space(self):
        return self.env.observation_space
    
    def __getattr__(self, name):
        return getattr(self.env, name)
```

## DataWrapper

**File:** `envs/data_wrapper.py` (~39 KB)

**Purpose:** Base class for data collection wrappers. Handles observation, action, and reward logging across episodes.

### Key Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `output_path` | str | Directory for saved data files |
| `only_successes` | bool | Only save successful episodes |
| `flush_every_n_traj` | int | Flush buffer every N trajectories |
| `buffer_size` | int | Buffer size before flushing |
| `current_traj_rewards` | list[float] | Current episode rewards |
| `current_traj_observations` | list[dict] | Current episode observations |

### Key Methods

| Method | Return Type | Description |
|--------|-------------|-------------|
| `collect_data()` | None | Collect current step data |
| `flush_buffer()` | None | Save buffered data to disk |
| `save_trajectory()` | None | Save a single trajectory |
| `reset()` | None | Reset collection state |
| `on_episode_end()` | None | Callback when episode ends |

### Data Collection Flow

```python
class DataWrapper:
    def reset(self, **kwargs):
        self.current_traj_rewards = []
        self.current_traj_observations = []
        return self.env.reset(**kwargs)
    
    def step(self, action):
        obs, reward, terminated, truncated, info = self.env.step(action)
        self.collect_data(obs, action, reward, info)
        
        if terminated or truncated:
            self.on_episode_end(terminated, truncated, info)
        
        return obs, reward, terminated, truncated, info
    
    def collect_data(self, obs, action, reward, info):
        self.current_traj_rewards.append(reward)
        self.current_traj_observations.append({
            "observation": obs,
            "action": action,
            "reward": reward,
            "info": info,
        })
        
        if len(self.current_traj_observations) >= self.flush_every_n_traj:
            self.flush_buffer()
```

## HDF5DataWrapper

**File:** `envs/hdf5_data_wrapper.py` (~36 KB)

**Purpose:** Collects data into HDF5 format. This is the primary data collection format for BEHAVIOR dataset creation.

### HDF5 Structure

```
episode_0/
├── observations/
│   ├── VisionSensor_0/
│   │   ├── rgb      # (T, H, W, 3) uint8
│   │   ├── depth    # (T, H, W) float32
│   │   └── seg_semantic  # (T, H, W) int32
│   └── ScanSensor_0/
│       └── scan     # (T, N, 3) float32
├── actions          # (T, D) float32
├── rewards          # (T,) float32
├── dones            # (T,) bool
├── infos            # (T,) dict (metadata)
└── metadata/
    ├── success      # bool
    ├── task_name    # str
    ├── robot_name   # str
    └── scene_name   # str
```

### Configuration

```python
wrapper = HDF5DataWrapper(
    env=env,
    output_path="/data/episodes",
    only_successes=True,
    flush_every_n_traj=10,
    compression="gzip",
    compression_level=5,
)
```

### Key Features

- **Checkpoint/rollback** — Saves intermediate checkpoints
- **Only successes** — Filter to save only successful episodes
- **Compression** — Gzip compression for storage efficiency
- **Multi-episode** — Supports collecting hundreds of episodes

## LeRobotDataWrapper

**File:** `envs/lerobot_data_wrapper.py` (~20 KB)

**Purpose:** Collects data in LeRobot dataset format for training robot learning policies.

### LeRobot Structure

```
dataset/
├── metadata.json    # Dataset metadata
├── observations.image/  # Image data
│   ├── wrist_left/
│   ├── wrist_right/
│   └── base/
├── action            # (N, D) actions
├── success           # (N,) success flags
└── is_success        # (N,) binary success flags
```

### Configuration

```python
wrapper = LeRobotDataWrapper(
    env=env,
    output_path="/data/lerobot_dataset",
    camera_names=["wrist_left", "wrist_right", "base"],
)
```

## MetricsWrapper

**File:** `envs/metrics_wrapper.py` (~3 KB)

**Purpose:** Collects task-level metrics during episodes. Aggregates metrics from the `omnigibson.metrics/` module.

```python
class MetricsWrapper:
    def __init__(self, env):
        self.env = env
        self.metrics = {}
    
    def step(self, action):
        obs, reward, terminated, truncated, info = self.env.step(action)
        
        # Collect metrics from info
        for metric_name, metric_value in info.get("metrics", {}).items():
            self._accumulate_metric(metric_name, metric_value)
        
        if terminated or truncated:
            self.metrics["episode"] = self._compute_episode_metrics()
        
        return obs, reward, terminated, truncated, info
```

## Wrapper Factory

Wrappers are applied via a factory pattern in the environment config:

```python
from omnigibson.envs.env_wrapper import create_wrapper

# Create a wrapper from config
wrapper_cls = REGISTERED_ENV_WRAPPERS["HDF5DataWrapper"]
wrapper = wrapper_cls(env=base_env, **wrapper_config)
```

### Applying Wrappers

```python
# Method 1: Direct wrapping
base_env = og.Environment(configs=my_config)
wrapped_env = HDF5DataWrapper(base_env, output_path="/data/episodes")

# Method 2: Stacked wrappers
base_env = og.Environment(configs=my_config)
data_env = HDF5DataWrapper(base_env, output_path="/data/episodes")
metrics_env = MetricsWrapper(data_env)

# Method 3: Config-based (preferred)
env = og.Environment(configs={
    "env": {
        "wrappers": [
            {
                "type": "HDF5DataWrapper",
                "output_path": "/data/episodes",
                "only_successes": True,
            }
        ]
    }
})
```

## Vectorized Wrappers

### SB3VecEnv

**File:** `envs/sb3_vec_env.py` (~5 KB)

**Purpose:** Wraps multiple environments into a Stable-Baselines3 VecEnv for RL training.

```python
from omnigibson.envs.sb3_vec_env import SB3VecEnv

envs = SB3VecEnv(
    num_envs=4,
    env_fn=lambda: og.Environment(configs=my_config),
)
```

### VectorEnvironment

**File:** `envs/vec_env_base.py` (~2 KB)

**Purpose:** Multi-subprocess vectorized environment base.

```python
from omnigibson.envs.vec_env_base import VectorEnvironment

envs = VectorEnvironment(
    num_envs=8,
    env_fn=lambda: og.Environment(configs=my_config),
)
```

## Testing

- `test_data_collection.py` (~27 KB) — Tests HDF5/LeRobot data collection:
  - Data writing and reading
  - Checkpoint/rollback functionality
  - Success filtering
  - Multi-episode collection
- `test_multiple_envs.py` (~21 KB) — Tests vectorized environments

## See Also

<!-- openwiki: broken internal link [omnigibson/environments.md] file "omnigibson/environments.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/environments.md`](omnigibson/environments.md) — Base environment
<!-- openwiki: broken internal link [omnigibson/data_wrappers.md] file "omnigibson/data_wrappers.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/data_wrappers.md`](omnigibson/data_wrappers.md) — Data collection deep-dive
<!-- openwiki: broken internal link [omnigibson/metrics.md] file "omnigibson/metrics.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/metrics.md`](omnigibson/metrics.md) — Metric system
