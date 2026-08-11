---
type: wiki-component
title: OmniGibson Object States
description: 40+ dynamic state classes (OnTop, Filled, Cooked, Open, etc.), ObjectStateFactory dependency-ordered initialization, ability_dependencies, and state taxonomy.
tags: [omnigibson, object-states, dynamics, state-machine]
---

# OmniGibson Object States

**Object states** are dynamic, time-varying properties of simulated objects that change based on physical interactions and transition rules. OmniGibson provides 40+ state classes covering pose, temperature, fill level, cloth configuration, attachment, and more.

## State Registry

```python
from omnigibson.object_states import REGISTERED_STATES, ObjectStateFactory
```

## State Taxonomy

States are organized by category:

### Pose & Position States

| State | Arity | Description |
|-------|-------|-------------|
| `Pose` | - | Object position/orientation |
| `AABB` | - | Axis-aligned bounding box |
| `OnTop` | Binary (obj, target) | Object is on top of target |
| `Under` | Binary (obj, target) | Object is under target |
| `Inside` | Binary (obj, container) | Object is inside container |
| `NextTo` | Binary (obj, target) | Object is next to target |
| `Touching` | Binary (obj, target) | Object is touching target |
| `AttachedTo` | Binary (obj, target) | Object is attached to target |
| `AttachedToConstraint` | Binary (obj, target) | Attached via physx constraint |
| `HorizontalAdjacency` | Binary (obj, target) | Horizontal adjacency relationship |
| `VerticalAdjacency` | Binary (obj, target) | Vertical adjacency relationship |

### Temperature & Cooking States

| State | Arity | Description |
|-------|-------|-------------|
| `Temperature` | Unary (obj, value) | Object temperature in Kelvin |
| `Heated` | Unary (obj, temp) | Object is heated (above threshold) |
| `Cooked` | Unary (obj) | Object is cooked |
| `Burnt` | Unary (obj) | Object is burnt |
| `Frozen` | Unary (obj) | Object is frozen |
| `Saturated` | Unary (obj) | Object is saturated (liquid absorbed) |
| `MaxTemperature` | Unary (obj) | Object has reached max temperature |
| `HeatSourceOrSink` | Unary (obj) | Object is a heat source or sink |

### Fill & Containment States

| State | Arity | Description |
|-------|-------|-------------|
| `Filled` | Unary (obj, substance, amount) | Container is filled with substance |
| `Contains` | Binary (container, obj) | Container contains object |
| `ContainedParticles` | Unary (obj) | Object contains particles |
| `ContactParticles` | Unary (obj) | Object is contacted by particles |

### Cloth & Configuration States

| State | Arity | Description |
|-------|-------|-------------|
| `Draped` | Binary (obj, target) | Cloth is draped over target |
| `Folded` | Unary (obj) | Cloth is folded |
| `Unfolded` | Unary (obj) | Cloth is unfolded |

### Open/Switch States

| State | Arity | Description |
|-------|-------|-------------|
| `Open` | Unary (obj, position) | Object is open (drawer, cabinet) |
| `ToggledOn` | Unary (obj) | Machine is toggled on |
| `Covered` | Unary (obj) | Object is covered (by lid, etc.) |
| `Overlaid` | Binary (obj, overlay) | Object is overlaid |
| `OnFire` | Unary (obj) | Object is on fire |
| `SlicerActive` | Unary (obj) | Slicer is active |
| `SliceableRequirement` | Unary (obj) | Object requires slicing |

### Robot States

| State | Arity | Description |
|-------|-------|-------------|
| `IsGrasping` | Unary (robot, obj) | Robot is grasping object |
| `ObjectsInFOVOfRobot` | Unary (robot, obj) | Object is in robot's field of view |

### Particle System States

| State | Arity | Description |
|-------|-------|-------------|
| `ParticleApplier` | Unary (obj) | Object applies particles |
| `ParticleRemover` | Unary (obj) | Object removes particles |
| `ParticleSource` | Unary (obj) | Object is a particle source |
| `ParticleSink` | Unary (obj) | Object is a particle sink |

## State Mixins

State classes use mixins for common behavior:

| Mixin | Description |
|-------|-------------|
| `UpdateStateMixin` | Base mixin for states updated per step |
| `GlobalUpdateStateMixin` | States updated globally (once per step) |
| `KinematicsMixin` | States based on kinematics (position, velocity) |
| `LinkBasedStateMixin` | States based on link indices |
| `ContactSubscribedStateMixin` | States that react to contact events |
| `JointBreakSubscribedStateMixin` | States that react to joint breaks |
| `TensorizedValueState` | States with tensorized values |

