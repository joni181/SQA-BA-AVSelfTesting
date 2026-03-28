# Troubleshooting

This guide collects issues that occurred during the original setup and demonstration process.
It is organized by symptom so that the main setup and reproduction guides can stay linear.

If you are starting from an empty workspace, begin with [Autoware setup](./01_autoware_setup.md).
If the workspace is already prepared, use [Demonstration reproduction](./02_demo_reproduction.md).

## Launch and build issues

### `sample_vehicle_description` or `sample_sensor_kit_description` not found

Build the missing description packages:

```sh
cd /workspace
colcon build --symlink-install --packages-select \
  sample_vehicle_description \
  sample_sensor_kit_description

source /workspace/install/setup.bash
```

Verify:

```sh
ros2 pkg prefix sample_vehicle_description
ros2 pkg prefix sample_sensor_kit_description
```

### `sample_vehicle_launch` or `sample_sensor_kit_launch` not found

Build the missing launch packages:

```sh
cd /workspace
colcon build --symlink-install --packages-up-to \
  sample_vehicle_launch \
  sample_sensor_kit_launch

source /workspace/install/setup.bash
```

### `/map/vector_map_marker` is missing

Build the map visualizer package:

```sh
cd /workspace
source /opt/ros/humble/setup.bash
colcon build --symlink-install --packages-select autoware_lanelet2_map_visualizer
source /workspace/install/setup.bash
```

Verify:

```sh
ros2 pkg prefix autoware_lanelet2_map_visualizer
```

### `colcon build` fails in unrelated deprecated packages

Use a focused build and, if necessary, continue on error to identify which packages are irrelevant for the demonstration:

```sh
cd /workspace
colcon build --symlink-install --continue-on-error \
  --packages-up-to \
  autoware_launch \
  autoware_behavior_velocity_planner \
  autoware_behavior_velocity_detection_area_module \
  autoware_self_test_infrastructure \
  autoware_self_test_types \
  --packages-skip autoware_lane_departure_checker
```

## Routing and operation mode issues

### Target mode is not available

Typical symptom:

```text
The target mode is not available for the following reasons:
- /autoware/modes/autonomous ERROR
- /autoware/planning ERROR
- /autoware/vehicle ERROR
...
```

First inspect the route and the relevant planning state:

```sh
ros2 topic echo /planning/mission_planning/route --once
ros2 topic echo /api/routing/state --once
ros2 topic echo /planning/mission_planning/state --once
```

If the route is missing, continue with the route-related checks below.
If the route exists but the planner still does not proceed, check whether the operation-mode state is being published.

### Route stays empty or planning fails with a goal-footprint error

Check the mission-planner configuration:

```sh
grep -E "check_footprint|goal_angle" \
  /workspace/install/autoware_launch/share/autoware_launch/config/planning/mission_planning/mission_planner/mission_planner.param.yaml
```

If the values are still unchanged, apply the demo-specific configuration from [Demonstration reproduction](./02_demo_reproduction.md#2-apply-the-demo-specific-configuration) and restart the simulation.

### The planner pipeline is stalled although the route is set

Check whether `path_with_lane_id` is being produced and whether the operation-mode state is published:

```sh
ros2 topic hz /planning/scenario_planning/lane_driving/behavior_planning/path_with_lane_id --window 3
ros2 topic hz /system/operation_mode/state --window 3
```

If `/system/operation_mode/state` has no publisher, publish it manually as a workaround:

```sh
ros2 topic pub /system/operation_mode/state autoware_adapi_v1_msgs/msg/OperationModeState "{
  stamp: {sec: 0, nanosec: 0},
  mode: 1,
  is_autoware_control_enabled: true,
  is_in_transition: false
}" -r 10
```

Open a second terminal and re-check:

```sh
sleep 5
ros2 topic hz /planning/scenario_planning/lane_driving/behavior_planning/path_with_lane_id --window 3
```

### Additional route diagnostics

If route planning still fails, these commands can help narrow the problem down:

```sh
ros2 topic echo /planning/mission_planning/route --once --no-arr 2>&1 | head -30
ros2 topic echo /api/routing/state --once
ros2 topic echo /planning/mission_planning/state --once
ros2 topic info /planning/mission_planning/route -v
ros2 node info /planning/scenario_planning/lane_driving/behavior_planning/behavior_path_planner
ros2 topic echo /rosout --no-arr 2>&1 | grep -i -E "error|fail|warn" | head -20
```

## Self-test execution issues

### The self-test call reports that no tests were executed

Check whether the `Detection Area` module is active on the current route:

```sh
ros2 topic echo /planning/scenario_planning/lane_driving/behavior_planning/behavior_velocity_planner/debug/detection_area --once --no-arr 2>&1 | head -10
ros2 topic echo /planning/mission_planning/route --once --no-arr 2>&1 | grep -o "preferred_primitive_id: [0-9]*" | head -30
```

If the debug topic is empty, the module may not have been instantiated yet.
Verify the route, autonomous mode, and planner activity first.

### `test_predicted_objects_provider_available` is skipped although pointcloud detection was meant to be disabled

This usually means that the `Detection Area` configuration still enables pointcloud input or does not enable the object class used in the test.
Re-apply the `Detection Area` configuration from [Demonstration reproduction](./02_demo_reproduction.md#2-apply-the-demo-specific-configuration) and restart the simulation.

### The provider-availability test still passes after disabling the provider

Use a hard stop for the dummy perception publisher:

```sh
pkill -f dummy_perception_publisher
sleep 2
ros2 service call /self_test/run std_srvs/srv/Trigger "{}"
```

The wait time is required so that the object stream becomes stale according to the threshold used in the test.

## Revive the dummy perception publisher

If the dummy perception publisher needs to be started again without restarting the full simulation, use:

```sh
ros2 run autoware_dummy_perception_publisher autoware_dummy_perception_publisher_node --ros-args \
  --params-file /workspace/install/autoware_dummy_perception_publisher/share/autoware_dummy_perception_publisher/config/dummy_perception_publisher.param.yaml \
  -r __ns:=/simulation \
  -p visible_range:=300.0 \
  -p detection_successful_rate:=1.0 \
  -p enable_ray_tracing:=false \
  -p use_object_recognition:=true \
  -p use_base_link_z:=true \
  -r output/points_raw:=/perception/obstacle_segmentation/pointcloud \
  -r output/dynamic_object:=/perception/object_recognition/detection/labeled_clusters \
  -r output/objects_pose:=debug/object_pose \
  -r input/object:=dummy_perception_publisher/object_info \
  -r input/predicted_objects:=/perception/object_recognition/objects
```

Then re-run the self-test:

```sh
ros2 service call /self_test/run std_srvs/srv/Trigger "{}"
```

## Template for future entries

Use the following structure when adding new issues:

````md
### <short symptom>

Briefly describe the symptom and when it appears.

```sh
<diagnostic commands>
```

If known, explain the root cause briefly.

```sh
<fix commands>
```
````
