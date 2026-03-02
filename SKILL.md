---
name: autoware-dev
description: >-
  Assists with Autoware autonomous driving development, debugging, and navigation.
  Use this skill whenever the user works with Autoware, ROS 2 autonomous driving,
  perception/planning/control/localization modules, colcon builds, ament_cmake,
  autoware_cmake, launch files, component interfaces, agnocast, NDT scan matcher,
  EKF localizer, behavior velocity planner, trajectory planning, point cloud processing,
  TensorRT inference, lanelet2 maps, autoware_core, autoware_universe, or autoware_msgs.
  Even if the user doesn't say "Autoware" explicitly, trigger when the context involves
  ROS 2 autonomous driving packages with the autoware_ prefix.
---

# Autoware Development & Debugging

## Quick Start

| Goal | Go to |
|------|-------|
| Create a new package | [Creating a Package](#creating-a-package) then [references/development-guide.md](references/development-guide.md) for full templates |
| Debug a build or runtime issue | [Debugging Quick Reference](#debugging-quick-reference) then [references/debugging-guide.md](references/debugging-guide.md) for details |
| Understand the system architecture | [Architecture](#architecture) then [references/architecture.md](references/architecture.md) for deep dive |
| Implement or modify a node | [references/development-guide.md](references/development-guide.md) |
| Write tests | [references/development-guide.md § Testing](references/development-guide.md#testing) |

## Architecture

Autoware is a three-tier autonomous driving stack on ROS 2:

- **autoware** (meta-repo) — workspace definition, version pinning via `.repos` files
- **autoware_core** — stable, production-quality packages
- **autoware_universe** — experimental, community-contributed packages
- **agnocast** (optional) — zero-copy IPC middleware with kernel-level shared memory

Data flows through a pipeline: **Sensing → Perception → Planning → Control**, with Localization and Map feeding into the pipeline at multiple points.

Key message types between stages:

| From → To | Message | Topic |
|-----------|---------|-------|
| Perception → Planning | `PredictedObjects` | `/perception/object_recognition/objects` |
| Planning → Control | `Trajectory` | `/planning/trajectory` |
| Control → Vehicle | `Control` | `/control/command/control_cmd` |
| Localization → All | `Odometry` | `/localization/kinematic_state` |
| Map → Planning | `LaneletMapBin` | `/map/vector_map` |

All inter-module interfaces are defined in `autoware_component_interface_specs` with standardized topic names, message types, and QoS. VOLATILE for real-time streams, TRANSIENT_LOCAL for persistent data. See [references/architecture.md](references/architecture.md) for the full interface table, module breakdown, Agnocast details, and dependency graph.

## Creating a Package

Every Autoware package follows this structure. For copy-paste templates, see [references/development-guide.md](references/development-guide.md).

**Checklist:**

1. **CMakeLists.txt** — `find_package(autoware_cmake REQUIRED)` then `autoware_package()` which sets C++17, `-Werror`, and auto-finds deps from package.xml
2. **package.xml** — Format 3, always include `ament_cmake_auto` and `autoware_cmake` as buildtool deps
3. **Node class** — Inherit `rclcpp::Node`, use `InterProcessPollingSubscriber` for decoupled input polling, timer-driven processing
4. **Component registration** — `RCLCPP_COMPONENTS_REGISTER_NODE()` at end of .cpp, plus `rclcpp_components_register_node()` in CMake
5. **Launch file** (XML) — Load param YAML with `<param from="..."/>`, remap `~/input/*` and `~/output/*` topics
6. **Config YAML** — Under `/**:` → `ros__parameters:`
7. **Tests** — `ament_auto_add_gtest` in CMake, use `autoware_test_utils` helpers

**Naming:** packages `autoware_*`, topics `~/input/*` and `~/output/*`, classes `PascalCaseNode`, namespaces `autoware::package_name`

## Debugging Quick Reference

| Symptom | Likely Cause | First Step |
|---------|-------------|------------|
| CMake "package NOT found" | Missing dep or stale build | `rosdep install --from-paths src` |
| "CUDA::cudart not found" | CUDA/TensorRT path issue | Check `nvcc --version`, CMAKE_PREFIX_PATH |
| Topic not receiving data | QoS mismatch | `ros2 topic info /topic -v` |
| "parameter must be initialized" | Missing param in config | Check launch file loads YAML |
| "failed to create domain" | DDS participant limit | Set `MaxAutoParticipantIndex=1000` in cyclonedds.xml |
| Node crashes on empty input | Missing null guard | Add `if (msg->data.empty()) return;` |
| NDT huge latency | Pipeline bottleneck | Check per-stage timestamps |
| TensorRT engine fails | GPU arch mismatch | Rebuild engine on target hardware |

**Key diagnostic commands:**
```bash
ros2 topic list / info -v / hz / echo    # Topic debugging
ros2 node info /node_name                 # Node connections
ros2 param list /node_name                # Parameters
ros2 run tf2_tools view_frames            # TF tree
colcon build --packages-select <pkg>      # Rebuild one package
rosdep check --from-paths src             # Dependency check
```

For detailed diagnosis by category (build, runtime, perception, planning, localization, system), see [references/debugging-guide.md](references/debugging-guide.md).

## Conventions

- **C++17** enforced by `autoware_cmake`, compiler flags: `-Wall -Wextra -Wpedantic -Werror`
- **Formatting:** clang-format, clang-tidy, `autoware_lint_common`
- **License:** Apache 2.0
- **Commits:** `feat:`, `fix:`, `refactor:`, `docs:`, `test:` prefixes
- **Build:** `colcon build --symlink-install`, test with `colcon test --packages-select <pkg>`
