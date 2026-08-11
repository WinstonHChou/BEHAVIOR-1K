---
type: wiki-skeleton
title: BEHAVIOR-1K Wiki Skeleton
description: Evidence-based wiki skeleton for the BEHAVIOR-1K monorepo
---

# BEHAVIOR-1K Wiki Skeleton

## Scope
- **Repository**: BEHAVIOR-1K — monorepo for a simulation benchmark testing embodied AI agents on 1,000+ household activities
- **Components**: OmniGibson (sim engine), BDDL3 (domain language), JoyLo (teleop framework), asset_pipeline, knowledgebase, eval-jobqueue, docker
- **External deps**: NVIDIA Isaac Sim 5.1.0, Omniverse Kit 107.3.1

---

## Planned Pages

### 1. Root
- `quickstart.md` — Entrypoint: high-level map, task routing table
- `overview.md` — Repository overview, component relationships, monorepo layout

### 2. OmniGibson (largest component — grouped in directory)

#### Overview & Core
- `omnigibson/overview.md` — Architecture overview, registry pattern inventory, config-driven design, lifecycle, lazy imports
- `omnigibson/simulator.md` — Simulator singleton, step lifecycle, USD manipulation, physics
- `omnigibson/macros.md` — MacroDict, global config, all runtime knobs (physics, rendering, GPU, object states, transition rules)
- `omnigibson/registries.md` — Complete registry inventory (REGISTERED_OBJECTS, REGISTERED_ROBOTS, REGISTERED_CONTROLLERS, REGISTERED_SCENES, REGISTERED_TASKS, etc.), registry pattern implementation

#### Environment Layer
- `omnigibson/environments.md` — Gymnasium env wrapper, Environment class, vector envs (VectorEnvironment, SB3VecEnv)
- `omnigibson/env_wrappers.md` — EnvironmentWrapper, DataWrapper family, MetricsWrapper, HDF5/LeRobot wrappers, wrapper factory
- `omnigibson/data_wrappers.md` — Data collection deep-dive: HDF5DataWrapper, LeRobotDataWrapper, video recording, checkpointing

#### Robotics
- `omnigibson/robots.md` — Robot class, YAML-config-driven robot definitions, proprioception, grasping modes
- `omnigibson/robot_schema.md` — Robot definition schema validation, JSON schema for robot YAML
- `omnigibson/controllers.md` — Controller hierarchy (IK, OSC, Joint, Gripper, DD, Holonomic), REGISTERED_CONTROLLERS

#### Objects & States
- `omnigibson/objects.md` — USDObject base, DatasetObject, PrimitiveObject, LightObject, registry
- `omnigibson/object_states.md` — 40+ state classes, taxonomy (pose, cooking, fill, cloth, attachment), dependency-ordered initialization
- `omnigibson/state_mechanics.md` — State update lifecycle (pre-step, post-step, reset), dependency ordering, state mixins (UpdateStateMixin, GlobalUpdateStateMixin, ContactSubscribedStateMixin, JointBreakSubscribedStateMixin), factory ordering

#### Scenes
- `omnigibson/scenes.md` — Scene, TraversableScene, InteractiveTraversableScene, loading, asset handling

#### Tasks & RL
- `omnigibson/tasks.md` — BaseTask, BehaviorTask, GraspTask, PointNavigationTask, PointReachingTask, DummyTask
- `omnigibson/termination.md` — Termination conditions, success/failure conditions, registry (REGISTERED_TERMINATION_CONDITIONS, etc.)
- `omnigibson/reward_functions.md` — BaseRewardFunction, REGISTERED_REWARD_FUNCTIONS, each reward type (PointGoalReward, CollisionReward, GraspReward, PotentialReward, etc.)

#### Sensors
- `omnigibson/sensors.md` — VisionSensor (RGB, depth, seg_semantic, seg_instance, normal), ScanSensor, sensor modalities, noise

#### Primitives Layer
- `omnigibson/prims.md` — BasePrim → XFormPrim → GeomPrim → RigidPrim → RigidDynamicPrim → RigidKinematicPrim → EntityPrim → ClothPrim, JointPrim, MaterialPrim

