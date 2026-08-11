---
type: wiki-component
title: OmniGibson Tests
description: Test strategy — 25+ test files, CI test matrix, skipped tests, conftest.py fixtures, snapshot tests, benchmark tests.
tags: [omnigibson, tests, testing, pytest, snapshots]
---

# OmniGibson Tests

OmniGibson uses pytest for testing. Tests cover all major subsystems: controllers, sensors, objects, object states, environments, tasks, primitives, and more.

## Test Directory Structure

```
OmniGibson/tests/
├── conftest.py                   # Pytest fixtures (423 lines)
├── requirements-tests.txt        # Test dependencies
├── utils.py                      # Test utilities
├── benchmark/                    # Performance benchmarks
│   ├── benchmark_interactive_scene.py
│   ├── benchmark_object_count.py
│   └── profiling.py
├── snapshots/golden/             # Expected output images (PNG/NPY)
├── data/                         # Test data (YAML configs)
│   └── r1_pro_source_config.yaml
├── test_ag_states.py             # Attention Graph state tests (12 KB)
├── test_behavior_task.py         # BEHAVIOR task sampling tests
├── test_controllers.py           # Controller tests (24 KB)
├── test_curobo.py                # Curobo motion planner tests (21 KB) [SKIP]
├── test_data_collection.py       # Data collection tests (27 KB)
├── test_dump_load_states.py      # Save/load state tests
├── test_envs.py                  # Environment tests
├── test_examples.py              # Example tests
├── test_multiple_envs.py         # Multi-environment tests (21 KB)
├── test_object_removal.py        # Object lifecycle tests
├── test_object_states.py         # Object state tests (51 KB)
├── test_primitives.py            # Action primitives tests (6 KB)
├── test_robot_states_flatcache.py # Robot flat cache tests (17 KB)
├── test_robot_teleoperation.py   # Teleoperation tests
├── test_scene_graph.py           # Scene graph tests (4 KB)
├── test_sensors.py               # Sensor tests (5 KB)
├── test_snapshots.py             # Snapshot tests (7 KB)
├── test_symbolic_primitives.py   # Symbolic primitives tests (12 KB)
├── test_systems.py               # Particle system tests (2 KB)
├── test_transform_utils.py       # Transform utility tests (20 KB)
├── test_transition_rules.py      # Transition rule tests (59 KB)
├── test_ag_states.py             # Attention graph state tests (12 KB)
└── update_example_tests.py       # Example test utilities
```

## Running Tests

Tests require an NVIDIA RTX GPU (2080Ti+) and Isaac Sim runtime:

```bash
# Run all tests
cd OmniGibson
OMNIGIBSON_HEADLESS=1 pytest tests/

# Run specific test file
OMNIGIBSON_HEADLESS=1 pytest tests/test_object_states.py

# Run specific test by name
OMNIGIBSON_HEADLESS=1 pytest tests/test_envs.py -k "test_environment_creation"

# Run with verbose output
OMNIGIBSON_HEADLESS=1 pytest tests/ -v

# Run with coverage
OMNIGIBSON_HEADLESS=1 pytest tests/ --cov=omnigibson --cov-report=html
```

## Test Fixtures (conftest.py)

`conftest.py` (~423 lines) provides ~30 pytest fixtures:

### Environment Fixtures

| Fixture | Description |
|---------|-------------|
| `stopped_env` | Stopped environment (not yet playing) |
| `env` | Fully initialized environment |
| `stopped_vec_env` | Stopped vectorized environment |
| `vec_env` | Fully initialized vectorized environment |

### Robot Fixtures

| Fixture | Description |
|---------|-------------|
| `robot` | Pre-loaded robot (Locobot) |
| `assisted_robot` | Robot with assisted grasping |
| `multi_arm_robot` | Robot with multiple arms |

### Object Fixtures

~30 object fixtures for testing object states:

| Fixture | Description |
|---------|-------------|
| `microwave` | Microwave oven object |
| `stove` | Stove object |
| `fridge` | Refrigerator object |
| `bowl` | Bowl object |
| `bagel` | Bagel object |
| `apple` | Apple object |
| `pot` | Pot object |
| `sink` | Sink object |
| `faucet` | Faucet object |
| `cabinet` | Cabinet object |
| `drawer` | Drawer object |
| `table` | Table object |
| `counter` | Counter object |
| `water` | Water object |
| `sponge` | Sponge object |
| `knife` | Knife object |
| `cloth` | Cloth object |
| `towel` | Towel object |
| `cup` | Cup object |
| `plate` | Plate object |
| `spoon` | Spoon object |
| `fork` | Fork object |
| `glass` | Glass object |
| `mug` | Mug object |
| `bottle` | Bottle object |
| `jar` | Jar object |
| `pan` | Pan object |
| `tray` | Tray object |
| `basket` | Basket object |
| `box` | Box object |

