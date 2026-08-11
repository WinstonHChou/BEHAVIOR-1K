---
type: wiki-component
title: OmniGibson Scripts
description: CLI scripts — sampling/ (task generation), learning/ (data pipeline scripts), profiling scripts.
tags: [omnigibson, scripts, cli, sampling, learning, profiling]
---

# OmniGibson Scripts

OmniGibson includes standalone CLI scripts for data sampling, learning, and profiling. These are located in `OmniGibson/scripts/`.

## Directory Structure

```
OmniGibson/scripts/
├── sampling/                     # Task generation scripts
│   ├── autogenerate_task_custom_list_template.py
│   ├── challenge_instance_qc_gui.py
│   ├── constants.py
│   ├── extract_task_information.py
│   ├── floor_plan_visualization.py
│   ├── multiply_b1k_tasks.py
│   ├── postprocess_sampled_task.py
│   ├── sample_b1k_tasks.py
│   ├── sample_robot_pose.py
│   ├── task_custom_lists.json
│   ├── utils.py
│   └── templates/challenge_instance_qc_gui.html
├── learning/                     # Data pipeline scripts
│   ├── aggregate_lerobot_task_shards.py
│   ├── benchmark.py
│   ├── download_gcs_rawdata.py
│   ├── download_gcs_rawdata.sbatch.sh
│   ├── generate_gcs_rawdata_manifest.py
│   ├── replay_data_sc.sbatch
│   ├── replay_obs.py
│   ├── update_lerobot_base_qvel.py
│   ├── upload_to_hf.py
│   ├── upload_to_hf.sbatch.sh
│   └── visualize_lerobot_pointcloud.py
└── profiling/                    # Performance profiling
    ├── profiling.css
    ├── profiling.html
    ├── profiling.js
    └── sh/
```

## Sampling Scripts

Task generation scripts for creating BEHAVIOR-1K task instances:

### sample_b1k_tasks.py

Generates sampled task instances for BEHAVIOR-1K challenges:

```bash
python scripts/sampling/sample_b1k_tasks.py \
    --output-dir ./tasks \
    --num-tasks 100 \
    --scene-list ./scene_lists.json
```

### challenge_instance_qc_gui.py

Quality control GUI for challenge task instances:

```bash
python scripts/sampling/challenge_instance_qc_gui.py \
    --task-dir ./tasks \
    --output-file qc_results.json
```

### extract_task_information.py

Extracts metadata from task definitions:

```bash
python scripts/sampling/extract_task_information.py \
    --task-dir ./tasks \
    --output-file task_info.json
```

## Learning Scripts

Data pipeline scripts for learning and dataset management:

### download_gcs_rawdata.py

Downloads raw data from Google Cloud Storage:

```bash
python scripts/learning/download_gcs_rawdata.py \
    --manifest gcs_manifest.json \
    --output-dir ./raw_data
```

### upload_to_hf.py

Uploads datasets to Hugging Face Hub:

```bash
python scripts/learning/upload_to_hf.py \
    --dataset-dir ./dataset \
    --hf-username my_username \
    --hf-repo my_dataset
```

### replay_obs.py

Replays observations from collected data:

```bash
python scripts/learning/replay_obs.py \
    --data-file ./episodes/episode_0.h5
```

### aggregate_lerobot_task_shards.py

Aggregates LeRobot task shards into a single dataset:

```bash
python scripts/learning/aggregate_lerobot_task_shards.py \
    --shard-dir ./shards \
    --output-file aggregated_dataset.parquet
```

## Profiling Scripts

Performance profiling tools for benchmarking simulation:

### benchmark.py

Runs simulation benchmarks:

```bash
python scripts/learning/benchmark.py \
    --config configs/benchmark_config.yaml \
    --output-dir ./benchmark_results
```

## Testing

Scripts are tested indirectly through:
- `test_examples.py` — Tests example environments that use sampling
- CI test workflow validates sampling output

## See Also

<!-- openwiki: broken internal link [omnigibson/scripts.md] file "omnigibson/scripts.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/scripts.md`](omnigibson/scripts.md) — Overview
<!-- openwiki: broken internal link [omnigibson/examples.md] file "omnigibson/examples.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`omnigibson/examples.md`](omnigibson/examples.md) — Example demos
<!-- openwiki: broken internal link [../cross-system/eval-pipeline.md] file "../cross-system/eval-pipeline.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`cross-system/eval-pipeline.md`](../cross-system/eval-pipeline.md) — Evaluation pipeline
