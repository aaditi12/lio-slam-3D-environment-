# liThis is a **ROS 2 Humble + Gazebo Classic 11** demo comparing localization/SLAM algorithms: **Nav2 AMCL** (working, apt-installable) vs **LIO-SAM** (documented but requires separate manual setup — the bundled robot's 2D LiDAR isn't sufficient for it). Here's how to run it.

## Setup

```bash
sudo apt install tmux \
  ros-humble-nav2-amcl \
  ros-humble-nav2-map-server \
  ros-humble-nav2-lifecycle-manager \
  ros-humble-nav2-bringup \
  ros-humble-slam-toolbox \
  ros-humble-teleop-twist-keyboard \
  ros-humble-gazebo-ros-pkgs \
  ros-humble-xacro
```

## Build

```bash
cd amcl_liosam_ws
colcon build --packages-select amcl_nav_demo --symlink-install
source install/setup.bash
```

## Easiest way — use the tmux demo script

**Phase 1 — build a map first (slam_toolbox):**
```bash
bash run_amcl_demo.sh map
```
This opens a tmux session with windows: Gazebo, SLAM (slam_toolbox), RViz, Teleop (drive around with `i/,/j/l/k`), and a SaveMap window. Once the map looks good in RViz, go to the SaveMap window and run:
```bash
ros2 launch amcl_nav_demo map_saver.launch.py
```
Then rebuild so the saved map gets installed:
```bash
colcon build --packages-select amcl_nav_demo --symlink-install
```

**Phase 2 — localize on the saved map (Nav2 AMCL):**
```bash
bash run_amcl_demo.sh localize
```
Opens tmux windows: Gazebo, AMCL, RViz, Teleop, and MapSub. Drive the robot and watch the particle cloud converge in RViz.

```bash
# Attach to the running session anytime:
tmux attach -t amcl_demo
# Kill it:
tmux kill-session -t amcl_demo && killall -9 gzserver gzclient
```

## Manual, per-component launch (without the tmux script)

```bash
ros2 launch amcl_nav_demo gazebo_spawn.launch.py      # Gazebo + robot spawn
ros2 launch slam_toolbox online_async_launch.py use_sim_time:=true \
  slam_params_file:=/opt/ros/humble/share/slam_toolbox/config/mapper_params_online_async.yaml   # mapping
ros2 launch amcl_nav_demo rviz.launch.py              # RViz
ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args --remap cmd_vel:=/cmd_vel   # drive
ros2 launch amcl_nav_demo map_saver.launch.py         # save map
ros2 launch amcl_nav_demo amcl.launch.py              # AMCL localization on saved map
ros2 run amcl_nav_demo map_subscriber.py              # map subscriber utility
```

## Fixing AMCL if it diverges

```bash
# Via RViz: click "2D Pose Estimate" tool and click the robot's real position
# Or via CLI:
ros2 topic pub /initialpose geometry_msgs/msg/PoseWithCovarianceStamped \
  "{ header: {frame_id: map}, pose: { pose: { position: {x: -1.8, y: 3.0} } } }" --once
```

## If you actually want LIO-SAM (3D LiDAR+IMU SLAM)

The bundled robot doesn't support it out of the box — README notes you'd need to add a 3D LiDAR (Velodyne/Ouster) and IMU plugin to the URDF first, then:

```bash
sudo add-apt-repository ppa:borglab/gtsam-release-4.1
sudo apt update && sudo apt install libgtsam-dev libgtsam-unstable-dev

mkdir -p ~/lio_sam_ws/src && cd ~/lio_sam_ws/src
git clone https://github.com/TixiaoShan/LIO-SAM.git
cd ~/lio_sam_ws
colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=Release

source ~/lio_sam_ws/install/setup.bash
ros2 launch lio_sam run.launch.py
```

Note: needs a real Ubuntu 22.04 + ROS 2 Humble + Gazebo machine (ideally with GPU) — won't run in this sandbox.o-slam-3D-environment-
ros2 run nav2_map_server map_saver_cli -f ~/maps/my_map     ros2 launch slam_toolbox online_async_launch.py
