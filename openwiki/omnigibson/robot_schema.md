---
type: wiki-component
title: OmniGibson Robot Definition Schema
description: JSON schema for robot YAML definitions, the RobotDefinition and EndEffectorDefinition classes that validate robot configuration files.
tags: [omnigibson, robot, schema, configuration]
---

# OmniGibson Robot Definition Schema

**`definition_schema.py`** (~4.3 KB) defines the JSON schema used to validate robot YAML configuration files. It ensures that robot definitions conform to the expected structure before they are used to create `Robot` instances.

## Schema Structure

### `RobotDefinition`

The top-level schema defines the required and optional fields for a robot config:

```python
ROBOT_DEFINITION_SCHEMA = {
    "type": "object",
    "properties": {
        "robot": {
            "type": "object",
            "properties": {
                "name": {"type": "string"},
                "model": {"type": "string"},
                "action_type": {"enum": ["continuous", "discrete"]},
                "action_normalize": {"type": "boolean"},
                "grasping_mode": {"enum": ["physical", "assisted"]},
                "proprio_obs": {"type": "array", "items": {"type": "string"}},
                "controller_config": {"type": "object"},
                "sensor_config": {"type": "object"},
            },
            "required": ["name", "model", "controller_config"],
        }
    },
    "required": ["robot"],
}
```

### `EndEffectorDefinition`

Each arm's end-effector has its own schema:

```python
END_EFFECTOR_DEFINITION_SCHEMA = {
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "joint_names": {"type": "array", "items": {"type": "string"}},
        "link_names": {"type": "array", "items": {"type": "string"}},
        "ik_controller": {
            "type": "object",
            "properties": {
                "damping_alpha": {"type": "number"},
                "position_gain": {"type": "number"},
                "orientation_gain": {"type": "number"},
            }
        },
    },
    "required": ["name"],
}
```

## Validation Flow

When a robot is created from YAML, the schema is used to validate the configuration:

```python
from omnigibson.robots.definition_schema import RobotDefinition, validate_robot_config

config = load_yaml("fetch.yaml")
errors = validate_robot_config(config)
if errors:
    raise ValueError(f"Invalid robot config: {errors}")
```

Validation checks:
1. Required fields are present (`name`, `model`, `controller_config`)
2. Enum fields have valid values (`action_type`, `grasping_mode`)
3. Controller config references known controller types
4. Sensor configs specify valid sensor types

## Integration with Robot Class

The `Robot` class uses the schema to:

1. **Parse controller config** — Maps controller keys (base, trunk, arm_0, gripper_0) to controller names
2. **Validate proprio_obs** — Ensures observation keys are valid
3. **Determine DOF structure** — Infers the robot's kinematic chain from the controller config
4. **Set up subsume_controllers** — Handles controller hierarchy (e.g., IK subsuming trunk)

## Testing

Robot schema validation is tested indirectly through:
- `test_controllers.py` — Tests that controller configs are parsed correctly
- `test_examples.py` — Tests that example robot configs can create robots
- Environment creation tests that load robots from YAML

## See Also

<!-- openwiki: broken internal link [omnigibson/robots.md] file "omnigibson/robots.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/robots.md`](omnigibson/robots.md) — Robot class documentation
<!-- openwiki: broken internal link [omnigibson/configs.md] file "omnigibson/configs.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/configs.md`](omnigibson/configs.md) — Config structure
<!-- openwiki: broken internal link [omnigibson/controllers.md] file "omnigibson/controllers.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/controllers.md`](omnigibson/controllers.md) — Controller configuration
