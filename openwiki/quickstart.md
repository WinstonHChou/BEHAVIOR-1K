---
type: wiki-quickstart
title: BEHAVIOR-1K Wiki Quickstart
description: High-level guide to navigating the BEHAVIOR-1K repository wiki, with maps, concepts, and task routing.
tags: [overview, navigation]
---

# BEHAVIOR-1K Wiki Quickstart

Welcome to the BEHAVIOR-1K source-grounded wiki. This guide helps you navigate the repository and find the documentation you need.

## Repository Overview

**BEHAVIOR-1K** is a simulation benchmark for testing embodied AI agents on 1,000+ household activities. It is a monorepo with these major components:

| Component | Path | Purpose |
|-----------|------|---------|
| **OmniGibson** | `/OmniGibson/` | Physics simulation engine on NVIDIA Isaac Sim |
| **BDDL3** | `/bddl3/` | Behavior Domain Definition Language — 1,000+ activity definitions |
| **JoyLo** | `/joylo/` | Bimanual teleoperation framework (JoyCon → R1/R1Pro robots) |
| **Asset Pipeline** | `/asset_pipeline/` | 3D asset conversion (3ds Max → USD) |
| **Knowledgebase** | `/knowledgebase/` | Static site generator for browsing BDDL entities |
| **Eval Job Queue** | `/eval-jobqueue/` | Distributed evaluation infrastructure |
| **Docker** | `/docker/` | Container images (behavior, behavior-dev, behavior-gha) |

## Navigation Guide

### I want to understand the simulation engine
→ [`omnigibson/overview.md`](omnigibson/overview.md) — Architecture, registry pattern, config-driven design

### I need to create a custom environment
→ [`omnigibson/environments.md`](omnigibson/environments.md) — Environment creation from YAML configs

### I want to control a robot
→ [`omnigibson/robots.md`](omnigibson/robots.md) — Robot class, controllers, proprioception

### I'm working with BDDL activity definitions
<!-- openwiki: broken internal link [bddl3/overview.md] file "bddl3/overview.md" does not exist. Fix the href or restore the target, then delete this comment. -->
→ [`bddl3/overview.md`](bddl3/overview.md) — BDDL structure, 1,018 activities

### I'm setting up teleoperation
<!-- openwiki: broken internal link [joylo/overview.md] file "joylo/overview.md" does not exist. Fix the href or restore the target, then delete this comment. -->
→ [`joylo/overview.md`](joylo/overview.md) — Bimanual teleop architecture, ZMQ IPC

### I need to generate 3D assets
<!-- openwiki: broken internal link [asset-pipeline/overview.md] file "asset-pipeline/overview.md" does not exist. Fix the href or restore the target, then delete this comment. -->
→ [`asset-pipeline/overview.md`](asset-pipeline/overview.md) — DVC pipeline, 3ds Max → USD

### I'm running evaluation for the challenge
<!-- openwiki: broken internal link [eval-jobqueue/overview.md] file "eval-jobqueue/overview.md" does not exist. Fix the href or restore the target, then delete this comment. -->
→ [`eval-jobqueue/overview.md`](eval-jobqueue/overview.md) — Job queue, SLURM workers

### I need to understand evaluation results
<!-- openwiki: broken internal link [omnigibson/eval/overview.md] file "omnigibson/eval/overview.md" does not exist. Fix the href or restore the target, then delete this comment. -->
→ [`omnigibson/eval/overview.md`](omnigibson/eval/overview.md) — Challenge evaluation pipeline

## Task Routing Table

| Change Area | Relevant Page | Source Files | Tests |
|-------------|---------------|-------------|-------|
| Sim physics | `omnigibson/simulator.md` | `simulator.py` | `test_envs.py` |
| Config system | `omnigibson/macros.md` | `macros.py` | — |
| Robot control | `omnigibson/robots.md` | `robots/robot.py` | `test_controllers.py` |
| Controllers | `omnigibson/controllers.md` | `controllers/*.py` | `test_controllers.py` |
| Object states | `omnigibson/object_states.md` | `object_states/*.py` | `test_object_states.py` |
| Tasks | `omnigibson/tasks.md` | `tasks/*.py` | `test_behavior_task.py` |
| Sensors | `omnigibson/sensors.md` | `sensors/*.py` | `test_sensors.py` |
| Primitives | `omnigibson/prims.md` | `prims/*.py` | — |
| Transition rules | `omnigibson/transition_rules.md` | `transition_rules.py` | `test_transition_rules.py` |
| Particle systems | `omnigibson/systems.md` | `systems/*.py` | `test_systems.py` |
| BDDL parsing | `bddl3/parsing.md` | `bddl/parsing.py` | `tests/bddl_tests.py` |
| Knowledge base | `bddl3/knowledgebase.md` | `bddl/knowledge_base/*.py` | `tests/test_knowledgebase.py` |
| Teleop client | `joylo/agents.md` | `gello/agents/*.py` | — |
| Teleop server | `joylo/robots.md` | `gello/robots/og_robot.py` | — |
| Asset pipeline | `asset-pipeline/pipeline.md` | `b1k_pipeline/*.py` | — |
| Evaluation | `omnigibson/eval_cli.md` | `omnigibson/eval/eval.py` | — |
| Docker | `docker/overview.md` | `docker/Dockerfile` | — |
| CI/CD | `ci-cd/workflows.md` | `.github/workflows/*.yml` | — |

## Key Concepts

### Registry Pattern
OmniGibson uses a registry pattern for all extensible components (objects, robots, controllers, scenes, tasks, sensors). New components are registered via `@REGISTERED_XYZ.register()` decorators.

### Config-Driven Design
Every component is fully configurable via YAML files. Configs are merged in order: default → robot YAML → user config.

### Simulation Lifecycle
```
og.launch() → og.Environment(config) → env.reset() → [env.step(action)]* → env.close()
```

### BDDL Integration
BDDL3 activity definitions → BDDLSampler → BehaviorTask → termination/reward functions

## Running the Code

### Installation
```bash
bash setup.sh --new-env behavior --omnigibson --bddl
```

### Running Tests (OmniGibson)
```bash
cd OmniGibson
OMNIGIBSON_HEADLESS=1 pytest tests/
```

### Running Examples
```bash
cd OmniGibson
OMNIGIBSON_HEADLESS=1 python omniGibson/examples/environments/behavior_env_demo.py
```

### Running Teleoperation
```bash
cd joylo
python scripts/run_joylo.py          # Teleop client
python scripts/launch_og.py          # OmniGibson server
```

## Related Documentation

- [BEHAVIOR-1K Main Website](https://behavior.stanford.edu/)
- [BEHAVIOR-1K GitHub](https://github.com/StanfordVL/BEHAVIOR-1K)
- [OmniGibson API Reference](https://behavior.stanford.edu/reference/)
- [Knowledgebase](https://behavior.stanford.edu/knowledgebase/)
