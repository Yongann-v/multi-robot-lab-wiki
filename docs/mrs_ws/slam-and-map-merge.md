# Map the site with both robots (SLAM + map merge)

Part of [mrs_ws](https://github.com/GSU-CARL/mrs_ws), the lab's Warthog + Husky multi-robot inspection system (ROS 2 Jazzy, Gazebo Harmonic). The source of this page is `docs/tutorials/02-slam-and-map-merge.md` in that repo.

You will launch the Warthog (`w200_0111`) and the Husky (`a200_1103`) in Gazebo with SLAM running on each, open RViz on the team map, and drive both robots until one merged map covers the arena and the building.

Time: about 10 min. Do [Run the two-robot simulation](run-simulation.md) first: it builds the image and the workspace and explains the `docker run` flags.

## What runs

`multi_sim.launch.py` now starts, on top of tutorial 01:

| Node | Namespace | Does |
| --- | --- | --- |
| `pointcloud_to_laserscan` | `/<ns>` | Lidar cloud `sensors/lidar3d_0/points` -> planar `scan`, a 0.2-1.8 m slice above `base_link` |
| `slam_toolbox` (online async) | `/<ns>` | `scan` -> robot map `/<ns>/map`, and `map -> odom` on the robot's own `/<ns>/tf` |
| `tf_relay` | team | Every `/<ns>/tf` -> team `/tf` with `<ns>/` frame prefixes |
| `map_merger` | team | `map -> <ns>/map` from each robot's start pose, and the merged team map `/map` |

Team TF tree: `map` -> `<ns>/map` -> `<ns>/odom` -> `<ns>/base_link`.

The start poses are the `spawn` entries in `src/mrs_description/config/fleet.yaml`: Warthog at (0, 0), Husky at (0, 3), both facing +x. The merger places each robot's map at its start pose, so the two maps line up without any matching step. On site, the robots must start at the poses listed there.

## 1. Rebuild the image and the workspace

Stage 1b added `pointcloud_to_laserscan` to the image. Rebuild it once, then rebuild the workspace:

```bash
cd ~/mrs_ws
docker build -f docker/Dockerfile.dev -t mrs_ws:dev docker
docker run --rm -u $(id -u):$(id -g) -e HOME=/tmp/h \
  -v ~/mrs_ws:/ws -w /ws mrs_ws:dev \
  bash -c 'source /opt/ros/jazzy/setup.bash && colcon build --symlink-install'
```

## 2. Start the simulation (terminal 1)

Same command as tutorial 01:

```bash
xhost +local:docker
docker run -it --rm --name mrs_sim --net=host --ipc=host \
  -u $(id -u):$(id -g) -e HOME=/tmp/h \
  -e DISPLAY=$DISPLAY -e LIBGL_ALWAYS_SOFTWARE=1 -e ROS_DOMAIN_ID=42 \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v ~/mrs_ws:/ws -w /ws mrs_ws:dev \
  bash -c 'source install/setup.bash && ros2 launch mrs_bringup multi_sim.launch.py'
```

Wait until both robots have spawned and both SLAM nodes report `Activating` (about 15 s):

```text
[w200_0111.slam_toolbox]: Activating
[a200_1103.slam_toolbox]: Activating
[map_merger]: merging maps of ['w200_0111', 'a200_1103'] into 'map'
```

## 3. Open RViz (terminal 2)

```bash
docker exec -it mrs_sim bash -c \
  'source install/setup.bash && rviz2 -d install/mrs_bringup/share/mrs_bringup/rviz/team_slam.rviz --ros-args -p use_sim_time:=true'
```

The view is top-down in the team `map` frame:

| Display | Shows |
| --- | --- |
| Team map (`/map`) | The merged map, grey = unknown, white = free, black = occupied |
| Warthog scan (orange), Husky scan (blue) | The planar scans SLAM uses |
| TF | `map`, `<ns>/map` and `<ns>/base_link` with names |
| Warthog map, Husky map | Each robot's own map. Off by default: tick one to compare it with `/map` |

Right after startup each robot has mapped only what it sees from its start pose, a rough disc around it.

## 4. Drive and watch the map grow (terminals 3 and 4)

Drive each robot with the keyboard. Clearpath robots take `TwistStamped` on `/<ns>/cmd_vel`, hence `stamped:=true`:

```bash
# terminal 3: Warthog
docker exec -it mrs_sim bash -c 'source install/setup.bash && \
  ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args \
  -r cmd_vel:=/w200_0111/cmd_vel -p stamped:=true -p frame_id:=base_link'

# terminal 4: Husky
docker exec -it mrs_sim bash -c 'source install/setup.bash && \
  ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args \
  -r cmd_vel:=/a200_1103/cmd_vel -p stamped:=true -p frame_id:=base_link'
```

Keys: `i` forward, `j`/`l` turn, `k` stop, `z` slower. Drive at or below 0.5 m/s. Each robot's collision monitor slows and stops it before it touches an obstacle; reverse or turn away to continue.

A route that covers the site without the robots meeting:

- **Warthog:** a loop around the outdoor arena (x -12..10, y -10..14). Keep about 2 m from the crates, columns and the barrier.
- **Husky:** east to the building, through the door at x = 10, y = 0..3, then along the corridor (y = 1.5) to x = 28 and back.

The team map updates every 2 s. The building's west wall from the Husky's map should land exactly on the arena's east edge from the Warthog's map.

## 5. Check the result (any terminal)

```bash
docker exec -it mrs_sim bash
source install/setup.bash && export ROS_DOMAIN_ID=42
```

Team map size and origin:

```bash
ros2 topic echo --once --qos-durability transient_local --qos-reliability reliable /map --field info
```

Robot maps stay in their namespaces, and nothing else publishes `/map`:

```bash
ros2 topic info /map    # Publisher count: 1 (map_merger)
```

SLAM pose against Gazebo's true pose. They should agree within about 0.1 m and a few degrees:

```bash
ros2 run tf2_ros tf2_echo map a200_1103/base_link --ros-args -p use_sim_time:=true
gz model -m a200_1103/robot -p
```

Save the merged map to `maps/team.pgm` + `.yaml`:

```bash
mkdir -p /ws/maps && ros2 run nav2_map_server map_saver_cli -f /ws/maps/team \
  --ros-args -p map_subscribe_transient_local:=true
```

Leave out `use_sim_time` here: with it, the saver's 2 s timeout runs on the sim clock and it fails with `Failed to spin map subscription`.

`maps/` is git-ignored. Do not commit maps.

## 6. Stop

Press Ctrl+C in terminal 1, or run `docker rm -f mrs_sim`. RViz and the teleop terminals close with the container.

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| A rotated "ghost" copy of the arena, or a shifted copy, appears in a robot's map | The odometry under SLAM was wrong. Skid-steer wheels slip in turns and when pushing against an obstacle, so wheel odometry heading and distance drift. The Warthog's sim EKF takes heading from its IMU, and the collision monitor stops robots before contact. If you still see it, check `ros2 lifecycle get /<ns>/collision_monitor` (should be `active`) and that you are not driving with the joystick, which bypasses the monitor. Restart the sim to clear the map. `ros2 run mrs_sim heading_check` measures the drift against Gazebo truth (see `src/mrs_sim/README.md`). |
| `/<ns>/map` never appears | SLAM gets no scans or no TF. Check `ros2 topic hz /<ns>/scan` (about 10-15 Hz) and the terminal 1 log for `Message Filter dropping message`. A few at startup are normal. |
| `/map` exists but stays grey | The merger has no robot maps yet. Check `ros2 topic list | grep /map`, which should list `/map`, `/w200_0111/map` and `/a200_1103/map`. |
| `ros2 topic echo /map` prints nothing | Maps are latched (transient local, reliable). Add `--qos-durability transient_local --qos-reliability reliable`. |
| RViz shows `Frame [map] does not exist` | The merger starts with the sim and publishes `map -> <ns>/map` at once. Check that `use_sim_time:=true` was passed to RViz. |
| The robot does not move with teleop | You forgot `-p stamped:=true`: Clearpath's twist_mux ignores plain `Twist`. Or the collision monitor is not running: `cmd_vel` only reaches the robot through it (as `cmd_vel_safe`). Check `ros2 lifecycle get /<ns>/collision_monitor`. Or the robot is already at an obstacle: drive away from it. |
| Robot maps line up badly in `/map` | A `spawn` pose in fleet.yaml does not match where the robot actually started. |

## Where things live (in mrs_ws)

| Path | What |
| --- | --- |
| `src/mrs_slam/launch/robot_slam.launch.py` | Per-robot SLAM, same file for sim and hardware |
| `src/mrs_slam/config/slam_toolbox.yaml` | slam_toolbox settings (`map_name: map` keeps each map in its namespace) |
| `src/mrs_slam/config/pointcloud_to_laserscan.yaml` | Height band and range of the planar scan |
| `src/mrs_slam/mrs_slam/map_merger.py`, `merge.py` | Team map merger and its grid math ([details](https://github.com/GSU-CARL/mrs_ws/blob/main/src/mrs_slam/README.md)) |
| `src/mrs_description/config/fleet.yaml` | Start poses used by the merger |
| `src/mrs_navigation/config/collision_monitor.yaml`, `footprint_<platform>.yaml` | Collision monitor between `cmd_vel` and the robot |
| `src/mrs_bringup/rviz/team_slam.rviz` | The RViz view used here |
