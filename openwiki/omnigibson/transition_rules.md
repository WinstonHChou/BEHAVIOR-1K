---
type: wiki-component
title: OmniGibson Transition Rules
description: TransitionRuleAPI class, REGISTERED_RULES, ObjectAttrs/TransitionResults, recipe execution (CookingRecipe, MixingRecipe, MachineRecipe, SubstanceCookingRecipe, WasherRecipe), particle system interactions.
tags: [omnigibson, transition-rules, state-machine, cooking, washing, slicing]
---

# OmniGibson Transition Rules

**Transition rules** (defined in `transition_rules.py`, ~111 KB — one of the largest files in the codebase) are the runtime engine that implements discrete state transitions between object states. They handle cooking, washing, mixing, machine operations, slicing, and other transformations.

## Entry Point

```python
from omnigibson.transition_rules import (
    TransitionRuleAPI,
    REGISTERED_RULES,
    ObjectAttrs,
    TransitionResults,
)
```

## Class Hierarchy

```
BaseTransitionRule (abstract, defined in transition_rules.py)
│
├── CookingRule (subclass of BaseTransitionRule)
├── WashingRule
├── MixingRule
├── MachineRule
├── SubstanceCookingRule
├── WasherRule (particleremover/nonparticleremover variants)
├── DicingRule
├── MeltingRule
├── SlicingRule
└── AttachmentRule
```

## TransitionRuleAPI

**Purpose:** Main API that scans all registered rules each simulation step, finds matching object candidates, evaluates conditions, and returns transition results (object creation/removal).

```python
class TransitionRuleAPI:
    def __init__(self, scene, kb):
        self.scene = scene
        self.kb = kb  # BDDL knowledge base for recipe lookup
        self.rules = list(REGISTERED_RULES.values())
    
    def step(self):
        """Execute all registered rules once."""
        for rule_class in self.rules:
            if rule_class.is_disabled():
                continue
            # Find matching object candidates
            candidates = self._find_candidates(rule_class)
            # Evaluate rule conditions
            for candidate in candidates:
                if rule_class.evaluate(candidate, self.scene):
                    # Execute rule → returns TransitionResults
                    results = rule_class.execute(candidate, self.scene)
                    self._apply_transition(results)
    
    def _find_candidates(self, rule_class):
        """Find objects that could be involved in this rule."""
        # Uses ObjectAttrs to specify candidate objects
        ...
    
    def _apply_transition(self, results):
        """Apply object creation/removal from transition."""
        for attrs in results.remove:
            self.scene.remove_object(attrs.name)
        for attrs in results.add:
            obj = self.scene.create_object(attrs)
            for state_cls, state_args in attrs.states.items():
                obj.add_state(state_cls, *state_args)
```

## Key Data Structures

### ObjectAttrs

```python
@dataclass
class ObjectAttrs:
    """Attributes for creating or identifying an object."""
    category: str  # Object category (e.g., "apple")
    model: str  # Model name (e.g., "apple.n.01")
    name: str  # Object instance name (e.g., "apple.n.01_1")
    scale: float  # Object scale
    obj: USDObject  # Existing object (for removal)
    pos: np.ndarray  # Position (for creation)
    orn: np.ndarray  # Orientation (for creation)
    bb_pos: np.ndarray  # Bounding box position
    bb_orn: np.ndarray  # Bounding box orientation
    states: dict  # {state_cls: state_args}
    callback: callable  # Post-creation callback
```

### TransitionResults

```python
@dataclass
class TransitionResults:
    """Result of a rule execution."""
    add: list[ObjectAttrs]  # Objects to create
    remove: list[ObjectAttrs]  # Objects to remove
```

## Registered Rules

```python
REGISTERED_RULES = {
    "cooking": CookingRule,
    "washing": WashingRule,
    "mixing": MixingRule,
    "machine": MachineRule,
    "substance_cooking": SubstanceCookingRule,
    "washer_particleremover": WasherRule,
    "washer_nonparticleremover": WasherRule,
    "dicing": DicingRule,
    "melting": MeltingRule,
    "slicing": SlicingRule,
    "attachment": AttachmentRule,
    # ... more rules ...
}
```

## Rule Types

### CookingRule

**Purpose:** Implements cooking transitions (raw → cooked, bagel → toasted).

**Conditions:**
- Object has `cooked` predicate in BDDL
- Heat source (stove, microwave, oven) is active and nearby
- Container (pot, pan) is on heat source

**Execution:**
```python
class CookingRule(BaseTransitionRule):
    def evaluate(self, candidate, scene):
        # Check if object is cookable and heat source is active
        if not self._is_cookable(candidate):
            return False
        if not self._has_active_heat_source(candidate, scene):
            return False
        return True
    
    def execute(self, candidate, scene):
        # Create cooked version of object
        new_attrs = ObjectAttrs(
            category=candidate.category + "_cooked",
            model=f"{candidate.model}_cooked",
            states={Cooked: {}, Heated: {100.0}},
        )
        return TransitionResults(add=[new_attrs], remove=[candidate])
```

