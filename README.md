# vrpn_client_ros2

ROS 2 port of the classic `vrpn_client_ros` package. Connects to any VRPN-compatible motion capture server (OptiTrack Motive, Vicon, etc.) and publishes tracker poses as ROS 2 topics and/or TF frames.

---

## Overview

The node connects to a VRPN server, auto-discovers all streaming rigid bodies, and for each tracker publishes:

| Topic | Type | Notes |
|---|---|---|
| `/vrpn_client_node/<name>/pose` | `geometry_msgs/PoseStamped` | 6-DOF pose |
| `/vrpn_client_node/<name>/twist` | `geometry_msgs/TwistStamped` | Velocity (if server publishes it) |
| `/vrpn_client_node/<name>/accel` | `geometry_msgs/AccelStamped` | Acceleration (if available) |
| TF broadcast | `world → <name>` | Only when `broadcast_tf: true` |

---

## Package Structure

```
vrpn_client_ros2/
├── include/vrpn_client_ros2/
│   └── vrpn_client_ros2.h       # VrpnTrackerRos + VrpnClientRos class declarations
├── src/
│   ├── vrpn_client_ros2.cpp     # Main implementation (bug-fixed — see below)
│   ├── vrpn_client_node.cpp     # Node entry point
│   └── vrpn_tracker_node.cpp    # Single-tracker standalone entry point
├── config/
│   └── param.yaml               # Node parameters (edit server IP here)
└── launch/
    ├── sample.launch.py         # Python launch — loads param.yaml ✅ (use this)
    └── sample.launch            # XML launch — inline params only (legacy)
```

---

## Configuration (`config/param.yaml`)

```yaml
vrpn_client_node:
  ros__parameters:
    server: "192.168.2.5"          # OptiTrack PC IP address
    port: 3883                     # VRPN default port
    update_frequency: 100.0        # Hz — how often to poll the VRPN connection
    frame_id: "world"              # Parent frame for all tracker TF broadcasts
    use_server_time: false         # Use ROS clock (recommended for sync with other nodes)
    broadcast_tf: true             # Publish world → <tracker_name> TF
    refresh_tracker_frequency: 1.0 # Hz — how often to query for new rigid bodies
                                   # Set > 0.0 to auto-discover; set 0.0 if listing
                                   # trackers manually below
    # trackers:                    # Optional: pre-create specific trackers at startup
    #   - arm_hex                  # (auto-discovery handles this when freq > 0)
```

> **Tip:** With `refresh_tracker_frequency: 1.0` and no `trackers` list, the node auto-discovers all rigid bodies that Motive is streaming. This is the recommended mode.

---

## Running

```bash
source /opt/ros/humble/setup.bash
source ~/kfupm-arm-lab/arm_robot_ws/install/setup.bash

# Use the Python launch (loads param.yaml correctly):
ros2 launch vrpn_client_ros2 sample.launch.py
```

> ⚠️ **Do not use `sample.launch` (XML)** for parameter-rich configurations — it passes parameters as inline `<param>` tags which an older compiled binary may reject. Always use `sample.launch.py`.

---

## OptiTrack Motive Setup (prerequisite)

1. Open **Motive → Edit → Settings → Streaming**
2. Enable **VRPN Streaming**
3. Confirm the **Local Interface** IP matches `server:` in `param.yaml`
4. Ensure your ROS 2 machine can reach the Motive PC on port `3883` (UDP)

---

## Coordinate System Note

OptiTrack VRPN output uses a **Y-Up right-handed** coordinate system (same as the main NatNet / mocap4r2 output):

```
Motive VRPN:  X = North/forward,  Y = Up,  Z = East/right
ROS ENU:      X = East,           Y = North, Z = Up
```

The VRPN node publishes data **as-is** (no coordinate conversion). If you consume the pose topic in `arm_tf_dual`, the NUE→ENU position remap is applied there.

---

## Bug Fix Applied (required to build correctly)

### Problem: `trackers` parameter crash

**File:** `src/vrpn_client_ros2.cpp`, line 357

**Original code (broken):**
```cpp
this->declare_parameter<std::vector<std::string>>("trackers", {});
```

**Root cause:** The empty brace-initializer `{}` is **ambiguous** in C++11 and above. The compiler resolves it to `const ParameterDescriptor` (a struct with an explicit constructor from initializer list), not to `std::vector<std::string>`. This selects the two-argument `declare_parameter(name, descriptor)` overload — which declares `trackers` as a **required parameter with no default value**. The node then crashes at startup with:

```
rclcpp::exceptions::InvalidParameterValueException:
  parameter_value_from failed for parameter 'trackers': No parameter value set
```

Even providing `trackers: []` in the YAML param file fails because rclcpp cannot parse an empty YAML sequence into a `STRING_ARRAY` parameter type in some versions.

**Fix (applied):**
```cpp
this->declare_parameter("trackers", std::vector<std::string>());
```

The explicit `std::vector<std::string>()` removes the ambiguity — the compiler unambiguously picks the `declare_parameter(name, default_value)` overload, registering an empty string vector as the default. The node starts without requiring `trackers` in the param file.

**Rebuild required after fix:**
```bash
cd ~/kfupm-arm-lab/arm_robot_ws
rm -rf build/vrpn_client_ros2 install/vrpn_client_ros2
colcon build --packages-select vrpn_client_ros2
source install/setup.bash
```

---

## Dependencies

- `rclcpp`, `geometry_msgs`, `tf2_ros` (ROS 2 core)
- `libvrpn-dev` (system package: `sudo apt install libvrpn-dev`)

---

## Integration with armpx4_tf

When `arm_tf_dual` is running, it subscribes to:
```
/vrpn_client_node/arm_hex/pose   →   world → vrpn/px4_uav
```

The tracker name (`arm_hex`) matches the rigid body name in Motive and is configurable via the `vrpn_tracker` ROS 2 parameter on the `arm_tf_dual` node.
