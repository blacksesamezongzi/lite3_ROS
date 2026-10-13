# Lite3 ROS 2 Velocity Control

This repository provides the Lite3 bidirectional UDP/ROS 2 bridge: it receives robot state and forwards `/cmd_vel` velocity commands to the motion host. The following sequence was successfully tested on our robot with Ubuntu 24.04 and ROS 2 Jazzy.

## Installation

### 1. Environment and changes from the official version

This version is based on the official [Lite3_ROS `ros2-foxy` branch](https://github.com/DeepRoboticsLab/Lite3_ROS/tree/ros2-foxy), adapted for Ubuntu 24.04 / ROS 2 Jazzy.

| File | Change |
| --- | --- |
| `src/transfer/CMakeLists.txt` | Changed the default C++ standard from C++14 to C++17; added explicit `std_msgs`, `tf2`, and `tf2_geometry_msgs` build dependencies. |
| `src/transfer/package.xml` | Added the required message, TF, and launch dependencies; removed the invalid `rosidl_interface_packages` dependency and the interface-group declaration from this node package. |
| `src/transfer_interfaces/package.xml` | Removed the invalid `rosidl_interface_packages` dependency while retaining the interface-group declaration. |
| `src/transfer/src/Jetson2Motion.cpp` | Changed the TF conversion header from `tf2_geometry_msgs.h` to `tf2_geometry_msgs.hpp`. The original receiving and command-sending logic is retained. |
| `src/transfer/src/Jetson2App.cpp` | Made the same TF conversion header change. |
| `src/transfer/launch/transfer_launch.py` | Changed the motion-host IP from `192.168.1.120` to `192.168.2.1` for our robot Wi-Fi connection. |

For another Ubuntu/ROS 2 version, use that distribution's setup script and adjust the C++ standard, header paths, and dependencies as needed. Rebuild from source in the target environment. Also check the motion-host IP and telemetry destination for your network.

The commands below assume the workspace is at `~/lite3/lite_cog_ros2/transfer` and ROS 2 Jazzy, `colcon`, and the package dependencies are installed. Replace the workspace path if your checkout is elsewhere.

### 2. Build `transfer` and `transfer_interfaces`

```bash
cd ~/lite3/lite_cog_ros2/transfer
source /opt/ros/jazzy/setup.bash

MAKEFLAGS=-j1 CMAKE_BUILD_PARALLEL_LEVEL=1 \
colcon build --symlink-install \
  --packages-up-to transfer \
  --executor sequential \
  --cmake-force-configure \
  --cmake-args \
    -DCMAKE_BUILD_TYPE=Debug \
    '-DCMAKE_CXX_FLAGS_DEBUG=-O0 -g0'
```

`--packages-up-to transfer` builds both `transfer` and its workspace dependency `transfer_interfaces`. The parallelism settings limit compilation to one task at a time to reduce CPU and memory load. The Debug flags disable optimization and debug symbols for this initial build.

After a successful build, `install/setup.bash` is generated. Build success is required before continuing.

## Running

Connect the development computer to the robot network. Exit any MotionSDK joint-control demo and ensure its control authority has been returned. Use an area suitable for motion testing, with the handheld controller available for takeover.

### 1. Source ROS 2 — Terminal A

```bash
source /opt/ros/jazzy/setup.bash
```

### 2. Source the workspace — Terminal A

```bash
source ~/lite3/lite_cog_ros2/transfer/install/setup.bash
```

### 3. Start the bridge — Terminal A

```bash
ros2 launch transfer transfer_launch.py
```

Keep this terminal running. The launch starts `jetson2motion`, `jetson2app`, and `sensor_checker`. The bridge receives telemetry on UDP port `43897` and sends commands to `192.168.2.1:43893`. Do not run another receiver on the same telemetry port.

Before using **each additional terminal**, load both environments:

```bash
source /opt/ros/jazzy/setup.bash
source ~/lite3/lite_cog_ros2/transfer/install/setup.bash
```

The following command codes are the sequence verified on our robot. Confirm their applicability when using a different Lite3 firmware or controller configuration.

### 4. Toggle standing/sitting — Terminal B

```bash
ros2 topic pub --once /simple_cmd transfer_interfaces/msg/MotionSimpleCMD \
  "{cmd_code: 0x21010202, size: 0, type: 0}"
```

This is a toggle between standing and sitting. Ensure the robot is standing before proceeding; skip the toggle if it is already in the required standing state.

### 5. Publish the heartbeat — Terminal B

```bash
ros2 topic pub -r 2 /simple_cmd transfer_interfaces/msg/MotionSimpleCMD \
  "{cmd_code: 0x21040001, size: 0, type: 0}"
```

Keep this command running. It publishes the heartbeat at **2 Hz** and occupies Terminal B, so use another terminal for the next steps.

### 6. Enter navigation mode — Terminal C

After sourcing both environments in Terminal C, run:

```bash
ros2 topic pub --once /simple_cmd transfer_interfaces/msg/MotionSimpleCMD \
  "{cmd_code: 0x21010C03, size: 0, type: 0}"
```

Entering navigation mode enables the robot to execute the velocity commands in this tested sequence. Starting the bridge and standing the robot alone are insufficient.

## Send a Velocity Command — Terminal C

With the bridge and heartbeat still running, publish to `/cmd_vel`:

```bash
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.05, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"
```

This requests forward motion at **0.05 m/s**, with zero lateral and turning velocity. The command is published at **10 Hz**.

| Field | Command | Unit |
| --- | --- | --- |
| `linear.x` | Forward/backward velocity | m/s |
| `linear.y` | Lateral velocity | m/s |
| `angular.z` | Turning velocity | rad/s |

To stop, press Ctrl+C in Terminal C to end the velocity publisher, then explicitly send zero velocity:

```bash
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"
```

Confirm the robot has stopped before ending the heartbeat or bridge. Ctrl+C alone ends the local publisher; it does not establish the robot's response to command loss.
