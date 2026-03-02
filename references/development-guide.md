# Autoware Development Guide

## Table of Contents
- [CMakeLists.txt Template](#cmakeliststxt-template)
- [package.xml Template](#packagexml-template)
- [Node Implementation](#node-implementation)
- [Launch File Template](#launch-file-template)
- [Config YAML Template](#config-yaml-template)
- [InterProcessPollingSubscriber](#interprocesspollingsubscriber)
- [Naming Conventions](#naming-conventions)
- [Testing](#testing)

## CMakeLists.txt Template

```cmake
cmake_minimum_required(VERSION 3.14)
project(autoware_my_package)

find_package(autoware_cmake REQUIRED)
autoware_package()

ament_auto_add_library(${PROJECT_NAME} SHARED
  src/node.cpp
)

rclcpp_components_register_node(${PROJECT_NAME}
  PLUGIN "autoware::my_package::MyNode"
  EXECUTABLE my_node_exe
  EXECUTOR SingleThreadedExecutor
)

if(BUILD_TESTING)
  find_package(ament_cmake_gtest REQUIRED)
  ament_auto_add_gtest(test_my_package test/test_my_node.cpp)
  target_link_libraries(test_my_package ${PROJECT_NAME})
endif()

autoware_ament_auto_package(INSTALL_TO_SHARE config launch)
```

**What `autoware_package()` does:** sets C++17, adds `-Wall -Wextra -Wpedantic -Werror`, calls `ament_auto_find_build_dependencies()` to auto-resolve deps from package.xml, sets ROS distro compile definitions.

**What `autoware_ament_auto_package()` does:** exports deps, installs headers/libraries/executables, installs directories listed in `INSTALL_TO_SHARE`.

## package.xml Template

```xml
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd"
  schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
  <name>autoware_my_package</name>
  <version>0.1.0</version>
  <description>Brief description</description>
  <maintainer email="name@example.com">Your Name</maintainer>
  <license>Apache License 2.0</license>

  <buildtool_depend>ament_cmake_auto</buildtool_depend>
  <buildtool_depend>autoware_cmake</buildtool_depend>

  <depend>rclcpp</depend>
  <depend>rclcpp_components</depend>
  <depend>autoware_utils_rclcpp</depend>
  <!-- Add message/library deps here -->

  <test_depend>ament_cmake_gtest</test_depend>
  <test_depend>ament_lint_auto</test_depend>
  <test_depend>autoware_lint_common</test_depend>
  <test_depend>autoware_test_utils</test_depend>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

Dependencies from package.xml are auto-found by `autoware_package()`, so you rarely need extra `find_package()` calls in CMake.

## Node Implementation

### Header (`include/autoware/my_package/node.hpp`)

```cpp
#ifndef AUTOWARE__MY_PACKAGE__NODE_HPP_
#define AUTOWARE__MY_PACKAGE__NODE_HPP_

#include <autoware_utils_rclcpp/polling_subscriber.hpp>
#include <rclcpp/rclcpp.hpp>
#include <nav_msgs/msg/odometry.hpp>
#include <autoware_planning_msgs/msg/trajectory.hpp>

namespace autoware::my_package
{

class MyNode : public rclcpp::Node
{
public:
  explicit MyNode(const rclcpp::NodeOptions & node_options);

private:
  autoware_utils_rclcpp::InterProcessPollingSubscriber<nav_msgs::msg::Odometry>
    sub_odometry_{this, "~/input/odometry"};

  rclcpp::Publisher<autoware_planning_msgs::msg::Trajectory>::SharedPtr pub_trajectory_;
  rclcpp::TimerBase::SharedPtr timer_;
  const double example_param_;

  void on_timer();
  friend class MyNodeTest;  // Allow test access to private members
};

}  // namespace autoware::my_package
#endif
```

### Source (`src/node.cpp`)

```cpp
#include "autoware/my_package/node.hpp"
#include <rclcpp_components/register_node_macro.hpp>

namespace autoware::my_package
{

MyNode::MyNode(const rclcpp::NodeOptions & node_options)
: Node("my_node", node_options),
  pub_trajectory_(create_publisher<autoware_planning_msgs::msg::Trajectory>(
    "~/output/trajectory", rclcpp::QoS(1))),
  example_param_(declare_parameter<double>("example_param"))
{
  using namespace std::literals::chrono_literals;
  timer_ = rclcpp::create_timer(
    this, get_clock(), 100ms, std::bind(&MyNode::on_timer, this));
}

void MyNode::on_timer()
{
  const auto odom_ptr = sub_odometry_.take_data();
  if (!odom_ptr) return;

  const auto & odom = *odom_ptr;
  // ... processing logic ...
  // pub_trajectory_->publish(result);
}

}  // namespace autoware::my_package

RCLCPP_COMPONENTS_REGISTER_NODE(autoware::my_package::MyNode)
```

## Launch File Template

```xml
<launch>
  <arg name="param_file" default="$(find-pkg-share autoware_my_package)/config/my_package.param.yaml"/>

  <node pkg="autoware_my_package" exec="my_node_exe" name="my_node" output="screen">
    <param from="$(var param_file)"/>
    <remap from="~/input/odometry" to="/localization/kinematic_state"/>
    <remap from="~/output/trajectory" to="/planning/trajectory"/>
  </node>
</launch>
```

## Config YAML Template

```yaml
/**:
  ros__parameters:
    example_param: 1.0
    limits:
      max_velocity: 11.1    # [m/s]
      max_acceleration: 1.0  # [m/s^2]
```

## InterProcessPollingSubscriber

Decouples data consumption from callback execution — the subscriber buffers data and you poll it in your timer.

```cpp
// Latest (default) — always returns most recent message
autoware_utils_rclcpp::InterProcessPollingSubscriber<Odometry>
  sub_{this, "~/input/odometry"};

// Newest — returns data only if updated since last take_data()
autoware_utils_rclcpp::InterProcessPollingSubscriber<
  VelocityLimit, autoware_utils_rclcpp::polling_policy::Newest>
  sub_{this, "~/input/velocity_limit"};

// With custom QoS for latched topics
autoware_utils_rclcpp::InterProcessPollingSubscriber<OperationModeState>
  sub_{this, "~/input/operation_mode", rclcpp::QoS{1}.transient_local()};

// In timer callback:
const auto ptr = sub_.take_data();
if (!ptr) return;  // No data yet
```

**Event-driven alternative** — use a regular subscription callback as the trigger, poll background data inside it:

```cpp
sub_trigger_ = create_subscription<PathWithLaneId>(
  "~/input/path", 1, std::bind(&MyNode::onTrigger, this, std::placeholders::_1));
```

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Package | `autoware_` + snake_case | `autoware_velocity_smoother` |
| Node name | snake_case | `velocity_smoother` |
| Class | PascalCase + `Node` | `VelocitySmootherNode` |
| Namespace | `autoware::pkg_name` | `autoware::velocity_smoother` |
| Input topics | `~/input/<name>` | `~/input/odometry` |
| Output topics | `~/output/<name>` | `~/output/trajectory` |
| Config files | `<name>.param.yaml` | `velocity_smoother.param.yaml` |
| Launch files | `<name>.launch.xml` | `velocity_smoother.launch.xml` |

## Testing

### Unit Test (gtest)

```cpp
#include <gtest/gtest.h>
#include <autoware_test_utils/autoware_test_utils.hpp>
#include "autoware/my_package/node.hpp"

class MyNodeTest : public ::testing::Test {
protected:
  void SetUp() override {
    rclcpp::init(0, nullptr);
    auto opts = rclcpp::NodeOptions{};
    autoware::test_utils::updateNodeOptions(opts,
      {autoware::test_utils::get_absolute_path_to_config(
        "autoware_my_package", "test.param.yaml")});
    node_ = std::make_shared<autoware::my_package::MyNode>(opts);
  }
  void TearDown() override { rclcpp::shutdown(); }
  std::shared_ptr<autoware::my_package::MyNode> node_;
};

TEST_F(MyNodeTest, Initialization) { EXPECT_NE(node_, nullptr); }
```

### Key autoware_test_utils helpers

```cpp
createPose(x, y, z, roll, pitch, yaw)    // Create geometry_msgs Pose
makeOdometry(shift)                        // Create nav_msgs Odometry
generateTrajectory<T>(n, interval, vel)    // Generate trajectory with n points
makeMapBinMsg(pkg, filename)               // Load lanelet2 map
updateNodeOptions(opts, param_files)       // Load YAML params into NodeOptions
```

`AutowareTestManager` handles pub/sub and clock in integration tests — call `test_pub_msg()` to publish and `set_subscriber()` to capture output.

### CMake integration

```cmake
# Basic gtest
ament_auto_add_gtest(test_name test/test_file.cpp)

# Isolated (needs ROS middleware)
ament_add_ros_isolated_gtest(test_name test/test_file.cpp)

# Launch test (Python)
add_launch_test(test/test_launch.py TIMEOUT "30")
```

### Common pitfalls

- Call `rclcpp::init` in SetUp and `rclcpp::shutdown` in TearDown — don't call init/shutdown multiple times in separate TEST() blocks
- Spin after publishing to let messages propagate before asserting
- Use `friend class` in node header to access private members from test fixtures
