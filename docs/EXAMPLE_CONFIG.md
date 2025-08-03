# Example Configuration for Data Generation

This file contains example configurations for different components of the SimLingo data generation pipeline.

## collect_dataset_slurm.py Configuration

Replace lines 213-230 with your specific paths:

```python
if __name__ == "__main__":
    repetitions = 1
    repetition_start = 0
    default_partition = "gpu"  # Your SLURM partition
    job_name = "collect"
    username = "your_username"
    code_root = r"/home/your_username/simlingo"
    carla_root = "/opt/carla/CARLA_0.9.15"
    date = datetime.today().strftime("%Y_%m_%d")
    dataset_name = "simlingo_v2_" + date
    root_folder = r"database/"  # With ending slash
    data_save_directory = root_folder + dataset_name
    log_root = f"{data_save_directory}/slurm"

    route_folder = f"{code_root}/data/simlingo"
```

## Environment Variables

```bash
export CARLA_ROOT=/opt/carla/CARLA_0.9.15
export WORK_DIR=/home/your_username/simlingo
export PYTHONPATH=$PYTHONPATH:${CARLA_ROOT}/PythonAPI/carla
export SCENARIO_RUNNER_ROOT=${WORK_DIR}/scenario_runner_autopilot
export LEADERBOARD_ROOT=${WORK_DIR}/leaderboard_autopilot
export PYTHONPATH="${CARLA_ROOT}/PythonAPI/carla/":"${SCENARIO_RUNNER_ROOT}":"${LEADERBOARD_ROOT}":${PYTHONPATH}
```

## SLURM Job Configuration

Content for `partition.txt`:
```
gpu
```

Content for `max_num_jobs.txt`:
```
5
```

## VQA Generator Configuration

Example command with all parameters:
```bash
python dataset_generation/language_labels/drivelm/carla_vqa_generator_main.py \
    --data-directory database/simlingo \
    --output-directory database/simlingo/drivelm \
    --target-image-size 1024 384 \
    --original-image-size 1024 512 \
    --original-fov 110 \
    --min-y 0 \
    --max-y 358 \
    --sample-frame-mode all \
    --save-examples
```

## Commentary Generator Configuration

Example command with all parameters:
```bash
python dataset_generation/language_labels/commentary/carla_commentary_generator_main.py \
    --data-directory database/simlingo \
    --output-directory database/simlingo/commentary \
    --target-image-size 1024 358 \
    --original-image-size 1024 512 \
    --original-fov 110 \
    --min-y 0 \
    --max-y 358 \
    --sample-frame-mode all
```

## Directory Structure Setup

Create the necessary directories:
```bash
mkdir -p database/simlingo
mkdir -p docs
mkdir -p slurm/logs
mkdir -p slurm/run_files/start_files
```