## ObjectStateFactory

**File:** `object_states/factory.py` (~6 KB)

The factory manages dependency-ordered initialization of all object states:

```python
class ObjectStateFactory:
    """Factory for creating and managing object states in dependency order."""
    
    @staticmethod
    def get_states_by_dependency_order():
        """Returns states sorted by dependency order."""
        # States with no dependencies come first
        # States that depend on other states come after
        return sorted_states
    
    @staticmethod
    def create_state(state_cls, obj, *args, **kwargs):
        """Create a state instance and attach it to the object."""
        state = state_cls(obj, *args, **kwargs)
        obj.states[state_cls] = state
        return state
```

### Dependency Ordering

States are ordered by their `ability_dependencies`:

```python
# If state A depends on state B (A.ability_dependencies = {B}),
# then B must be initialized before A
```

Example ordering:
1. `Pose` (no dependencies)
2. `OnTop` (depends on Pose)
3. `Inside` (depends on Pose, OnTop)
4. `Temperature` (depends on Pose)
5. `Heated` (depends on Temperature)
6. `Cooked` (depends on Temperature, Heated)

### Ability Dependencies

Each state declares its `ability_dependencies`:

```python
class Cooked(UpdateStateMixin, ObjectState):
    ability_dependencies = {Temperature, Heated}
    """Cooked requires Temperature and Heated states."""
```

This ensures states are created in the correct order and that prerequisite states exist.

## Creating States on Objects

```python
from omnigibson.object_states import OnTop, Filled, Cooked

# Add a state to an object
obj.add_state(OnTop, target="table.n.02_1", offset=[0, 0, 0])

# Check if an object has a state
if obj.has_state(Cooked):
    temp = obj.get_state(Cooked)

# Set a state value
obj.set_state(Filled, "water", 0.5)

# Remove a state
obj.remove_state(OnTop)
```

## State Update Lifecycle

States are updated at different phases:

| Phase | Update Method | Examples |
|-------|--------------|----------|
| Pre-step (before physics) | `UpdateStateMixin.update_state()` | `OnTop`, `Inside` |
| Global (once per step) | `GlobalUpdateStateMixin.update_state()` | `Temperature`, `Cooked` |
| Post-step (after physics) | Sensor-dependent states | `IsGrasping`, `Touching` |
| Contact event | `ContactSubscribedStateMixin.on_contact()` | `Touching`, `AttachedTo` |
| Joint break event | `JointBreakSubscribedStateMixin.on_joint_break()` | State invalidation |

## State Evaluation in BDDL

Object states map directly to BDDL predicates:

| BDDL Predicate | OmniGibson State |
|---------------|-----------------|
| `(ontop ?obj ?target)` | `OnTop(obj, target)` |
| `(inside ?obj ?container)` | `Inside(obj, container)` |
| `(cooked ?obj)` | `Cooked(obj)` |
| `(filled ?obj ?substance)` | `Filled(obj, substance)` |
| `(open ?obj)` | `Open(obj)` |
| `(toggledon ?obj)` | `ToggledOn(obj)` |

<!-- openwiki: broken internal link [../cross-system/omnigibson-bddl-integration.md] file "../cross-system/omnigibson-bddl-integration.md" does not exist. Fix the href or restore the target, then delete this comment. -->
See [`cross-system/omnigibson-bddl-integration.md`](../cross-system/omnigibson-bddl-integration.md) for details.

## Testing

- `test_object_states.py` (~51 KB) — Comprehensive tests for all state classes
  - State creation and initialization
  - State update at each phase
  - State dependency ordering
  - State removal and invalidation
  - Contact-based state updates
  - Temperature state transitions

## See Also

<!-- openwiki: broken internal link [omnigibson/state_mechanics.md] file "omnigibson/state_mechanics.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/state_mechanics.md`](omnigibson/state_mechanics.md) — State update lifecycle
<!-- openwiki: broken internal link [omnigibson/objects.md] file "omnigibson/objects.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/objects.md`](omnigibson/objects.md) — Object class
<!-- openwiki: broken internal link [omnigibson/transition_rules.md] file "omnigibson/transition_rules.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/transition_rules.md`](omnigibson/transition_rules.md) — Transition rules that modify states