#### Systems
- `omnigibson/systems.md` — Particle systems (Macro/Micro), fluid/granular/cloth simulation, physical simulation

#### Actions & Planning
- `omnigibson/action_primitives.md` — Curobo integration, starter semantic primitives, symbolic primitives, ActionPrimitiveSetBase

#### Transition Rules
- `omnigibson/transition_rules.md` — TransitionRuleAPI class, REGISTERED_RULES, ObjectAttrs/TransitionResults data structures, recipe execution (CookingRecipe, MixingRecipe, MachineRecipe, SubstanceCookingRecipe, WasherRecipe), particle system interactions, melting/cooking/washing behaviors, scene_base.py integration

#### Spatial
- `omnigibson/scene_graphs.md` — GraphBuilder, spatial relationship graphs
- `omnigibson/maps.md` — SegmentationMap, TraversableMap

#### Metrics
- `omnigibson/metrics.md` — MetricBase, AgentMetric, TaskMetric, metric aggregation during episodes

#### Eval Framework
- `omnigibson/eval/` directory:
  - `omnigibson/eval/overview.md` — Challenge eval pipeline: Evaluator class, two-phase evaluation (public/hidden test instances), policy integration, score computation, metric aggregation, relationship to eval-jobqueue, wrappers (default_wrapper, rgbd_full_res_wrapper), utils (eval_utils, dataset_utils, obs_utils, score_utils, light_utils, network_utils)
  - `omnigibson/eval_cli.md` — eval.py CLI entry point: argument structure (--task-name, --robot-config, --mode, --host, --port, --instance-indices, --max-steps, --output-dir, --write-video), WebsocketClientPolicy integration, rollout loop, output format, integration with score_utils (q_score, time, distance metrics) and obs_utils (video writing, HEAD_RESOLUTION/WRIST_RESOLUTION)

#### Configuration
- `omnigibson/configs.md` — Config file structure (default_cfg.yaml), sensor/robot/controller YAML configs, config merging (merge_nested_dicts), config precedence and lifecycle

#### Examples & Scripts
- `omnigibson/examples.md` — Example ecosystem (environments, learning, objects, robots, scenes, sensors, object_states, teleoperation, WIP demos)
- `omnigibson/scripts.md` — CLI scripts: sampling/ (task generation), learning/ (data pipeline scripts), profiling scripts

#### Utilities
- `omnigibson/utils.md` — Key utility modules: transform (transform_utils, transform_utils_np), config_utils, bddl_utils (bridge to BDDL3), asset_utils, asset_conversion_utils, urdfpy_utils, usd_utils, registry_utils, python_utils (Registerable metaclass, Serializable base), grasping/motion planning utils, teleop_utils, vision utils, sampling_utils, ui_utils, git_utils, deprecated_utils, pynvml_utils, coacd_runner, lazy_import_utils, processing_utils, numpy_utils, backend_utils, object_utils, object_state_utils, control_utils, geometry_utils, physx_utils, gym_utils, constants, deprecated_utils, profiling_utils

#### Testing
- `omnigibson/tests.md` — Test strategy, 25+ test files organized by subsystem, CI test matrix, skipped tests and why, fixture hierarchy in conftest.py, snapshot tests, benchmark tests

### 3. BDDL3
- `bddl3/overview.md` — Overview, purpose, relationship to OmniGibson, 1018 activity definitions, package structure
- `bddl3/parsing.md` — BDDL tokenizer, PDDL parser (adapted from pddl-parser), domain/problem file structure, natural language generation
- `bddl3/conditions.md` — Expression tree compilation, logical connectives (Conjunction, Disjunction, Negation, Implication), quantifiers (Universal, Existential, NQuantifier, ForPairs, ForNPairs), HEAD wrapper, ground state options
- `bddl3/predicates.md` — Predicate classes (UnaryPredicate, BinaryPredicate), 26 predicates, TOKEN_TO_PREDICATE mapping
- `bddl3/knowledgebase.md` — KnowledgeBase container, 20 dataclass models (Property, MetaLink, AttachmentPair, PredicateUsage, Scene, ParticleSystem, Category, Object, Synset, TransitionRule, Task, CompiledTask, RoomRequirement, RoomSynsetRequirement, Room, RoomObject, ComplaintType, Complaint, SynsetState, Recipe types), populate_knowledgebase() pipeline, view_* error detection methods
- `bddl3/taxonomy.md` — ObjectTaxonomy, synset hierarchy (NetworkX DiGraph), ancestors/descendants, ability lookup, category/substance mapping
- `bddl3/transition_rules.md` — Recipe dataclasses (CookingRecipe, MixingRecipe, MachineRecipe, SubstanceCookingRecipe, WasherRecipe), transition map JSONs (dicing, heat_cook, melting, mixing_stick, single_toggleable_machine, slicing, substance_cooking, etc.), data_generation scripts

