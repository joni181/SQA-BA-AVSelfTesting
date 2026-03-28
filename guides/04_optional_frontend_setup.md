# Optional Frontend Setup

This guide covers optional visualization-related steps.
It is not required for triggering the self-tests themselves, but it can be useful for observing the planning simulation or publishing poses through a browser-based UI.

If you are starting from an empty workspace, begin with [Autoware setup](./01_autoware_setup.md).
If you only want to reproduce the self-test runs, continue with [Demonstration reproduction](./02_demo_reproduction.md) and skip this guide unless you need it.

## Scope

This guide intentionally does not prescribe:

- VPN setup,
- SSH access,
- SSH tunneling,
- browser forwarding,
- or any other environment-specific remote access configuration.

Those steps depend on the local workstation and user environment.

## 1. Start the Autoware development container

On the host machine:

```sh
cd <WORKSPACE_PARENT_DIR>/autoware
./docker/run.sh --devel --headless
```

Inside the container:

```sh
cd /workspace
source /opt/ros/humble/setup.bash
source /workspace/install/setup.bash
```

## 2. Install and launch the Foxglove bridge

If the bridge is not already available in the container, install it:

```sh
sudo apt update
sudo apt install -y ros-humble-foxglove-bridge
```

Launch the bridge:

```sh
ros2 launch foxglove_bridge foxglove_bridge_launch.xml port:=8766
```

The bridge port can be adapted to your own environment if needed.

## 3. Connect the frontend

Expose the bridge according to your own remote setup and open it from the browser or frontend client of your choice.

In the original setup, the bridge was exposed on:

```text
ws://localhost:8766
```

## 4. Foxglove settings used in the original setup

If routing or pose publication does not work through Foxglove, verify the following UI settings:

- publish the initial pose on `/initialpose3d`
- publish the goal pose on `/rviz/routing/reroute`
- use `map` as the display frame

## 5. Notes

- Do not restart the container between launching the Foxglove bridge and launching Autoware unless your own setup requires it.
- If you prefer RViz or another frontend, you can use that instead.
- The runtime self-tests themselves are triggered from the ROS 2 command line and do not depend on Foxglove.
