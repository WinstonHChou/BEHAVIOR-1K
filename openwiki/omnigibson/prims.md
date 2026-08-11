---
type: wiki-component
title: OmniGibson USD Primitives
description: USD primitive layer — BasePrim → XFormPrim → GeomPrim → RigidPrim → EntityPrim → ClothPrim → JointPrim → MaterialPrim, the low-level abstraction over Isaac Sim's USD stage.
tags: [omnigibson, prims, usd, physics, articulation]
---

# OmniGibson USD Primitives

**Primitives** (prims) provide the low-level abstraction over Isaac Sim's USD stage. They are the foundational building blocks for all physical objects in simulation — from simple geometric shapes to articulated robots and soft-body cloth.

## Class Hierarchy

```
BasePrim (omnigibson/prims/base_prim.py, 11 KB, ABC)
│
├── XFormPrim (22 KB) — Transformable USD Xform
│   │
│   ├── GeomPrim (13 KB) — Geometric prim with collision shape
│   │   ├── RigidPrim (27 KB) — Static/dynamic rigid body
│   │   │   ├── RigidDynamicPrim (15 KB) — Dynamically simulated rigid body
│   │   │   └── RigidKinematicPrim (7 KB) — Kinematic (controlled) rigid body
│   │   └── EntityPrim (68 KB) — Articulated entity (composite of prims + joints)
│   │
│   ├── ClothPrim (43 KB) — Soft-body cloth simulation prim
│   │
│   ├── JointPrim (36 KB) — PhysX joint abstraction
│   │
│   └── MaterialPrim (29 KB) — USD material assignment
```

## BasePrim

**File:** `prims/base_prim.py` (~11 KB)

**Purpose:** Abstract base class for all USD primitives. Provides common USD path and attribute access.

```python
class BasePrim(ABC):
    """Abstract base for all USD primitives."""
    
    def __init__(self, prim_path, name):
        self.prim_path = prim_path  # USD path (e.g., "/scene/object_0")
        self.name = name
        self.prim = stage.GetPrimAtPath(prim_path)  # USD Prim object
```

## XFormPrim

**File:** `prims/xform_prim.py` (~22 KB)

**Purpose:** Transformable USD Xform prim. Provides position, orientation, and scale operations.

```python
class XFormPrim(BasePrim):
    """Transformable USD Xform prim."""
    
    def set_position(self, pos, world=True): ...
    def set_orientation(self, orn, world=True): ...
    def set_scale(self, scale, world=True): ...
    def get_position(self, world=True): ...
    def get_orientation(self, world=True): ...
    def get_scale(self, world=True): ...
```

## GeomPrim

**File:** `prims/geom_prim.py` (~13 KB)

**Purpose:** Geometric prim with collision shape. Adds collision geometry to an XForm.

```python
class GeomPrim(XFormPrim):
    """Geometric prim with collision shape."""
    
    def __init__(self, prim_path, name, collision=True, visual=True):
        super().__init__(prim_path, name)
        self.collision = collision  # Enable collision shape
        self.visual = visual  # Enable visual shape
```

## RigidPrim

**File:** `prims/rigid_prim.py` (~27 KB)

**Purpose:** Rigid body prim with PhysX physics properties.

```python
class RigidPrim(GeomPrim):
    """Rigid body prim with PhysX physics."""
    
    def set_mass(self, mass): ...
    def set_linear_damping(self, damping): ...
    def set_angular_damping(self, damping): ...
    def set_static_friction(self, friction): ...
    def set_dynamic_friction(self, friction): ...
    def set_restitution(self, restitution): ...
```

### RigidDynamicPrim

**File:** `prims/rigid_dynamic_prim.py` (~15 KB)

**Purpose:** Dynamically simulated rigid body. Responds to gravity and forces.

```python
class RigidDynamicPrim(RigidPrim):
    """Dynamically simulated rigid body."""
    
    def apply_force(self, force, point=None): ...
    def apply_torque(self, torque): ...
    def set_velocity(self, lin_vel, ang_vel): ...
```

### RigidKinematicPrim

**File:** `prims/rigid_kinematic_prim.py` (~7 KB)

**Purpose:** Kinematic (controlled) rigid body. Moves via direct position/orientation setting, not physics.

```python
class RigidKinematicPrim(RigidPrim):
    """Kinematic (controlled) rigid body."""
    
    def set_position_target(self, pos): ...  # Move to target via physics
    def set_position_absolute(self, pos): ...  # Set position directly
```

## EntityPrim

**File:** `prims/entity_prim.py` (~68 KB)

**Purpose:** Articulated entity — a composite of multiple prims connected by joints. This is the primary class for robots (arms, bases) and articulated objects (drawers, cabinets).