### 4. JoyLo
- `joylo/overview.md` — Overview, bimanual teleop architecture, ZMQ IPC, R1/R1Pro robots
- `joylo/agents.md` — JoyconAgent (JoyCon BLE input, button mapping, filtering), DynamixelArmAgent (motor control, impedance feedback), BimanualAgent (R1_CONFIG/R1PRO_CONFIG, Jacobian computation, arm locking)
- `joylo/robots.md` — Robot ABC (num_dofs, get_joint_state, command_joint_state), PrintRobot, BimanualRobot, DynamixelRobot (motor driver, operating modes, current feedback), OGRobotServer (OmniGibson simulator server, 1173 lines: observation pipeline, action pipeline, button processing, recording, ghost robot, VR support)
- `joylo/teleop.md` — Teleop config, robot configs (ROBOT_CONFIGS, ROBOT_TELEOP_CONFIGS), feature flags (USE_FLUID, USE_CLOTH, FULL_SCENE, etc.), camera setup, ghost robot, status UI
- `joylo/utils.md` — Utility modules: dynamixel_utils (DynamixelDriver, OperatingMode, GainType, SDK wrappers), zmq_utils (ZMQRobotServer/ZMQServerThread/ZMQRobotClient), og_teleop_cfg.py (ROOM_DEPENDENCIES, ViewingMode, RobotTeleopConfig), og_teleop_utils.py (task_requires_attached_state, get_task_relevant_room_types, augment_rooms, infer_trunk_translate, setup_cameras, setup_flashlights, setup_ghost_robot, SignalChangeDetector, optimize_sim_settings), qa_utils.py (ALL_QA_METRICS, aggregate_episode_validation)
- `joylo/scripts.md` — Script entry points: run_joylo.py (main teleop client), launch_og.py (OmniGibson server), calibrate_joints.py, calibrate_joycons.py, replay_data.py, run_qa_analysis.py, scan_and_reboot.py, test_joints.py, kill_nodes.sh

### 5. Asset Pipeline
- `asset-pipeline/overview.md` — Overview, DVC pipeline, 3ds Max → USD conversion, data versioning, pipeline participants
- `asset-pipeline/pipeline.md` — DVC DAG structure (50+ stages: object_list, sanitycheck, export_meshes, export_objs_global, usdify_objects, usdify_scenes, generate_systems, aggregate_*, etc.), run_stages.py parallel execution, params.yaml targets
- `asset-pipeline/usd-conversion.md` — usd_conversion/ modules: usdify_objects.py (URDF→USD orchestration), usdify_objects_process.py (worker subprocess), usdify_scenes.py (scene conversion), generate_fillable_volumes.py
- `asset-pipeline/max-scripts.md` — 3ds Max integration: batch_3dsmax.py (batch executor, RPC mode), max/ scripts (export_meshes.py, fix_common_issues.py, generate_images.py, generate_fillable_volume.py, collision_vertex_reduction.py, check_collisions.py, rpc_server.py), render_presets
- `asset-pipeline/acquisition.md` — Object acquisition: acquisition/ scripts, mesh_tree.py (mesh hierarchy DiGraph), export_objs_global.py (OBJ→URDF), export_scenes_global.py (scene URDF)
- `asset-pipeline/qa.md` — Quality assurance: category_qa.py (51K lines), qa-logs/, todos/, metadata/ validation files

