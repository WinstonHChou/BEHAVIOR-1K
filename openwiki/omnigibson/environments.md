---
type: wiki-component
title: OmniGibson Environment
description: Base Environment class — Gymnasium-compatible interface, config merging, step/reset lifecycle, observation/action spaces.
tags: [omnigibson, environment, gymnasium, rl-interface]
---

# OmniGibson Environment

**Environment** (defined in `envs/environment.py`, ~34 KB) is the main Gymnasium-compatible interface for interacting with a simulation. It wraps a scene, robots, and task into a single env that can be used for RL training, evaluation, or data collection.

## Registry

```python
from omnigibson.envs import Environment, REGISTERED_ENV_WRAPPERS
```

## Class Interface

```python
class Environment(gym.Env, GymObservable, Recreatable):
    """Main Gymnasium environment for OmniGibson."""
    
    def __init__(self, config):
        self.cfg = self._merge_config(config)
        self._init_simulator()  # og.sim or create new one
        self._load_variables()  # scene, robots, task
        self.load()
    
    def reset(self, **kwargs) -> tuple[dict, dict]:
        """Reset environment and return initial observation."""
        self._reset_scene()
        self._reset_agent()
        obs = self.get_obs()
        return obs, self.get_info()
    
    def step(self, action) -> tuple[dict, float, bool, bool, dict]:
        """Execute one environment step."""
        self._apply_action(action)
        self.sim.step()
        reward = self.task.compute_reward()
        terminated = self._check_termination()
        truncated = self._check_truncation()
        obs = self.get_obs()
        info = self.get_info()
        return obs, reward, terminated, truncated, info
    
    def close(self):
        """Clean up resources."""
        og.sim.stop()
    
    def get_obs(self) -> dict:
        """Collect observations from sensors and proprioception."""
        obs = {}
        for robot in self.robots:
            obs.update(robot.get_observations())
        for scene in self.scenes:
            for sensor in scene.sensors:
                obs.update(sensor.get_observations())
        return obs
    
    def get_info(self) -> dict:
        """Collect info dict (metrics, status, etc.)."""
        return {
            "reward": self.task.compute_reward(),
            "metrics": self.task.get_metrics(),
        }
```

## Configuration

Environments are created from a config dict or YAML path:

```python
import omnigibson as og

# From dict config
env = og.Environment(configs={
    "scene": {
        "type": "InteractiveTraversableScene",
        "model": "Rs_int",
    },
    "robots": [
        {
            "name": "Locobot",
            "model": "locobot",
            "action_type": "continuous",
            "controller_config": {
                "base": {"name": "DifferentialDriveController"},
                "trunk": {"name": "JointController"},
                "arm_0": {
                    "name": "InverseKinematicsController",
                    "subsume_controllers": ["trunk"],
                },
                "gripper_0": {"name": "MultiFingerGripperController"},
            },
            "sensor_config": {
                "VisionSensor": {
                    "sensor_kwargs": {
                        "image_height": 128,
                        "image_width": 128,
                        "min_distance": 0.01,
                        "max_distance": 10.0,
                    }
                }
            },
        }
    ],
    "task": {
        "type": "PointNavigationTask",
        "termination_config": {
            "type": "Timeout",
            "max_steps": 1000,
        },
        "reward_config": {
            "type": "PointGoalReward",
        },
    },
    "env": {
        "action_frequency": 30,
        "rendering_frequency": 30,
        "physics_frequency": 120,
    }
})

# From YAML path
env = og.Environment(configs="configs/default_cfg.yaml")
```

### Config Merging

Configs are merged using `merge_nested_dicts()`:

```python
def _merge_config(self, config):
    """Merge user config with default config."""
    merged = merge_nested_dicts(default_config, config)
    
    # Load external YAML files referenced in config
    for key in merged:
        if isinstance(merged[key], str) and merged[key].endswith(".yaml"):
            merged[key] = merge_nested_dicts(
                load_yaml(merged[key]),
                merged.get(key, {})
            )
    
    return merged
```

### Default Config (default_cfg.yaml)

