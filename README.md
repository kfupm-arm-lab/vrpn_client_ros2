# vrpn_client_ros2

ROS 2 port of the classic `vrpn_client_ros` package. It connects to any VRPN-compatible motion-capture server, including OptiTrack Motive, and publishes tracker poses as ROS 2 topics and TF frames.

---

## Overview

The node connects to a VRPN server, auto-discovers all streaming rigid bodies, and publishes:

| Topic | Type | Notes |
|---|---|---|
| `/vrpn_client_node/<name>/pose` | `geometry_msgs/PoseStamped` | 6-DOF pose |
| `/vrpn_client_node/<name>/twist` | `geometry_msgs/TwistStamped` | Velocity, when provided by the server |
| `/vrpn_client_node/<name>/accel` | `geometry_msgs/AccelStamped` | Acceleration, when available |
| TF broadcast | `world -> <name>` | Only when `broadcast_tf: true` |

Use `vrpn_client_ros2` when you want:

- standard ROS 2 pose and TF outputs
- a simpler OptiTrack-to-ROS path through VRPN
- compatibility with other VRPN-capable systems beyond OptiTrack

If you want the native OptiTrack NatNet driver and the `mocap4r2_*` stack, use `mocap4ros2_optitrack` instead.

---

## Package Structure

```text
vrpn_client_ros2/
├── include/vrpn_client_ros2/
│   └── vrpn_client_ros2.h
├── src/
│   ├── vrpn_client_ros2.cpp
│   ├── vrpn_client_node.cpp
│   └── vrpn_tracker_node.cpp
├── config/
│   └── param.yaml
└── launch/
    ├── sample.launch.py
    └── sample.launch
```

---

## Installation

Create the workspace source directory if needed:

```bash
mkdir -p ~/kfupm-arm-lab/arm_robot_ws/src
cd ~/kfupm-arm-lab/arm_robot_ws/src
```

Install required dependencies:

```bash
sudo apt install libvrpn-dev
```

Build the package from your workspace:

```bash
cd ~/kfupm-arm-lab/arm_robot_ws
colcon build --packages-select vrpn_client_ros2 --symlink-install
```

Source the workspace:

```bash
source /opt/ros/humble/setup.bash
source ~/kfupm-arm-lab/arm_robot_ws/install/setup.bash
```

> [!IMPORTANT]
> If you are using this package inside the PX4/OptiTrack workflow, keep using `--symlink-install` for consistency with the rest of the motion-capture workspace.

---

## OptiTrack Motive Setup

Before launching the ROS 2 client in Ubuntu:

1. Open **Motive -> Edit -> Settings -> Streaming**.
2. Enable **VRPN Streaming**.
3. Confirm the **Local Interface** IP matches the `server` value in `config/param.yaml`.
4. Ensure the Ubuntu workstation can reach the Motive PC on port `3883`.

> [!IMPORTANT]
> Set the Motive `Up Axis` to `Z Up` so downstream ROS transforms stay consistent with the rest of the lab workflow.

---

## Configuration

Edit:

```bash
~/kfupm-arm-lab/arm_robot_ws/src/vrpn_client_ros2/config/param.yaml
```

Example configuration:

```yaml
vrpn_client_node:
  ros__parameters:
    server: "192.168.2.5"          # OptiTrack PC IP address
    port: 3883                     # VRPN default port
    update_frequency: 100.0        # Polling rate in Hz
    frame_id: "world"              # Parent frame for tracker TF broadcasts
    use_server_time: false         # Use ROS time on the client
    broadcast_tf: true             # Publish world -> <tracker_name> TF
    refresh_tracker_frequency: 1.0 # Auto-discover rigid bodies when > 0.0
    # trackers:
    #   - arm_hex
```

> [!TIP]
> With `refresh_tracker_frequency: 1.0` and no `trackers` list, the node auto-discovers all rigid bodies that Motive is streaming. This is the recommended mode.

---

## Running

Launch the node:

```bash
ros2 launch vrpn_client_ros2 sample.launch.py
```

> [!IMPORTANT]
> Use `sample.launch.py`, not `sample.launch`. The Python launch file correctly loads `config/param.yaml`.

---

## Coordinate System Note

OptiTrack VRPN output uses a Y-Up right-handed convention:

```text
Motive VRPN:  X = North/forward,  Y = Up,    Z = East/right
ROS ENU:      X = East,           Y = North, Z = Up
```

Note again that to align with ROS conventions, it is advised to make sure `Up Axis` is `Z Up` in the optitrack software. Downstrean conversions can also exist as necessary but one must be careful!

---

## Bug Fix Applied

### Problem: `trackers` parameter crash

**File:** `src/vrpn_client_ros2.cpp`

**Original code (broken):**

```cpp
this->declare_parameter<std::vector<std::string>>("trackers", {});
```

**Fix (applied):**

```cpp
this->declare_parameter("trackers", std::vector<std::string>());
```

The explicit default value removes the overload ambiguity and lets the node start without requiring `trackers` in the YAML file.

Rebuild after changing the package:

```bash
cd ~/kfupm-arm-lab/arm_robot_ws
colcon build --packages-select vrpn_client_ros2 --symlink-install
source install/setup.bash
```

---

## Integration with armpx4_tf

When `arm_tf_dual` is running, it subscribes to:

```text
/vrpn_client_node/arm_hex/pose   ->   world -> vrpn/px4_uav
```

The tracker name must match the rigid body name configured in Motive.
