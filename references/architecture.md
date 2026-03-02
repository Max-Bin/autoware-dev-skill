# Autoware Architecture Reference

## Table of Contents
- [Data Flow Pipeline](#data-flow-pipeline)
- [Three-Tier Repository Structure](#three-tier-repository-structure)
- [Module Breakdown](#module-breakdown)
- [Component Interface Specs](#component-interface-specs)
- [Agnocast Zero-Copy IPC](#agnocast-zero-copy-ipc)
- [Executor Types](#executor-types)
- [Repository Dependency Graph](#repository-dependency-graph)

## Data Flow Pipeline

```
Sensors (LiDAR, Camera, Radar, GNSS/IMU)
    ↓
SENSING — crop_box_filter, downsample, gnss_poser
    ↓ PointCloud2
PERCEPTION — object detection, tracking, ground filter
    ↓ PredictedObjects            LOCALIZATION — NDT, EKF, gyro odometer
                                      ↓ Odometry
PLANNING — route, behavior, velocity smoother  ←  MAP (lanelet2, PCD)
    ↓ Trajectory
CONTROL — pure pursuit or advanced controllers
    ↓ ControlCommand
Vehicle Actuators
```

## Three-Tier Repository Structure

| Tier | Repo | Purpose |
|------|------|---------|
| 1 | **autoware** | Meta-repo — `.repos` files for `vcs import`, version pinning |
| 2 | **autoware_core** | Stable, production-quality packages |
| 3 | **autoware_universe** | Experimental, community-contributed packages |

`autoware.repos` uses vcstool format organized by group (`core/`, `universe/`, `launcher/`, `sensor_component/`, `middleware/`). Versions can be semantic tags, branch names, or commit SHAs.

**autoware_core** defines standardized interfaces and core algorithms. **autoware_universe** extends with ML models, advanced planners, simulators. Code goes in core when it's stable and well-tested; universe is more permissive.

## Module Breakdown

| Module | Key Packages (Core) |
|--------|-------------------|
| **Sensing** | `autoware_crop_box_filter`, `autoware_downsample_filters`, `autoware_gnss_poser` |
| **Perception** | `autoware_euclidean_cluster_object_detector`, `autoware_ground_filter` |
| **Localization** | `autoware_ekf_localizer`, `autoware_ndt_scan_matcher`, `autoware_pose_initializer` |
| **Planning** | `autoware_mission_planner`, `autoware_velocity_smoother`, `behavior_velocity_planner`, `autoware_path_generator` |
| **Control** | `autoware_simple_pure_pursuit` |
| **Common** | `autoware_component_interface_specs`, `autoware_interpolation`, `autoware_motion_utils`, `autoware_trajectory`, `autoware_vehicle_info_utils` |
| **System** | Operation mode management, monitoring (Universe only) |

## Component Interface Specs

`autoware_component_interface_specs` standardizes all inter-module communication via C++ structs:

```cpp
// Pattern: struct with static constexpr members
struct Trajectory {
  using Message = autoware_planning_msgs::msg::Trajectory;
  static constexpr char name[] = "/planning/trajectory";
  static constexpr size_t depth = 1;
  static constexpr auto reliability = RMW_QOS_POLICY_RELIABILITY_RELIABLE;
  static constexpr auto durability = RMW_QOS_POLICY_DURABILITY_VOLATILE;
};

// Usage:
#include <autoware/component_interface_specs/planning.hpp>
auto pub = create_publisher<Trajectory::Message>(
  Trajectory::name, autoware::component_interface_specs::get_qos<Trajectory>());
```

### Full Interface Listing (22 total: 16 topics + 6 services)

**Perception** (`perception.hpp`)

| Interface | Topic | Message | Durability |
|-----------|-------|---------|------------|
| ObjectRecognition | `/perception/object_recognition/objects` | `autoware_perception_msgs::msg::PredictedObjects` | VOLATILE |

**Planning** (`planning.hpp`)

| Interface | Topic/Service | Message/Service Type | Durability |
|-----------|--------------|---------------------|------------|
| RouteState | `/planning/route_state` | `RouteState` | TRANSIENT_LOCAL |
| LaneletRoute | `/planning/route` | `LaneletRoute` | TRANSIENT_LOCAL |
| Trajectory | `/planning/trajectory` | `Trajectory` | VOLATILE |
| SetLaneletRoute | `/planning/set_lanelet_route` | `srv::SetLaneletRoute` | — |
| SetWaypointRoute | `/planning/set_waypoint_route` | `srv::SetWaypointRoute` | — |
| ClearRoute | `/planning/clear_route` | `srv::ClearRoute` | — |

**Control** (`control.hpp`)

| Interface | Topic | Message | Durability |
|-----------|-------|---------|------------|
| ControlCommand | `/control/command/control_cmd` | `autoware_control_msgs::msg::Control` | VOLATILE |

**Localization** (`localization.hpp`)

| Interface | Topic/Service | Message/Service Type | Durability |
|-----------|--------------|---------------------|------------|
| KinematicState | `/localization/kinematic_state` | `nav_msgs::msg::Odometry` | VOLATILE |
| Acceleration | `/localization/acceleration` | `geometry_msgs::msg::AccelWithCovarianceStamped` | VOLATILE |
| InitializationState | `/localization/initialization_state` | `LocalizationInitializationState` | TRANSIENT_LOCAL |
| Initialize | `/localization/initialize` | `srv::InitializeLocalization` | — |

**Map** (`map.hpp`)

| Interface | Topic | Message | Durability |
|-----------|-------|---------|------------|
| MapProjectorInfo | `/map/map_projector_info` | `MapProjectorInfo` | TRANSIENT_LOCAL |
| PointCloudMap | `/map/point_cloud_map` | `sensor_msgs::msg::PointCloud2` | TRANSIENT_LOCAL |
| VectorMap | `/map/vector_map` | `autoware_map_msgs::msg::LaneletMapBin` | TRANSIENT_LOCAL |

**Vehicle** (`vehicle.hpp`)

| Interface | Topic | Message | Durability |
|-----------|-------|---------|------------|
| SteeringStatus | `/vehicle/status/steering_status` | `SteeringReport` | VOLATILE |
| GearStatus | `/vehicle/status/gear_status` | `GearReport` | VOLATILE |
| TurnIndicatorStatus | `/vehicle/status/turn_indicators_status` | `TurnIndicatorsReport` | VOLATILE |
| HazardLightStatus | `/vehicle/status/hazard_lights_status` | `HazardLightsReport` | VOLATILE |

**System** (`system.hpp`)

| Interface | Topic/Service | Message/Service Type | Durability |
|-----------|--------------|---------------------|------------|
| OperationModeState | `/system/operation_mode/state` | `OperationModeState` | TRANSIENT_LOCAL |
| ChangeAutowareControl | `/system/operation_mode/change_autoware_control` | `srv::ChangeAutowareControl` | — |
| ChangeOperationMode | `/system/operation_mode/change_operation_mode` | `srv::ChangeOperationMode` | — |

**QoS rule:** All use RELIABLE, queue depth 1. VOLATILE for real-time streams, TRANSIENT_LOCAL for persistent/latched data.

## Agnocast Zero-Copy IPC

Agnocast is an rclcpp-compatible zero-copy middleware using a kernel module for shared memory management. It's optional but significantly reduces latency for large messages (point clouds, images).

**Architecture:** Kernel module manages shared memory (`/agnocast@pid`), heaphook library intercepts malloc/free via `LD_PRELOAD`.

**Bridge modes:**
- **Standard** (default) — 1 bridge per Agnocast process, easy setup
- **Performance** — 1 global bridge manager, lower overhead, requires pre-compiled plugins
- **Off** — Pure Agnocast, no ROS 2 bridging

**Integration:**
```cmake
find_package(agnocastlib REQUIRED)
ament_target_dependencies(target agnocastlib)
```
```xml
<env name="LD_PRELOAD" value="libagnocast_heaphook.so"/>
```

Supported: ROS 2 Humble/Jazzy, Ubuntu 22.04/24.04, Linux 5.x/6.x.

## Executor Types

- **SingleThreadedExecutor** — default, single thread for all callbacks
- **MultiThreadedExecutor** — thread pool
- **CallbackIsolatedExecutor** — dedicated thread per CallbackGroup, enables SCHED_FIFO/SCHED_DEADLINE real-time scheduling, requires cgroup v1
- **agnocast::CallbackIsolatedAgnocastExecutor** — handles both ROS 2 and Agnocast callbacks

## Repository Dependency Graph

```
autoware (meta)
├── core/autoware_core, autoware_msgs, autoware_cmake, autoware_utils
├── universe/autoware_universe
├── launcher/autoware_launch
├── middleware/agnocast, callback_isolated_executor
├── sensor_component/nebula (LiDAR driver)
└── external/modelzoo, ros2_socketcan
```
