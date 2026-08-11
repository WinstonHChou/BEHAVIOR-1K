---
type: wiki-component
title: OmniGibson Sensors
description: VisionSensor (RGB, depth, seg, normal), ScanSensor (LiDAR), sensor modalities, sensor noise, and observation space configuration.
tags: [omnigibson, sensors, vision, lidar, observation]
---

# OmniGibson Sensors

**Sensors** provide observation data for agents and visualization. OmniGibson supports vision sensors (camera-based) and scan sensors (LiDAR/ray-cast).

## Registry

```python
from omnigibson.sensors import (
    VisionSensor,
    ScanSensor,
    ALL_SENSOR_MODALITIES,
)
```

## Class Hierarchy

```
BaseSensor (omnigibson/sensors/base_sensor.py, 8 KB, ABC)
│
├── VisionSensor (46 KB) — Camera-based sensors (RGB, depth, seg, etc.)
├── ScanSensor (19 KB) — LiDAR/ray-cast sensors
│
└── Sensor Noise:
    └── BaseSensorNoise (3 KB)
        └── DropoutSensorNoise (3 KB)
```

## VisionSensor

**File:** `sensors/vision_sensor.py` (~46 KB)

**Purpose:** Camera-based sensor that captures RGB, depth, semantic segmentation, instance segmentation, and surface normal images.

### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `name` | str | Sensor name (e.g., "VisionSensor_0") |
| `image_height` | int | Image height |
| `image_width` | int | Image width |
| `min_distance` | float | Minimum capture distance |
| `max_distance` | float | Maximum capture distance |
| `clip_near` | float | Near clipping plane |
| `clip_far` | float | Far clipping plane |
| `semantic_segmentation` | bool | Enable seg_semantic output |
| `instance_segmentation` | bool | Enable seg_instance output |
| `depth` | bool | Enable depth output |
| `rgb` | bool | Enable RGB output |
| `normal` | bool | Enable surface normal output |
| `fov` | float | Field of view (degrees) |
| `position` | np.ndarray | Sensor position in world space |
| `orientation` | np.ndarray | Sensor orientation in world space |

### Sensor Modalities

```python
ALL_SENSOR_MODALITIES = {
    "rgb",
    "depth",
    "seg_semantic",
    "seg_instance",
    "normal",
}
```

### Configuration

```python
# In robot config:
sensor_config:
  VisionSensor:
    name: VisionSensor_0
    sensor_kwargs:
      image_height: 128
      image_width: 128
      min_distance: 0.01
      max_distance: 10.0
      semantic_segmentation: true
      instance_segmentation: true
      depth: true
      rgb: true
      position: [0.0, 0.0, 0.5]  # Relative to robot link
      orientation: [0.0, 0.0, 0.0]
```

### Output

```python
obs = sensor.get_observations()
# Returns dict with enabled modalities:
# {
#     "rgb": (H, W, 3) — uint8 [0, 255]
#     "depth": (H, W) — float32 meters
#     "seg_semantic": (H, W) — int32 semantic IDs
#     "seg_instance": (H, W) — int32 instance IDs
#     "normal": (H, W, 3) — float32 surface normals
# }
```

## ScanSensor

**File:** `sensors/scan_sensor.py` (~19 KB)

**Purpose:** LiDAR/ray-cast sensor that returns point clouds.

### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `name` | str | Sensor name |
| `min_range` | float | Minimum range (meters) |
| `max_range` | float | Maximum range (meters) |
| `resolution` | int | Number of rays |
| `horizontal_fov` | float | Horizontal field of view (degrees) |
| `vertical_fov` | float | Vertical field of view (degrees) |
| `position` | np.ndarray | Sensor position |
| `orientation` | np.ndarray | Sensor orientation |

### Configuration

```python
sensor_config:
  ScanSensor:
    name: ScanSensor_0
    sensor_kwargs:
      min_range: 0.05
      max_range: 10.0
      resolution: 360
      horizontal_fov: 360.0
      vertical_fov: 10.0
      position: [0.0, 0.0, 0.1]
      orientation: [0.0, 0.0, 0.0]
```

### Output

```python
obs = sensor.get_observations()
# {
#     "scan": (N, 3) — float32 point cloud in sensor frame
# }
```

## Sensor Noise

### DropoutSensorNoise

**File:** `sensors/dropout_sensor_noise.py` (~3 KB)

**Purpose:** Randomly drops out sensor observations (sets to zeros/NaN).

```python
noise = DropoutSensorNoise(probability=0.1)  # 10% dropout rate
obs_noisy = noise.apply(sensor.get_observations())
```

## Sensor Placement

Sensors are placed relative to robot links:

```python
# Sensor is attached to a specific robot link
sensor = VisionSensor(
    name="VisionSensor_0",
    position=[0.0, 0.0, 0.5],  # Relative to link
    orientation=[0.0, 0.0, 0.0],
    attach_to=robot,
    attach_link="arm_0_link",  # Attach to specific link
)
```

## Sensor Data Reading

Sensors are read in `simulator.py`'s `_on_post_physics_step()`:

```python
class Simulator:
    def _on_post_physics_step(self):
        for scene in self.scenes:
            for sensor in scene.sensors:
                sensor.read()  # Read latest data
```

## Sensor Observation Keys

Sensor observations are included in the environment observation dict with keys matching sensor names:

```python
obs = {
    "VisionSensor_0": {
        "rgb": (128, 128, 3),
        "depth": (128, 128),
        "seg_semantic": (128, 128),
        "seg_instance": (128, 128),
        "normal": (128, 128, 3),
    },
    "ScanSensor_0": {
        "scan": (360, 3),
    },
    # ... proprioception ...
}
```

## Testing

- `test_sensors.py` (~5 KB) — Tests sensor data output:
  - Vision sensor RGB/depth/seg output validation
  - Scan sensor LiDAR output validation
  - Sensor noise application
  - Sensor observation space verification

## See Also

<!-- openwiki: broken internal link [omnigibson/environments.md] file "omnigibson/environments.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/environments.md`](omnigibson/environments.md) — Observation space structure
<!-- openwiki: broken internal link [omnigibson/robots.md] file "omnigibson/robots.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/robots.md`](omnigibson/robots.md) — Sensor attachment to robots
<!-- openwiki: broken internal link [omnigibson/configs.md] file "omnigibson/configs.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/configs.md`](omnigibson/configs.md) — Sensor configuration