### WashingRule

**Purpose:** Implements washing transitions (dirty → clean).

**Conditions:**
- Object has `stained` state
- Cleaning tool (sponge, cloth) is present
- Water source (sink, faucet) is active

### MixingRule

**Purpose:** Implements mixing operations.

**Conditions:**
- Multiple liquid containers are nearby
- Mixing tool (spoon, blender) is present

### MachineRule

**Purpose:** Implements machine-based operations (washing machine, dryer, microwave).

**Conditions:**
- Object is inside a machine
- Machine is toggled on (`ToggledOn` state)
- Machine has appropriate cycle active

### SubstanceCookingRule

**Purpose:** Simple substance-to-substance cooking (e.g., water → steam).

**Conditions:**
- Substance is at boiling point
- Heat source is active

### WasherRule

**Purpose:** Washer-specific transitions with particle removal.

**Subtypes:**
- `WasherRule` (particleremover) — Washer removes particles
- `WasherRule` (nonparticleremover) — Washer without particle effects

### DicingRule

**Purpose:** Dicing/cutting operations (apple → diced_apple).

**Conditions:**
- Slicing tool is active (`SlicerActive`)
- Object is sliceable (`SliceableRequirement`)

### MeltingRule

**Purpose:** Melting transitions (ice → water).

**Conditions:**
- Object temperature exceeds melting point
- Heat source is active

```python
class MeltingRule(BaseTransitionRule):
    def evaluate(self, candidate, scene):
        temp_state = candidate.get_state(Temperature)
        if temp_state and temp_state.value >= m.MELTING_TEMPERATURE:
            return True
        return False
```

### SlicingRule

**Purpose:** Slicing operations with knife tool.

**Conditions:**
- Object has `slicable` capability
- Slicing tool is in contact with object
- Tool is moving across object

## Particle System Interactions

Transition rules can trigger particle system operations:

```python
class WashingRule(BaseTransitionRule):
    def execute(self, candidate, scene):
        results = TransitionResults(
            add=[new_attrs],
            remove=[candidate],
        )
        # Also trigger particle system to remove water particles
        if scene.particle_systems:
            for system in scene.particle_systems:
                if system.parent_obj is candidate:
                    system.remove_all_particles()
        return results
```

## Disable Rules

Rules can be disabled globally via macros:

```python
# In omniGibson/macros or omniGibson/transition_rules.py
DISABLED_TRANSITION_RULES = {
    "cooking",
    "washing",
    "mixing",
    # ... rules to disable
}

class BaseTransitionRule:
    @classmethod
    def is_disabled(cls):
        return cls.name in DISABLED_TRANSITION_RULES
```

This is used by JoyLo and eval frameworks to disable certain transitions during teleoperation/evaluation.

## Integration with Scene

Transition rules are invoked from `scene_base.py`:

```python
class Scene:
    def __init__(self):
        self.transition_rule_api = TransitionRuleAPI(self, kb)
    
    def step(self):
        # ... object state updates ...
        # Execute transition rules
        self.transition_rule_api.step()
        # ... physics step ...
```

## BDDL Integration

Transition rules are derived from BDDL transition map JSONs:

```python
class TransitionRuleAPI:
    def __init__(self, scene, kb):
        # Load recipes from KB
        self.recipes = {}
        for recipe_cls in kb.transition_rules:
            self.recipes[recipe_cls.name] = recipe_cls
    
    def translate_bddl_recipe_to_og_recipe(self, recipe_name):
        """Translate BDDL recipe to OmniGibson recipe."""
        bddl_recipe = self.recipes[recipe_name]
        og_recipe = Recipe(
            name=bdll_recipe.name,
            input_states=bdll_recipe.input_states,
            output_states=bdll_recipe.output_states,
            ...
        )
        return og_recipe
```

## Testing

- `test_transition_rules.py` (~59 KB) — Comprehensive tests:
  - Rule candidate finding
  - Rule condition evaluation
  - Rule execution and object creation/removal
  - Particle system triggers
  - Disabled rules handling
  - Temperature state integration
  - Slicing/cooking/washing/mixing rules

## See Also

<!-- openwiki: broken internal link [../cross-system/transition-rules-execution.md] file "../cross-system/transition-rules-execution.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`cross-system/transition-rules-execution.md`](../cross-system/transition-rules-execution.md) — Cross-system transition rules
<!-- openwiki: broken internal link [omnigibson/object_states.md] file "omnigibson/object_states.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/object_states.md`](omnigibson/object_states.md) — State classes modified by rules
<!-- openwiki: broken internal link [omnigibson/systems.md] file "omnigibson/systems.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/systems.md`](omnigibson/systems.md) — Particle system interactions
<!-- openwiki: broken internal link [../bddl3/transition_rules.md] file "../bddl3/transition_rules.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`bddl3/transition_rules.md`](../bddl3/transition_rules.md) — BDDL recipe definitions
