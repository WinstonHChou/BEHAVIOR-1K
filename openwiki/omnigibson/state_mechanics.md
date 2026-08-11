---
type: wiki-component
title: OmniGibson State Mechanics
description: State update lifecycle — pre-step, post-step, reset phases, dependency-ordered initialization, and state mixin behavior.
tags: [omnigibson, state-mechanics, lifecycle, dependency-order]
---

# OmniGibson State Mechanics

**State mechanics** govern when and how object states are updated during the simulation lifecycle. States are updated at different phases (pre-step, post-step, reset) based on their mixin type and dependency order.

## State Update Phases

### Phase 1: Pre-Physics Step

```
sim._on_pre_physics_step()
  │
  └─► Update all state classes with UpdateStateMixin
       │
       └─► Iterates states in dependency order:
            1. Pose (no dependencies)
            2. OnTop (depends on Pose)
            3. Inside (depends on Pose, OnTop)
            4. ...
            5. Cooked (depends on Temperature, Heated)
```

**States updated:** Position-dependent states that determine spatial relationships before physics advances.

```python
class UpdateStateMixin:
    """States that must be updated before each physics step."""
    
    @classmethod
    def update_state(cls, obj, scene):
        """Update state value based on current simulation state."""
        pass
```

### Phase 2: Global Update

```
sim._on_pre_physics_step()
  │
  └─► Update all state classes with GlobalUpdateStateMixin
       │
       └─► Once per step (not per-object):
            - Temperature updates
            - Cooking progress
            - Particle interactions
```

**States updated:** States that depend on global simulation state or require cross-object computation.

```python
class GlobalUpdateStateMixin:
    """States that are updated globally once per step."""
    
    @classmethod
    def update_state(cls, scene):
        """Update state for all objects in scene."""
        pass
```

### Phase 3: Post-Physics Step

```
sim._on_post_physics_step()
  │
  └─► Update sensor states
       │
       └─► States dependent on sensor data:
            - IsGrasping (contact sensor data)
            - Touching (contact events)
            - ObjectsInFOVOfRobot (vision data)
```

**States updated:** States that require post-simulation data (contact, vision, etc.).

### Phase 4: Reset

```
env.reset()
  │
  └─► _reset_scene()
       │
       └─► Clear all object states
            │
            └─► Re-initialize states from initial configuration
```

**States reset:** All states are cleared and re-initialized from the object's initial state configuration.

## State Mixin Behavior

### UpdateStateMixin

```python
class UpdateStateMixin:
    """Base mixin for per-object state updates."""
    
    # Class attributes
    ability_dependencies: set = set()
    """Other state classes this state depends on."""
    
    @classmethod
    def update_state(cls, obj, scene):
        """Called once per simulation step for each object."""
        pass
```

### GlobalUpdateStateMixin

```python
class GlobalUpdateStateMixin:
    """Base mixin for global state updates."""
    
    @classmethod
    def update_state(cls, scene):
        """Called once per step for the entire scene."""
        pass
```

### ContactSubscribedStateMixin

```python
class ContactSubscribedStateMixin:
    """States that react to contact events."""
    
    contact_callback: callable = None
    
    def on_contact(self, obj1, obj2, contact_point):
        """Called when a contact event occurs between two objects."""
        pass
```

This mixin subscribes to PhysX contact events and updates state values when contacts are detected or broken.

### JointBreakSubscribedStateMixin

```python
class JointBreakSubscribedStateMixin:
    """States that react to joint break events."""
    
    def on_joint_break(self, obj, joint_idx):
        """Called when a joint breaks."""
        pass
```

## Dependency Ordering

The `ObjectStateFactory` determines the initialization order:

```python
def get_states_by_dependency_order():
    """Returns states sorted by dependency order using topological sort."""
    # Build dependency graph
    graph = {}
    for state_cls in REGISTERED_STATES.values():
        deps = state_cls.ability_dependencies
        graph[state_cls] = deps
    
    # Topological sort
    sorted_states = []
    visited = set()
    temp_visited = set()
    
    def visit(cls):
        if cls in temp_visited:
            raise ValueError(f"Circular dependency detected: {cls}")
        if cls not in visited:
            temp_visited.add(cls)
            for dep in graph[cls]:
                visit(dep)
            temp_visited.remove(cls)
            visited.add(cls)
            sorted_states.append(cls)
    
    for cls in graph:
        visit(cls)
    
    return sorted_states
```

### Example Dependency Graph

```
Pose (no deps)
  └─► OnTop (deps: Pose)
       └─► Inside (deps: Pose, OnTop)
            └─► Contains (deps: Inside)
  
Pose (no deps)
  └─► Temperature (deps: Pose)
       └─► Heated (deps: Temperature)
            └─► Cooked (deps: Temperature, Heated)
                 └─► Burnt (deps: Cooked)
```

## State Lifecycle

### 1. Object Creation

```python
obj = DatasetObject(name="bowl.n.01_1", category="bowl.n.01", position=[0.5, 0.5, 0.0])

# States are created from initial_states dict
for state_cls, args in obj.initial_states.items():
    factory.create_state(state_cls, obj, *args)
```

### 2. Object Addition to Scene

```python
with og.sim.adding_objects():
    og.sim.add_object(obj)

# Scene iterates over all object states
# and sets up subscriptions for contact/joint-break events
```

### 3. Simulation Step

```python
# Pre-step: Update position-dependent states
for state_cls in dependency_order:
    state_cls.update_state(obj, scene)

# Pre-step: Global updates
for state_cls in GlobalUpdateStateMixin.__subclasses__():
    state_cls.update_state(scene)

# Physics step
sim_context.step()

# Post-step: Update sensor-dependent states
for state_cls in ContactSubscribedStateMixin.__subclasses__():
    state_cls.update_state(obj, scene)
```

### 4. Object Removal

```python
obj.remove_state(OnTop)  # Remove specific state
obj.states.clear()       # Remove all states
```

## TensorizedValueState

Some states use tensorized values for batch processing:

```python
class TensorizedValueState:
    """State with tensorized values for batch processing."""
    
    # Values stored as tensors for GPU transfer
    value_tensor: torch.Tensor = None
    
    @property
    def value(self):
        return self.value_tensor.cpu().numpy()
```

This is used for states that need to be computed in parallel for many objects (e.g., `Temperature` for all objects).

## Testing

- `test_object_states.py` (~51 KB) — Tests all state update phases:
  - Pre-step state updates
  - Global state updates
  - Post-step sensor state updates
  - Reset state re-initialization
  - Dependency ordering verification
  - Contact-based state updates
  - Joint-break state updates
- `test_dump_load_states.py` — Tests state persistence across save/load

## See Also

<!-- openwiki: broken internal link [omnigibson/object_states.md] file "omnigibson/object_states.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/object_states.md`](omnigibson/object_states.md) — State class taxonomy
<!-- openwiki: broken internal link [omnigibson/state_mechanics.md] file "omnigibson/state_mechanics.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/state_mechanics.md`](omnigibson/state_mechanics.md) — State lifecycle
<!-- openwiki: broken internal link [omnigibson/simulator.md] file "omnigibson/simulator.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/simulator.md`](omnigibson/simulator.md) — Step lifecycle integration
