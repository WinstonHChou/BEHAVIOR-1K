---
type: wiki-component
title: OmniGibson Object Classes
description: USDObject base class, DatasetObject, PrimitiveObject, LightObject — object abstraction, state tracking, abilities, and the object registry.
tags: [omnigibson, objects, dynamics, states, registry]
---

# OmniGibson Object Classes

Objects in OmniGibson are physical entities in the simulation with positions, velocities, collision shapes, and dynamic states. They all inherit from `USDObject` (abstract base with `ABCMeta` + `Registerable`).

## Registry

```python
from omnigibson.objects import (
    USDObject,
    DatasetObject,
    PrimitiveObject,
    LightObject,
    REGISTERED_OBJECTS,
)
```

## Class Hierarchy

```
USDObject (omnigibson/objects/usd_object.py, 54 KB, ABCMeta + Registerable)
│
├── DatasetObject (5 files, 24 KB) — Pre-sampled BEHAVIOR objects with abilities
│   └── (loaded from dataset YAML/JSON, inherits all abilities from object taxonomy)
│
├── PrimitiveObject (18 KB) — Simple geometric primitives
│   ├── Box
│   ├── Sphere
│   └── Cylinder
│
└── LightObject (9 KB) — Light emitters
    ├── DomeLight
    ├── PointLight
    ├── SpotLight
    └── DirectionalLight
```

## USDObject

**File:** `objects/usd_object.py` (~54 KB)

**Purpose:** Abstract base class for all simulated objects. Defines the interface that all objects implement.

### Key Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `name` | str | Object name (unique in scene) |
| `model` | str | Model name (e.g., "bowl.n.01") |
| `category` | str | Object category (e.g., "bowl") |
| `pos_w` | np.ndarray | World-space position (N, 3) |
| `orn_w` | np.ndarray | World-space orientation (N, 4) |
| `vel_lin_w` | np.ndarray | Linear velocity (N, 3) |
| `vel_ang_w` | np.ndarray | Angular velocity (N, 3) |
| `states` | dict | Dynamic states dict (state class → args) |
| `abilities` | dict | `{ability_name: {param: value, ...}}` |
| `emitters` | list | Particle system emitters |
| `prims` | list | USD primitive references |
| `rigid_body_indices` | list | PhysX rigid body indices |
| `link_indices` | list | PhysX link indices |

### Key Methods

| Method | Return Type | Description |
|--------|-------------|-------------|
| `set_position(pos)` | None | Set world-space position |
| `set_orientation(orn)` | None | Set world-space orientation |
| `set_velocity(vel_lin, vel_ang)` | None | Set velocities |
| `get_state(state_cls)` | Any | Get specific state value |
| `set_state(state_cls, value)` | None | Set specific state value |
| `has_state(state_cls)` | bool | Check if state is active |
| `get_abilities()` | dict | Get all abilities |
| `add_state(state, *args)` | None | Add a dynamic state |
| `remove_state(state_cls)` | None | Remove a dynamic state |
| `step()` | None | Update object state (called each sim step) |
| `reset()` | None | Reset object to initial state |

### Dynamic States

Objects maintain a dictionary of dynamic states:

```python
self.states = {
    OnTop: {"target": "table.n.02_1", "offset": [0, 0, 0]},
    Filled: {"substance": "water", "amount": 0.5},
    Cooked: {"temperature": 85.0},
    Open: {"joint_position": 0.5},
    ...
}
```

States are managed by the `ObjectStateFactory` and updated by the state mechanics system.

### Abilities

Objects declare their abilities via the taxonomy:

```python
self.abilities = {
    "heatSource": {"heat_capacity": 500.0},
    "fillable": {"max_volume": 0.002},
    "slicable": {"slicing_threshold": 0.5},
    "openable": {"joint_limits": [0, 0.5]},
    ...
}
```

Abilities are inherited from the object taxonomy and determine which predicates can be satisfied.

## DatasetObject

**File:** `objects/dataset_object.py` (~24 KB)

**Purpose:** Pre-sampled BEHAVIOR-1K objects with abilities and states derived from the object taxonomy. These are the primary objects used in BEHAVIOR tasks.

### Creation

```python
obj = DatasetObject(
    name="bowl.n.01_1",
    category="bowl.n.01",
    position=[0.5, 0.5, 0.0],
    orientation=[0, 0, 0, 1],
)
```

### Taxonomy Integration

DatasetObjects load their abilities and states from the BDDL3 object taxonomy:

```python
from bddl.object_taxonomy import ObjectTaxonomy

taxonomy = ObjectTaxonomy()
synset = taxonomy.get_synset_from_category("bowl.n.01")
abilities = taxonomy.get_abilities(synset)
# abilities = {"fillable": {"max_volume": 0.002}, ...}
```

### Pre-defined States

DatasetObjects come with pre-computed initial states based on their category:

```python
obj.initial_states = {
    OnTop: {"target": None, "offset": [0, 0, 0]},  # Not on anything initially
    Filled: {},  # Not filled initially
    Cooked: {},  # Not cooked initially
}
```

## PrimitiveObject

**File:** `objects/primitive_object.py` (~18 KB)

**Purpose:** Simple geometric primitives for testing, debugging, and simple scenes.

### Types

- **Box** — Rectangular prism
- **Sphere** — Sphere
- **Cylinder** — Cylinder

### Creation

```python
box = PrimitiveObject(
    name="box_0",
    shape="box",
    size=[0.1, 0.1, 0.05],
    position=[0.0, 0.0, 0.025],
    color=[1.0, 0.0, 0.0, 1.0],
)
```

## LightObject

**File:** `objects/light_object.py` (~9 KB)

**Purpose:** Light emitters for scene illumination.

### Types

| Type | Description |
|------|-------------|
| `DomeLight` | Hemisphere dome light (environment lighting) |
| `PointLight` | Point light (omnidirectional) |
| `SpotLight` | Spot light (directional with cone) |
| `DirectionalLight` | Directional light (parallel rays) |

### Creation

```python
dome = LightObject(
    name="dome_0",
    light_type="dome",
    intensity=1000.0,
    color=[1.0, 1.0, 1.0],
)
```

## Registration

### Object Registration

Objects are registered via the `Registerable` metaclass:

```python
@Registerable.register()
class MyObject(USDObject):
    pass
```

### Dataset Object Registration

DatasetObjects are auto-discovered from dataset YAML files. The taxonomy system maps object categories to synsets and abilities.

## Testing

- `test_object_states.py` (~51 KB) — Tests dynamic states for all object types
- `test_object_removal.py` — Tests object lifecycle and garbage collection
- `test_examples.py` — Tests object creation in simulated environments

## See Also

<!-- openwiki: broken internal link [omnigibson/object_states.md] file "omnigibson/object_states.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/object_states.md`](omnigibson/object_states.md) — Dynamic state classes
<!-- openwiki: broken internal link [omnigibson/state_mechanics.md] file "omnigibson/state_mechanics.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/state_mechanics.md`](omnigibson/state_mechanics.md) — State update lifecycle
<!-- openwiki: broken internal link [omnigibson/objects.md] file "omnigibson/objects.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/objects.md`](omnigibson/objects.md) — USDObject base class
<!-- openwiki: broken internal link [omnigibson/registries.md] file "omnigibson/registries.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/registries.md`](omnigibson/registries.md) — Object registry