### 6. Knowledgebase (web app)
- `knowledgebase/overview.md` — Overview, static site generator vs legacy Flask, build_static_site.py architecture (generate_site_async, collect_pages, generate_searchable_items), data flow from BDDL KnowledgeBase to HTML
- `knowledgebase/templates.md` — Jinja2 template structure (15 detail/list pairs: attachment_pair, category, complaint_type, meta_link, object, particle_system, property, room_object, room, scene, synset, task, transition, usage), table_modules/ (14 table templates), frontend features (Fuse.js search, Mermaid diagrams, Bootstrap 4.3)

### 7. Evaluation Infrastructure
- `eval-jobqueue/overview.md` — FastAPI job queue (Job state machine, ResourcePool, ResourceLease), two-phase evaluation (task pre-load → resource acquisition → simulation → cleanup), SLURM workers (eval.sh), job generation (generate_jobs.py), result output, heartbeat/timeout mechanics, Docker integration

### 8. Docker
- `docker/overview.md` — Container images (behavior:latest, behavior-dev:latest with DEV_MODE=1, behavior-gha:latest for GHA runner), build_docker.sh, run_docker.sh (X11, headless), volume mounts (/data, /isaac-sim/cache/kit, etc.), OMNIGIBSON_DATA_PATH/OMNIGIBSON_APPDATA_PATH
- `docker/submissions.md` — Submission.Dockerfile for challenge submissions
- `docker/ci-runner.md` — GitHub Actions runner (gh-actions/ Dockerfile, entrypoint.sh, token management, Docker-in-Docker, runner reusage)

### 9. CI/CD
- `ci-cd/workflows.md` — 8 GitHub Actions workflows: tests.yml (bdddl_tests, check_example_list, run_test matrix on self-hosted GPU runners), build-push-containers.yml, build-website.yml (MkDocs), openwiki-update.yml, profiling.yml, publish-pypi.yml, pull-sheets-bddl.yml, pull-sheets.yml, self-hosted runner requirements, dataset-enabled label

### 10. Datasets
- `datasets/overview.md` — Mount point placeholder, OMNIGIBSON_DATA_PATH, external dataset fetching (Google Cloud Storage), Docker integration

### 11. Cross-System Workflows
- `cross-system/omnigibson-bddl-integration.md` — Bridge between OmniGibson and BDDL3: bddl_utils.py, KnowledgeBase → BDDLSampler → BehaviorTask, condition evaluation callback, predicate grounding, state transitions
- `cross-system/joylo-omnigibson-zmq.md` — ZMQ IPC protocol: ZMQRobotServer/ZMQRobotClient, OGRobotServer → OmniGibson env, message types (num_dofs, get_joint_state, command_joint_state, get_observations), ghost robot, camera setup, button processing mapping
- `cross-system/transition-rules-execution.md` — Transition rules from BDDL definition to OmniGibson runtime: BDDL recipe dataclasses → translate_bddl_recipe_to_og_recipe → TransitionRuleAPI (RuleCandidates, rule matching, object creation/removal, particle system triggers), DISABLED_TRANSITION_RULES mechanism
- `cross-system/eval-pipeline.md` — Full evaluation pipeline: generate_jobs.py → jobs.json → jobqueue.py FastAPI → SLURM workers → Evaluator class → results, two-phase (public/hidden test instances), resource management (GPU pool, websocket policy servers), result output format

### Diagrams to Include
- Mermaid sequence diagram: OmniGibson sim lifecycle (launch → Environment → step → close)
- Mermaid class diagram: OmniGibson registry pattern (objects, controllers, tasks, scenes)
- Mermaid flowchart: BDDL evaluation flow (KnowledgeBase → task compile → evaluate)
- Mermaid architecture diagram: JoyLo ZMQ IPC (client ZMQ → server ZMQ → OmniGibson env)
- Mermaid flowchart: Asset pipeline DVC DAG
- Mermaid flowchart: Eval pipeline (generate_jobs → jobqueue → SLURM → Evaluator → results)
- Mermaid state diagram: Transition rules state machine (RuleCandidates → matching → creation/removal)
- Mermaid class diagram: BDDL expression tree (HEAD → Conjunction → Predicate)
- Mermaid class diagram: BDDL knowledge base models (20 dataclasses)
- Mermaid flowchart: State update lifecycle (pre-step → post-step → reset)