```python
class EntityPrim(GeomPrim):
    """Articulated entity with joints."""
    
    def add_link(self, link_name, link_path): ...  # Add a rigid link
    def add_joint(self, joint_name, joint_path, parent, child): ...  # Add joint
    def set_joint_position(self, joint_idx, position): ...
    def set_joint_velocity(self, joint_idx, velocity): ...
    def set_joint_effort(self, joint_idx, effort): ...
    def get_joint_positions(self): ...  # Return all joint positions
    def get_joint_velocities(self): ...  # Return all joint velocities
    def get_link_indices(self): ...  # Return PhysX link indices
    def get_rigid_body_indices(self): ...  # Return PhysX rigid body indices
```

### EntityPrim Anatomy

```
EntityPrim (e.g., a robot arm)
├── Link 0 (base) — RigidDynamicPrim
├── Joint 0 (revolute) — JointPrim
│   └── Link 1 (upper arm) — RigidDynamicPrim
│       ├── Joint 1 (revolute) — JointPrim
│       │   └── Link 2 (forearm) — RigidDynamicPrim
│       └── ...
└── Collision shapes on each link
```

## ClothPrim

**File:** `prims/cloth_prim.py` (~43 KB)

**Purpose:** Soft-body cloth simulation prim using PhysX cloth simulation.

```python
class ClothPrim(GeomPrim):
    """Soft-body cloth simulation prim."""
    
    def set_thickness(self, thickness): ...
    def set_mass(self, mass): ...
    def set_bending_stiffness(self, stiffness): ...
    def set_shrinkage_stiffness(self, stiffness): ...
    def set_pressure_factor(self, factor): ...
    def set_velocity_damping(self, damping): ...
```

## JointPrim

**File:** `prims/joint_prim.py` (~36 KB)

**Purpose:** PhysX joint abstraction. Supports revolute, prismatic, fixed, and spherical joints.

```python
class JointPrim(BasePrim):
    """PhysX joint abstraction."""
    
    def set_type(self, joint_type): ...  # "revolute", "prismatic", "fixed", "spherical"
    def set_limits(self, lower, upper): ...  # Joint position limits
    def set_drive(self, target_velocity, max_force): ...  # Joint drive parameters
    def set_motor_torque(self, torque): ...  # Joint motor torque
    def set_position(self, position): ...  # Joint position
    def set_velocity(self, velocity): ...  # Joint velocity
    def set_effort(self, effort): ...  # Joint effort/torque
```

## MaterialPrim

**File:** `prims/material_prim.py` (~29 KB)

**Purpose:** USD material assignment for visual appearance.

```python
class MaterialPrim(BasePrim):
    """USD material assignment."""
    
    def set_color(self, color): ...  # Diffuse color (R, G, B, A)
    def set_metallic(self, value): ...
    def set_roughness(self, value): ...
    def set_emissive(self, color): ...  # Emissive color
    def assign_material(self, prim_path): ...  # Assign material to prim
```

## Primitive Creation Flow

Primitives are created by the USDObject class:

```python
# USDObject creates primitives internally
class USDObject:
    def _create_prims(self):
        # Create rigid body prim
        self.rigid_body = RigidDynamicPrim(
            prim_path=f"/World/{self.name}",
            name=self.name,
            collision=True,
            visual=True,
        )
        # Set material
        self.material = MaterialPrim(
            prim_path=f"/World/{self.name}/material",
            name="material",
        )
        self.material.set_color(self.color)
```

For articulated entities:

```python
# EntityPrim creates link + joint hierarchies
entity = EntityPrim(
    prim_path="/World/robot",
    name="robot",
    collision=True,
    visual=True,
)
entity.add_link("base", "/World/robot/base")
entity.add_joint("joint_0", "/World/robot/joint_0", "base", "link_1")
entity.add_link("link_1", "/World/robot/link_1")
```

## PhysX Integration

Prims integrate with PhysX through Isaac Sim's PhysX APIs:

```python
# Access PhysX rigid body
physx_rigid_body = self.rigid_body_physx_rigid_body

# Read/update state
self.pos_w, self.orn_w = physx_rigid_body.get_world_pose()
physx_rigid_body.set_world_pose(pos, orn)
```

## Testing

- `test_snapshots.py` (~7 KB) — Tests primitive rendering output
- `test_object_removal.py` — Tests primitive lifecycle
- `benchmark/benchmark_object_count.py` — Primitive load scalability

## See Also

<!-- openwiki: broken internal link [omnigibson/objects.md] file "omnigibson/objects.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/objects.md`](omnigibson/objects.md) — USDObject (uses prims internally)
<!-- openwiki: broken internal link [omnigibson/robots.md] file "omnigibson/robots.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/robots.md`](omnigibson/robots.md) — Robot (uses EntityPrim internally)
<!-- openwiki: broken internal link [omnigibson/simulator.md] file "omnigibson/simulator.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/simulator.md`](omnigibson/simulator.md) — Simulator USD management
