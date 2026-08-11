---
type: wiki-component
title: OmniGibson Systems
description: Particle systems — ParticleSystem, PhysicalParticleSystem, VisualParticleSystem, MacroPhysicalParticleSystem, Micro (fluid/granular/cloth) — for fluid, granular, and cloth simulation.
tags: [omnigibson, systems, particles, fluid, cloth, granular]
---

# OmniGibson Particle Systems

**Particle systems** provide physical simulation of fluids, granular materials, and cloth through GPU-accelerated particle dynamics. They integrate with object states to create dynamic substance interactions (filling, pouring, splashing, dripping).

## Class Hierarchy

```
BaseSystem (omnigibson/systems/system_base.py, 45 KB, ABC)
│
├── PhysicalParticleSystem (45 KB) — Base for physically simulated particles
│   │
│   ├── MacroPhysicalParticleSystem (70 KB) — Large-scale particle systems
│   │   ├── Fluid particle systems
│   │   ├── Granular particle systems
│   │   └── Cloth particle systems (macro-level)
│   │
│   └── MicroParticleSystem (71 KB) — Fine-grained particle behaviors
│       ├── Cloth folding/unfolding
│       ├── Fluid flow dynamics
│       └── Granular pile formation
│
└── VisualParticleSystem (subclassed from PhysicalParticleSystem)
    └── Visual representation layer
```

## BaseSystem

**File:** `systems/system_base.py` (~45 KB)

**Purpose:** Abstract base class for all particle systems. Defines the common interface for particle creation, update, and rendering.

```python
class BaseSystem(ABC):
    """Abstract base for particle systems."""
    
    def __init__(self, name, parent_obj):
        self.name = name
        self.parent_obj = parent_obj  # Parent object (e.g., cup, sink)
        self.particle_count = 0
        self.max_particles = 10000  # Default max
    
    @abstractmethod
    def step(self, scene): ...
    
    @abstractmethod
    def reset(self): ...
    
    @abstractmethod
    def add_particles(self, positions, velocities, properties): ...
    
    @abstractmethod
    def remove_particles(self, indices): ...
    
    @property
    @abstractmethod
    def particle_properties(self): ...
```

## PhysicalParticleSystem

**File:** `systems/physical_particle_system.py` (~45 KB)

**Purpose:** Base class for physically simulated particle systems. Handles particle physics, collision detection, and inter-particle interactions.

### Key Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `max_particles` | int | Maximum number of particles |
| `particle_count` | int | Current number of particles |
| `positions` | np.ndarray | Particle positions (N, 3) |
| `velocities` | np.ndarray | Particle velocities (N, 3) |
| `properties` | dict | Particle properties (mass, temperature, etc.) |
| `collisions` | bool | Enable collision detection |
| `particle_interactions` | bool | Enable inter-particle interactions |
| `external_forces` | dict | External force sources (gravity, wind, etc.) |

### Key Methods

| Method | Return Type | Description |
|--------|-------------|-------------|
| `step(scene)` | None | Update all particles (physics simulation) |
| `add_particles(positions, velocities, properties)` | list[int] | Add new particles, return indices |
| `remove_particles(indices)` | None | Remove particles by index |
| `reset()` | None | Reset particle state |
| `get_particle_positions()` | np.ndarray | Get current positions |
| `get_particle_velocities()` | np.ndarray | Get current velocities |
| `get_collision_contacts()` | list[(obj, idx)] | Get current collision contacts |

## VisualParticleSystem

**Purpose:** Adds visual representation to particle systems. Renders particles as visible objects (droplets, grains, fabric threads).

```python
class VisualParticleSystem(PhysicalParticleSystem):
    """Physical particle system with visual representation."""
    
    def step(self, scene):
        # Update physics
        super().step(scene)
        # Update visual representation
        self._update_visuals()
    
    def _update_visuals(self):
        """Update particle visual objects."""
        for i, pos in enumerate(self.positions):
            self.visual_objects[i].set_position(pos)
```

## MacroPhysicalParticleSystem

**File:** `systems/macro_particle_system.py` (~70 KB)

**Purpose:** Large-scale particle systems for fluid, granular, and cloth simulation. Manages thousands to tens of thousands of particles.

### Particle Types

| Type | Description | Max Particles | Use Case |
|------|-------------|---------------|----------|
| `Fluid` | Incompressible fluid with surface tension | ~50K | Water, coffee, milk |
| `Granular` | Granular material with friction | ~100K | Rice, sugar, sand |
| `Cloth` | Soft-body cloth simulation | ~20K | Towel, napkin, shirt |

### Fluid Configuration

