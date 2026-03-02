# Autoware Debugging Guide

## Table of Contents
- [Build Failures](#build-failures)
- [Runtime Errors](#runtime-errors)
- [Perception Issues](#perception-issues)
- [Planning Issues](#planning-issues)
- [Localization Issues](#localization-issues)
- [System Issues](#system-issues)
- [Diagnostic Commands](#diagnostic-commands)

## Build Failures

### Dependency Version Conflicts

**Symptom:** CMake "package NOT found" or linker errors despite packages being installed.

**Fix:**
- Re-sync repos: `vcs import src < autoware.repos`
- Install missing deps: `rosdep install -y --from-paths src --ignore-src --rosdistro $ROS_DISTRO`
- Clear stale artifacts: `rm -rf build/ install/ log/` and rebuild

### CUDA / TensorRT / cuDNN

**Symptom:** "CUDA::cudart target not found", "TensorRT variables not found", "spconv NOT available"

**Fix:**
1. Verify installed: `nvcc --version`, `dpkg -l | grep tensorrt`
2. Set paths: `export CMAKE_PREFIX_PATH=/usr/local/cuda:$CMAKE_PREFIX_PATH`
3. WSL2: `sudo apt install -y linux-headers-generic`
4. Skip if not needed: `colcon build --packages-ignore autoware_tensorrt_plugins`
5. NVIDIA 50-series GPUs may need latest TensorRT for Blackwell architecture

### Jazzy vs Humble Incompatibilities

**Symptom:** Compilation errors after switching ROS distro.

Check `#ifdef ROS_DISTRO_JAZZY` blocks in source. Ensure all `autoware.repos` packages have compatible versions.

### pip PEP 668

**Symptom:** "externally-managed-environment" error during build.

pip >= 23.0 blocks system-wide installs. Use `--break-system-packages` in Docker, or use venvs.

### Docker Build Failures

- GPG key expired → Update ROS 2 GPG keys in Dockerfile
- No space left → `docker system prune -af`
- Layer too large → Use multi-stage builds

## Runtime Errors

### Topic Not Connecting

**Symptom:** Subscriber receives no data.

**Causes:**
- **QoS mismatch** — publisher BEST_EFFORT vs subscriber RELIABLE. Check with `ros2 topic info /topic -v`. Use `autoware_component_interface_specs` for standardized QoS.
- **Wrong topic name** — check launch file `<remap>` entries and `~/` prefix expansion
- **Different DDS domains** — ensure matching `ROS_DOMAIN_ID`

### Node Crash on Startup

**Causes:**
- **Missing parameters** — "Statically typed parameter must be initialized". Ensure YAML loaded in launch file.
- **TF timeout** — check tree: `ros2 run tf2_tools view_frames`
- **Missing map data** — ensure map loading completes before dependent nodes start

### DDS Domain Participant Limit

**Symptom:** "rmw_create_node: failed to create domain"

CycloneDDS default limit (~50) exceeded. Fix in `cyclonedds.xml`:
```xml
<Domain Id="any">
  <Discovery>
    <ParticipantIndex>auto</ParticipantIndex>
    <MaxAutoParticipantIndex>1000</MaxAutoParticipantIndex>
  </Discovery>
</Domain>
```

### Excessive Callback Frequencies

CPU spikes, unresponsiveness. Check `ros2 topic hz`. Common causes: timer too fast, process respawn loop creating DDS storms, unbounded callback rates.

## Perception Issues

### Empty Point Cloud Crashes

Missing empty data guards in many perception packages. Always validate:
```cpp
if (msg->data.empty()) {
  RCLCPP_WARN(get_logger(), "Empty point cloud, skipping");
  return;
}
```

### ML Model / TensorRT

- **Engine mismatch** — TensorRT engines are GPU-architecture-specific, rebuild on target
- **Model version mismatch** — update from modelzoo
- **GPU not available in Docker** — add `--gpus all`
- **CUDA API changes** — newer CUDA versions deprecate old APIs, add conditional includes

### QoS Mismatch for Sensor Data

RViz can't visualize point clouds when publisher uses BEST_EFFORT. Align QoS or use component_interface_specs.

## Planning Issues

### Empty Trajectory

Control node receives empty trajectory → vehicle stops. Causes: no valid path, route not set, map not loaded. Add trajectory validation in control; publish previous valid trajectory as fallback.

### Lane Change Failures

Fails initially, succeeds after reversing. Feasibility check constraints too conservative for current speed — review lateral acceleration limits.

### Non-Monotonic Trajectory

Freespace planner generates backwards segments. Add monotonicity validation and post-processing.

## Localization Issues

### NDT Scan Matcher Divergence

**Symptom:** Localization jumps, huge latency (500+ ms), "No InputSource" warning.

**Causes:** pipeline latency in crop_box_filter/downsampler, point cloud too sparse, initial pose too far from truth. Monitor per-stage latency, check voxel sizes.

### EKF Drift

Pose estimate drifts or oscillates. Check sensor noise parameters, IMU bias calibration, and whether all inputs (gyro, wheel odometry) are connected.

### Pose Initializer Failures

Components die after setting init pose. Race condition in state transitions — add defensive checks, ensure initialization service completes before processing.

### GNSS Offset

Eagleye produces position offset. Verify antenna calibration and map projection settings.

## System Issues

### Memory Reporting Bug

`autoware_component_monitor` reports memory ~1024x too small. `top -b -n 1 -E k` outputs KiB but parser treats as bytes. Fix: multiply suffix-less values by 1024.

### GPU Monitor

No metrics on Jetson (different nvidia-smi API, use `tegrastats`). False warnings when GPU idle (threshold logic doesn't handle idle state).

### Deprecated glog Warnings

Migrate to `RCLCPP_INFO`, `RCLCPP_WARN`, etc.

## Diagnostic Commands

```bash
# Topics
ros2 topic list / info -v / echo / hz / bw / delay

# Nodes
ros2 node list / info

# Parameters
ros2 param list / get / set

# TF
ros2 run tf2_tools view_frames
ros2 run tf2_ros tf2_echo frame1 frame2

# Build
colcon build --packages-select <pkg> --cmake-args -DCMAKE_BUILD_TYPE=Debug
rosdep check --from-paths src --ignore-src

# System
ros2 doctor
```
