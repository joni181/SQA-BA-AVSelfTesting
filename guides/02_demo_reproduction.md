# Demonstration Reproduction

This guide describes how to run the runtime self-testing demonstration once the modified Autoware workspace has already been set up.

If you are starting from an empty workspace, begin with [Autoware setup](./01_autoware_setup.md).
If a step fails, use [Troubleshooting](./03_troubleshooting.md).

## Outcome

After completing this guide, you should be able to:

- launch the planning simulation on the modified map,
- set a route that activates the `Detection Area` module,
- trigger the runtime self-tests through the `Trigger` service,
- and reproduce the representative baseline and fault-injected runs discussed in the thesis.

## 1. Start the development container

On the host machine:

```sh
cd <WORKSPACE_PARENT_DIR>/autoware
./docker/run.sh --devel --headless
```

Inside the container, source the ROS 2 and workspace environments:

```sh
cd /workspace
source /opt/ros/humble/setup.bash
source /workspace/install/setup.bash
```

## 2. Apply the demo-specific configuration

The demonstration expects two configuration adjustments.
If your demonstration-ready branches already contain these values, this step can be skipped.

First, configure the `Detection Area` module to use predicted objects instead of pointcloud input for the provider-availability test:

```sh
sed -i 's/pointcloud: true/pointcloud: false/' \
  /workspace/install/autoware_launch/share/autoware_launch/config/planning/scenario_planning/lane_driving/behavior_planning/behavior_velocity_planner/detection_area.param.yaml

sed -i 's/car: false/car: true/' \
  /workspace/install/autoware_launch/share/autoware_launch/config/planning/scenario_planning/lane_driving/behavior_planning/behavior_velocity_planner/detection_area.param.yaml
```

Second, apply the mission-planner settings that were required for the modified map:

```sh
sed -i 's/check_footprint_inside_lanes: true/check_footprint_inside_lanes: false/' \
  /workspace/install/autoware_launch/share/autoware_launch/config/planning/mission_planning/mission_planner/mission_planner.param.yaml

sed -i 's/goal_angle_threshold_deg: 45.0/goal_angle_threshold_deg: 180.0/' \
  /workspace/install/autoware_launch/share/autoware_launch/config/planning/mission_planning/mission_planner/mission_planner.param.yaml
```

## 3. Launch the planning simulation

Start the simulation on the demonstration map:

```sh
ros2 launch autoware_launch planning_simulator.launch.xml \
  map_path:=/SQA-BA-AVSelfTesting-Maps/autoware_map/detection-area-map \
  vehicle_model:=sample_vehicle \
  sensor_model:=sample_sensor_kit \
  use_deprecated_behavior_planning:=true \
  rviz:=false
```

If the launch fails or key nodes are missing, see [Launch and build issues](./03_troubleshooting.md#launch-and-build-issues).

## 4. Open a second terminal in the same environment

In a second shell, attach to the same container or start another shell with the same sourced environment:

```sh
cd /workspace
source /opt/ros/humble/setup.bash
source /workspace/install/setup.bash
```

## 5. Set the initial pose and route

Set the initial pose:

```sh
ros2 topic pub -1 /initialpose geometry_msgs/msg/PoseWithCovarianceStamped "{
  header: {frame_id: map},
  pose: {
    pose: {
      position: {x: 3755.14, y: 73737.37, z: 0.0},
      orientation: {z: -0.9711, w: 0.2387}
    }
  }
}"
```

Set the goal past the detection area used in the demonstration:

```sh
ros2 service call /api/routing/set_route_points autoware_adapi_v1_msgs/srv/SetRoutePoints "{
  header: {frame_id: map},
  option: {allow_goal_modification: true},
  goal: {
    position: {x: 3697.92, y: 73723.45, z: 0.0},
    orientation: {z: -0.9711, w: 0.2387}
  },
  waypoints: []
}"
```

Verify that the route is available:

```sh
ros2 topic echo /planning/mission_planning/route --once --no-arr 2>&1 | head -20
```

If the route remains empty or planning fails, see [Routing and operation mode issues](./03_troubleshooting.md#routing-and-operation-mode-issues).

## 6. Enable autonomous mode and control

Enable autonomous mode, Autoware control, and engagement:

```sh
ros2 service call /api/operation_mode/change_to_autonomous autoware_adapi_v1_msgs/srv/ChangeOperationMode "{}"
ros2 service call /api/operation_mode/enable_autoware_control autoware_adapi_v1_msgs/srv/ChangeAutowareControl "{}"
ros2 service call /api/autoware/set/engage autoware_adapi_v1_msgs/srv/Engage "{engage: true}"
```

If the target mode is not available or the pipeline stays stalled, see [Routing and operation mode issues](./03_troubleshooting.md#routing-and-operation-mode-issues).

## 7. Verify that the `Detection Area` module is active

Check that the planner is producing path output and that the detection-area debug topic is populated:

```sh
ros2 topic hz /planning/scenario_planning/lane_driving/behavior_planning/path_with_lane_id --window 3
ros2 topic echo /planning/scenario_planning/lane_driving/behavior_planning/behavior_velocity_planner/debug/detection_area --once --no-arr 2>&1 | head -10
```

If the self-test later reports that no tests were executed, see [Self-test execution issues](./03_troubleshooting.md#self-test-execution-issues).

## 8. Trigger the self-tests

The runtime self-tests are exposed through the `Trigger` service:

```sh
ros2 service call /self_test/run std_srvs/srv/Trigger "{}"
```

The JSON report is returned in the `message` field of the response.

## 9. Reproduce the representative runs

### 9.1 Baseline run

Run the self-tests without any fault injection:

```sh
ros2 service call /self_test/run std_srvs/srv/Trigger "{}"
```

Expected outcome:

- `test_predicted_objects_provider_available` passes.
- `test_detection_area_geometric_misfit` passes.

### 9.2 Geometric fault only

Inject the perception offset:

```sh
ros2 param set /planning/scenario_planning/lane_driving/behavior_planning/behavior_velocity_planner detection_area.self_test.perception_offset_m 1.4
ros2 service call /self_test/run std_srvs/srv/Trigger "{}"
```

Expected outcome:

- `test_predicted_objects_provider_available` passes.
- `test_detection_area_geometric_misfit` fails.

Reset the offset afterwards:

```sh
ros2 param set /planning/scenario_planning/lane_driving/behavior_planning/behavior_velocity_planner detection_area.self_test.perception_offset_m 0.0
```

### 9.3 Provider fault only

Simulate a runtime failure of the predicted-objects provider:

```sh
pkill -f dummy_perception_publisher
sleep 2
ros2 service call /self_test/run std_srvs/srv/Trigger "{}"
```

Expected outcome:

- `test_predicted_objects_provider_available` fails.
- `test_detection_area_geometric_misfit` passes.

If the provider test still passes, see [Self-test execution issues](./03_troubleshooting.md#self-test-execution-issues).

### 9.4 Combined-failure run

Keep the provider disabled and inject the geometric offset again:

```sh
ros2 param set /planning/scenario_planning/lane_driving/behavior_planning/behavior_velocity_planner detection_area.self_test.perception_offset_m 1.4
ros2 service call /self_test/run std_srvs/srv/Trigger "{}"
```

Expected outcome:

- `test_predicted_objects_provider_available` fails.
- `test_detection_area_geometric_misfit` fails.

## 10. Optional cleanup

If you want to restore the provider without restarting the full simulation, see [Revive the dummy perception publisher](./03_troubleshooting.md#revive-the-dummy-perception-publisher).
