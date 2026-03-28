# Autoware Setup

This guide describes how to prepare the modified Autoware workspace for the runtime self-testing demonstration.

If you already have a working workspace with the correct forks, branches, map repository, and required packages built, continue with [Demonstration reproduction](./02_demo_reproduction.md).

If a step fails, use [Troubleshooting](./03_troubleshooting.md).

## Outcome

After completing this guide, you should have:

- a cloned Autoware workspace,
- the demonstration-ready forked repositories checked out,
- the required packages built,
- the demonstration map available,
- and a Docker environment from which the planning simulation can be launched.

## 1. Prerequisites

The setup assumes the following:

- a Linux machine or remote workstation with Docker installed,
- NVIDIA GPU support configured for Docker if GPU-backed simulation is required,
- Git access to the repositories linked from the `SQA-BA-AVSelfTesting` repository,
- and a terminal environment from which you can run Docker and ROS 2 commands.

Environment-specific access steps such as SSH, VPN, VNC, or browser forwarding are intentionally left out of this guide.
If you are working on a remote workstation, perform those steps according to your own environment before continuing.

## 2. Verify GPU access

On the host machine, verify that the NVIDIA GPU is visible both directly and from Docker:

```sh
nvidia-smi
docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
```

If GPU access fails, fix that first before continuing.

## 3. Clone the base Autoware repository

Clone the top-level Autoware repository into an empty workspace directory:

```sh
cd <WORKSPACE_PARENT_DIR>
git clone https://github.com/autowarefoundation/autoware.git
cd autoware
```

## 4. Import the upstream workspace repositories

Start a temporary bootstrap container for importing the upstream repositories:

```sh
docker run -it --rm --net=host \
  -v <WORKSPACE_PARENT_DIR>/autoware:/workspace/autoware \
  -w /workspace/autoware \
  ubuntu:22.04
```

Inside that container, install the required tools:

```sh
apt update
apt install -y git python3-pip
pip3 install vcstool
```

Verify that `vcs` is available:

```sh
vcs --version
```

Then import the upstream repositories:

```sh
vcs import src < repositories/autoware.repos
```

## 5. Switch the required repositories to the demonstration forks

The runtime self-testing implementation is distributed across multiple forked repositories.
The exact fork URLs and demonstration-ready branches should be taken from the `SQA-BA-AVSelfTesting` repository README.

At minimum, update the checked-out repositories so that:

- `autoware_universe` points to the self-testing fork,
- `autoware_core` points to the corresponding fork if required by the demonstration,
- and `autoware_launch` points to the corresponding fork if required by the demonstration.

For `autoware_universe`, the original setup used:

```sh
cd /workspace/autoware/src/universe/autoware_universe
git remote -v
git remote set-url origin https://github.com/joni181/autoware_universe_self_testing.git
git fetch --all --prune
git switch self-testing-detection-area
git pull
```

Apply the equivalent remote and branch changes for the other required repositories using the URLs documented in the `SQA-BA-AVSelfTesting` repository.

Exit the bootstrap container afterwards:

```sh
exit
```

## 6. Start the Autoware development container

From the host machine, start the regular Autoware development container:

```sh
cd <WORKSPACE_PARENT_DIR>/autoware
./docker/run.sh --devel --headless
```

Inside the container, source the ROS 2 environment:

```sh
cd /workspace
source /opt/ros/humble/setup.bash
```

## 7. Build the packages required for the demonstration

Build only the packages needed for the planning simulation and the self-testing prototype:

```sh
cd /workspace
colcon build --symlink-install --continue-on-error \
  --packages-up-to \
  autoware_launch \
  autoware_behavior_velocity_planner \
  autoware_behavior_velocity_detection_area_module \
  autoware_self_test_infrastructure \
  autoware_self_test_types \
  sample_vehicle_description \
  sample_sensor_kit_description \
  sample_vehicle_launch \
  sample_sensor_kit_launch \
  autoware_lanelet2_map_visualizer \
```

Then source the workspace:

```sh
source /workspace/install/setup.bash
```

If package lookup still fails afterwards, see:

- [Launch and build issues](./03_troubleshooting.md#launch-and-build-issues)

## 8. Clone the demonstration map repository

Inside the same container, clone the map repository used in the demonstration:

```sh
cd /
git clone https://github.com/joni181/SQA-BA-AVSelfTesting-Maps.git
```

This repository is later referenced through:

```text
/SQA-BA-AVSelfTesting-Maps/autoware_map/detection-area-map
```

## 9. Optional frontend setup

If you need browser-based visualization or a Foxglove bridge, see [Optional frontend setup](./04_optional_frontend_setup.md).

If you do not need that, continue directly with [Demonstration reproduction](./02_demo_reproduction.md).