```yaml
env:
  device: null
  action_frequency: 30
  rendering_frequency: 30
  physics_frequency: 120
  automatic_reset: true
  flatten_obs_space: true
  flatten_action_space: true

render:
  viewer_width: 1280
  viewer_height: 720

scene:
  type: Scene
  model: null

robots: []

objects: []

task:
  type: DummyTask
  termination_config:
    type: Timeout
    max_steps: 1000
  reward_config:
    type: PointGoalReward
```

## Lifecycle

### 1. Launch Simulator

```python
og.launch(
    gravity=9.81,
    physics_dt=1/120,
    rendering_dt=1/30,
    sim_step_dt=1/30,
    device="cuda:0",
    viewer_width=1280,
    viewer_height=720,
)
```

### 2. Create Environment

```python
env = og.Environment(configs=my_config)
```

This triggers:
- `_load_variables()`: Create scene, robots, task from config
- `load()`: Scene.load() + task.load() + sim.play()

### 3. Main Loop

```python
obs, info = env.reset()
for step in range(max_steps):
    action = policy.get_action(obs)  # External policy
    obs, reward, terminated, truncated, info = env.step(action)
    
    if terminated or truncated:
        obs, info = env.reset()
```

### 4. Close

```python
env.close()  # Stops simulator, cleans up resources
og.clear()  # Clears global state
```

## Observation/Action Spaces

### Observation Space

```python
obs = env.observation_space.sample()
# {
#     "VisionSensor_0": {"rgb": (128, 128, 3), "depth": (128, 128), ...},
#     "ScanSensor_0": {"scan": (360, 3)},
#     "eef_0_pos": (3,), "eef_0_quat": (4,),
#     "arm_0_qpos": (6,), "arm_0_qpos_sin": (6,), "arm_0_qpos_cos": (6,),
#     "gripper_0_qpos": (1,),
#     "base_qpos": (2,), "base_vel": (2,),
#     "grasp_main": (1,), "grasp_center": (3,), "grasp_success": (1,),
# }
```

### Action Space

```python
action = env.action_space.sample()
# [base_linear_vel, base_angular_vel,  # Base (2)
#  arm_eef_x, arm_eef_y, arm_eef_z,  # Arm position (3)
#  arm_quat_x, arm_quat_y, arm_quat_z, arm_quat_w,  # Arm orientation (4)
#  gripper_position,  # Gripper (1)
# ]
# Total: 10 dimensions for Locobot
```

## Multiple Environments

### Vector Environment

```python
from omnigibson.envs import VectorEnvironment

envs = VectorEnvironment(
    num_envs=4,
    env_fn=lambda: og.Environment(configs=my_config),
)

obs = envs.reset()
for _ in range(1000):
    actions = envs.action_space.sample()
    obs, rewards, dones, infos = envs.step(actions)
    if any(dones):
        obs = envs.reset()
```

### SB3 Vec Env

```python
from omnigibson.envs import SB3VecEnv

envs = SB3VecEnv(
    num_envs=8,
    env_fn=lambda: og.Environment(configs=my_config),
)
```

## Wrapper Composition

```python
base_env = og.Environment(configs=my_config)
data_env = HDF5DataWrapper(base_env, output_path="/data/episodes")
metrics_env = MetricsWrapper(data_env)

# Use the fully wrapped environment
obs, info = metrics_env.reset()
```

## Testing

- `test_envs.py` — Tests environment creation, reset, and step
- `test_multiple_envs.py` (~21 KB) — Tests multi-environment and vector envs
- `test_examples.py` — Tests example environment creation

## See Also

<!-- openwiki: broken internal link [omnigibson/environments.md] file "omnigibson/environments.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/environments.md`](omnigibson/environments.md) — Overview
<!-- openwiki: broken internal link [omnigibson/env_wrappers.md] file "omnigibson/env_wrappers.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/env_wrappers.md`](omnigibson/env_wrappers.md) — Wrapper system
<!-- openwiki: broken internal link [omnigibson/configs.md] file "omnigibson/configs.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/configs.md`](omnigibson/configs.md) — Config structure