```python
fluid_system = MacroPhysicalParticleSystem(
    name="water_fluid",
    parent_obj="sink",
    particle_type="fluid",
    max_particles=50000,
    particle_radius=0.002,
    fluid_density=1000.0,
    fluid_viscosity=0.001,
    surface_tension=0.072,
    gravity=[0, -9.81, 0],
)
```

### Granular Configuration

```python
granular_system = MacroPhysicalParticleSystem(
    name="rice_granular",
    parent_obj="container",
    particle_type="granular",
    max_particles=100000,
    particle_radius=0.001,
    particle_density=800.0,
    friction_coefficient=0.5,
    restitution_coefficient=0.2,
    gravity=[0, -9.81, 0],
)
```

### Cloth Configuration

```python
cloth_system = MacroPhysicalParticleSystem(
    name="towel_cloth",
    parent_obj="towel",
    particle_type="cloth",
    max_particles=20000,
    particle_spacing=0.01,
    bending_stiffness=0.5,
    stretching_stiffness=1.0,
    gravity=[0, -9.81, 0],
)
```

## MicroParticleSystem

**File:** `systems/micro_particle_system.py` (~71 KB)

**Purpose:** Fine-grained particle behaviors that operate at a smaller scale. Handles cloth folding/unfolding, fluid flow dynamics, and granular pile formation.

### Capabilities

- **Cloth folding** — Particles form fold patterns based on temperature state
- **Fluid flow** — Particles flow between connected containers
- **Granular flow** — Particles flow from container to container
- **Substance transfer** — Particles transfer between objects (dripping, pouring)

```python
class MicroParticleSystem(PhysicalParticleSystem):
    """Fine-grained particle behaviors."""
    
    def step(self, scene):
        # Handle cloth folding/unfolding based on state
        self._handle_cloth_states()
        # Handle fluid flow between containers
        self._handle_fluid_flow()
        # Handle granular flow
        self._handle_granular_flow()
        # Update particle physics
        super().step(scene)
```

## Integration with Object States

Particle systems integrate with object states:

| Object State | Particle System Interaction |
|-------------|---------------------------|
| `Filled` | Particles added/removed from system |
| `ContainedParticles` | System attached to container object |
| `ContactParticles` | System emits particles on contact |
| `ParticleApplier` | System adds particles to environment |
| `ParticleRemover` | System removes particles from environment |
| `ParticleSource` | System is a continuous particle source |
| `ParticleSink` | System attracts nearby particles |

### Example: Filled State + Particle System

```python
# When an object has the "Filled" state, a particle system manages the substance
obj.add_state(Filled, "water", 0.5)  # 50% filled

# The particle system creates water particles
particle_system = MacroPhysicalParticleSystem(
    name="water",
    parent_obj=obj,
    particle_type="fluid",
    max_particles=int(0.5 * MAX_PARTICLES),
)

# When state changes, particle system updates
obj.set_state(Filled, "water", 0.8)  # 80% filled
particle_system.resize(0.8 * MAX_PARTICLES)
```

## System Lifecycle

### 1. Creation

```python
system = MacroPhysicalParticleSystem(
    name="water",
    parent_obj=container_obj,
    particle_type="fluid",
    max_particles=50000,
)
```

### 2. Addition to Scene

```python
scene.add_system(system)
```

### 3. Stepping

```python
# Called in simulator._on_pre_physics_step()
for system in scene.particle_systems:
    system.step(scene)
```

### 4. Particle Operations

```python
# Add particles (e.g., when pouring)
indices = system.add_particles(
    positions=new_positions,
    velocities=new_velocities,
    properties={"temperature": 373.0},
)

# Remove particles (e.g., when draining)
system.remove_particles(indices_to_remove)
```

### 5. Reset

```python
system.reset()  # Clear all particles
```

## Performance Considerations

| Factor | Impact | Mitigation |
|--------|--------|------------|
| Particle count | Linear in physics update | Cap max_particles |
| Collision detection | O(N²) naive | Spatial hashing (grid-based) |
| Inter-particle forces | O(N²) naive | Domain decomposition |
| Visual rendering | GPU bandwidth | Batch rendering |
| Memory usage | 3× particle count × sizeof(float) | Use float16 where possible |

## Testing

- `test_systems.py` (~2 KB) — Tests particle system creation and stepping
- Integration tests in `test_object_states.py` test particle-applier/remover states

## See Also

<!-- openwiki: broken internal link [omnigibson/object_states.md] file "omnigibson/object_states.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/object_states.md`](omnigibson/object_states.md) — Particle-related state classes
<!-- openwiki: broken internal link [omnigibson/transition_rules.md] file "omnigibson/transition_rules.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/transition_rules.md`](omnigibson/transition_rules.md) — Transition rules that trigger particle systems
<!-- openwiki: broken internal link [omnigibson/systems.md] file "omnigibson/systems.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/systems.md`](omnigibson/systems.md) — Systems overview
