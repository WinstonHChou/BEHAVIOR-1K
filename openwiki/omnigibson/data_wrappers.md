---
type: wiki-component
title: OmniGibson Data Wrappers
description: Deep-dive on HDF5DataWrapper, LeRobotDataWrapper — output formats, video recording, checkpointing, and episode filtering.
tags: [omnigibson, data-collection, hdf5, lerobot, video-recording]
---

# OmniGibson Data Wrappers

**Data wrappers** provide structured data collection for robot learning and dataset creation. They extend the `DataWrapper` base to save observations, actions, rewards, and metadata in formats suitable for downstream training.

## Registry

```python
from omnigibson.envs import (
    DataWrapper,
    HDF5DataWrapper,
    LeRobotDataWrapper,
    REGISTERED_ENV_WRAPPERS,
)
```

## HDF5DataWrapper

**File:** `envs/hdf5_data_wrapper.py` (~36 KB)

**Purpose:** Primary data collection format for BEHAVIOR dataset creation. Saves multi-modal observations (RGB, depth, segmentation) along with robot actions.

### Output Structure

```
{output_path}/
├── episode_0/
│   ├── observations/
│   │   ├── VisionSensor_0/
│   │   │   ├── rgb      # (T, 128, 128, 3) uint8
│   │   │   ├── depth    # (T, 128, 128) float32
│   │   │   └── seg_semantic  # (T, 128, 128) int32
│   │   └── ScanSensor_0/
│   │       └── scan     # (T, 360, 3) float32
│   ├── actions          # (T, D) float32 — Robot actions
│   ├── rewards          # (T,) float32 — Per-step rewards
│   ├── dones            # (T,) bool — Episode termination flags
│   ├── infos            # (T,) dict — Per-step metadata
│   └── metadata.json    # Episode metadata
├── episode_1/
│   └── ...
└── metadata.json        # Global dataset metadata
```

### Key Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `output_path` | str | "./data" | Directory for saved data |
| `only_successes` | bool | False | Only save successful episodes |
| `flush_every_n_traj` | int | 1 | Flush buffer every N episodes |
| `compression` | str | "gzip" | HDF5 compression type |
| `compression_level` | int | 5 | Compression level (1-9) |
| `record_video` | bool | False | Record video during collection |
| `video_path` | str | None | Video output path |
| `video_fps` | int | 30 | Video frame rate |

### Configuration

```python
wrapper = HDF5DataWrapper(
    env=env,
    output_path="/data/episodes",
    only_successes=True,
    flush_every_n_traj=10,
    compression="gzip",
    compression_level=5,
    record_video=True,
    video_path="/data/videos",
    video_fps=30,
)
```

### Checkpoint/Rollback

HDF5DataWrapper supports checkpointing for long-running data collection:

```python
# Save checkpoint
wrapper.save_checkpoint()  # Saves current buffer + metadata

# Rollback to checkpoint
wrapper.rollback_to_checkpoint(checkpoint_idx)  # Restores previous state

# List available checkpoints
checkpoints = wrapper.list_checkpoints()
```

### Video Recording

When `record_video=True`, RGB frames are collected and saved as MP4:

```python
# Video file structure
{video_path}/episode_0.mp4  # Video for episode 0
{video_path}/episode_1.mp4  # Video for episode 1
```

Video writing uses `create_video_writer()` and `write_video()` from `obs_utils.py`:

```python
from omnigibson.eval.utils.obs_utils import create_video_writer, write_video

video_writer = create_video_writer(video_path, fps=30)
write_video(video_writer, frames)  # frames: list of (H, W, 3) uint8 arrays
```

### Data Collection Flow

```
1. env.reset()
   → wrapper.reset()
   → Collect initial observation

2. env.step(action)
   → wrapper.step(action)
   → Collect obs, action, reward
   → If terminated/truncated:
     → wrapper.on_episode_end()
     → Save trajectory if successful (only_successes=True)
     → Increment episode counter
```

## LeRobotDataWrapper

**File:** `envs/lerobot_data_wrapper.py` (~20 KB)

**Purpose:** Collects data in [LeRobot](https://github.com/huggingface/lerobot) dataset format, enabling easy upload to Hugging Face Hub for training robot learning policies.

### LeRobot Format

```
dataset/
├── metadata.json
├── observations.image/
│   ├── wrist_left/   # (N, H, W, 3) uint8
│   ├── wrist_right/  # (N, H, W, 3) uint8
│   └── base/         # (N, H, W, 3) uint8
├── action            # (N, D) int16 — Quantized actions
├── is_success        # (N,) bool — Episode success flags
└── episode_index     # (N,) int32 — Episode indices
```

### Configuration

```python
wrapper = LeRobotDataWrapper(
    env=env,
    output_path="/data/lerobot_dataset",
    camera_names=["wrist_left", "wrist_right", "base"],
    action_quantization=True,  # Quantize actions to int16
)
```

### Key Differences from HDF5

| Feature | HDF5DataWrapper | LeRobotDataWrapper |
|---------|----------------|--------------------|
| Format | HDF5 | Parquet + PNG |
| Action encoding | float32 | int16 (quantized) |
| Storage efficiency | Moderate | High |
| HF Hub upload | Manual | Built-in |
| Compatible with | Custom training | Hugging Face LeRobot |

## DataWrapper Base Class

**File:** `envs/data_wrapper.py` (~39 KB)

### Core Interface

```python
class DataWrapper:
    def __init__(self, env, output_path="./data", only_successes=False, **kwargs):
        self.env = env
        self.output_path = output_path
        self.only_successes = only_successes
        self.current_episode_idx = 0
        self.buffer = []  # Current episode data
    
    def reset(self, **kwargs):
        obs = self.env.reset(**kwargs)
        self.buffer = [{"obs": obs, "action": None, "reward": 0.0, "done": False}]
        self.episode_rewards = [0.0]
        return obs
    
    def step(self, action):
        obs, reward, terminated, truncated, info = self.env.step(action)
        
        self.buffer.append({
            "obs": obs,
            "action": action,
            "reward": reward,
            "done": terminated or truncated,
            "info": info,
        })
        
        self.episode_rewards[-1] += reward
        
        if terminated or truncated:
            self.on_episode_end(terminated, truncated, info)
            self.current_episode_idx += 1
            self.buffer = []
        
        return obs, reward, terminated, truncated, info
    
    def on_episode_end(self, terminated, truncated, info):
        """Called when episode ends. Override in subclasses."""
        pass
    
    def flush_buffer(self):
        """Save buffered data. Override in subclasses."""
        pass
```

## Testing

- `test_data_collection.py` (~27 KB) — Comprehensive tests:
  - HDF5 file creation and reading
  - Multi-episode collection
  - Video recording
  - Checkpoint/rollback
  - Success filtering
  - LeRobot format validation
  - Compression verification

## See Also

<!-- openwiki: broken internal link [omnigibson/env_wrappers.md] file "omnigibson/env_wrappers.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/env_wrappers.md`](omnigibson/env_wrappers.md) — Wrapper hierarchy
<!-- openwiki: broken internal link [omnigibson/eval/overview.md] file "omnigibson/eval/overview.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/eval/overview.md`](omnigibson/eval/overview.md) — Evaluation data
<!-- openwiki: broken internal link [omnigibson/scripts.md] file "omnigibson/scripts.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/scripts.md`](omnigibson/scripts.md) — Data pipeline scripts