### Example: Using Fixtures

```python
import pytest

@pytest.fixture
def env(stopped_env):
    stopped_env.sim.play()
    return stopped_env

def test_object_state_creation(env):
    """Test that object states are created correctly."""
    robot = env.robots[0]
    assert robot is not None
    obs = robot.get_observations()
    assert "eef_0_pos" in obs
```

## Test Categories

### Controller Tests (`test_controllers.py` — 24 KB)

Tests all controller types:
- `test_ik_controller` — IK solution accuracy
- `test_osc_controller` — OSC force control
- `test_joint_controller` — Joint position/velocity/effort
- `test_gripper_controller` — Gripper operations
- `test_dd_controller` — Differential drive
- `test_holonomic_controller` — Holonomic base

### Object State Tests (`test_object_states.py` — 51 KB)

Comprehensive tests for all state classes:
- State creation and initialization
- State update at each phase (pre-step, post-step, reset)
- State dependency ordering
- State removal and invalidation
- Contact-based state updates
- Temperature state transitions
- Cooking state transitions

### Transition Rule Tests (`test_transition_rules.py` — 59 KB)

Tests transition rule engine:
- Rule candidate finding
- Rule condition evaluation
- Rule execution (object creation/removal)
- Particle system triggers
- Disabled rules handling
- Temperature state integration
- Slicing/cooking/washing/mixing rules

### Snapshot Tests (`test_snapshots.py` — 7 KB)

Pixel-accurate output comparison using golden files:

```python
# test_snapshots.py
def test_vision_sensor_output(env):
    """Test vision sensor output against golden files."""
    sensor = env.scene.sensors[0]  # VisionSensor
    obs = sensor.get_observations()
    
    # Compare against golden PNG
    golden_path = "tests/snapshots/golden/vision_rgb.png"
    golden_data = load_png(golden_path)
    
    # Allow small numerical differences
    np.testing.assert_allclose(
        obs["rgb"], golden_data, rtol=0.01, atol=1
    )
```

### Benchmark Tests (`tests/benchmark/`)

Performance benchmarks:
- `benchmark_interactive_scene.py` — Scene load/step timing
- `benchmark_object_count.py` — Object count scalability
- `profiling.py` — Performance profiling with HTML reports

## Skipped Tests

Some tests are conditionally skipped:

| Test | Skip Condition | Reason |
|------|---------------|--------|
| `test_curobo.py` | Curobo not installed | External dependency |
| `test_data_collection.py` | No write permission | Requires filesystem access |
| `test_primitives.py` | No GPU | Requires GPU |

### Skipping with pytest

```python
import pytest

@pytest.mark.skipif(not HAS_CURBOBO, reason="Curobo not installed")
def test_curobo_planning():
    ...

@pytest.mark.skipif(not HAS_GPU, reason="No GPU available")
def test_gpu_operations():
    ...

@pytest.mark.skipif(not CAN_WRITE_DISK, reason="No write permission")
def test_data_collection():
    ...
```

## CI Test Matrix

GitHub Actions runs tests on self-hosted GPU runners:

```yaml
# .github/workflows/tests.yml
jobs:
  test:
    runs-on: [self-hosted, gpu]
    steps:
      - name: Run BDDL tests
        run: cd bddl3 && pytest tests/
      - name: Check example list
        run: python scripts/check_example_list.py
      - name: Run OmniGibson tests
        run: cd OmniGibson && OMNIGIBSON_HEADLESS=1 pytest tests/
```

## Testing Strategy

1. **Unit tests** — Test individual classes and functions (e.g., `test_transform_utils.py`)
2. **Integration tests** — Test component interactions (e.g., `test_object_states.py`)
3. **Regression tests** — Compare output against golden files (e.g., `test_snapshots.py`)
4. **Performance benchmarks** — Measure simulation performance (e.g., `tests/benchmark/`)
5. **Example tests** — Run all example scripts (e.g., `test_examples.py`)

## See Also

<!-- openwiki: broken internal link [omnigibson/tests.md] file "omnigibson/tests.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/tests.md`](omnigibson/tests.md) — Overview
<!-- openwiki: broken internal link [../ci-cd/workflows.md] file "../ci-cd/workflows.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`ci-cd/workflows.md`](../ci-cd/workflows.md) — CI configuration
