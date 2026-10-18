# Send the robots to goals (Nav2)

Part of [mrs_ws](https://github.com/GSU-CARL/mrs_ws), the lab's Warthog + Husky multi-robot inspection system (ROS 2 Jazzy, Gazebo Harmonic). The source of this page is `docs/tutorials/03-navigation.md` in that repo.

You will launch both robots with Nav2 running on each, click a goal for each robot in the team map, and watch them plan and drive there on their live SLAM maps, while avoiding obstacles and each other.

Time: about 10 min. Do [Map the site with both robots](slam-and-map-merge.md) first.

## What runs

`multi_sim.launch.py` now also starts, per robot under `/<ns>`, on the robot's own `/<ns>/tf`:

| Node | Does |
| --- | --- |
| `planner_server` (Smac 2D) | Plans on the global costmap: the robot's SLAM `map` + `scan` and `teammates` obstacles + keepout zones. Unknown space counts as free, so robots can plan into areas they have not mapped yet |
| `controller_server` (Regulated Pure Pursuit) | Follows the plan on a 6 m local costmap around the robot. It turns in place toward the path first (skid-steer) |
| `behavior_server` | Recoveries: spin, back up, wait |
| `bt_navigator` | `navigate_to_pose` and `navigate_through_poses` actions (stock behavior trees with replanning and recovery) |
| `waypoint_follower` | `follow_waypoints` action (used later by routes) |
| `velocity_smoother` | Limits speed and acceleration per platform |
| `lifecycle_manager_navigation` | Starts the nodes above |

Two more per-robot nodes read the **team** TF tree on `/tf`, because they need where the other robots are, or where this robot started:

| Node | Does |
| --- | --- |
| `teammate_cloud` | Publishes every other robot's footprint (+0.1 m) as a point cloud on `/<ns>/teammates`, 10 Hz, in the robot's `base_link`. Both costmaps and the collision monitor treat it as obstacles |
| `keepout_mask` | Turns the site's keepout zones for this robot's platform into a Nav2 filter mask (`/<ns>/keepout_mask`), moved into the robot's own map. Both costmaps make the zones lethal |

And in the team layer:

| Node | Does |
| --- | --- |
| `goal_relay` | Takes a goal in the **team** map on `/<ns>/team_goal`, converts it to that robot's own map, and sends it to `/<ns>/navigate_to_pose` |

Every velocity command still goes through the collision monitor:

```text
controller / behaviors -> cmd_vel_nav -> velocity_smoother -> cmd_vel -> collision_monitor -> cmd_vel_safe -> robot
```

Speeds: Husky up to 0.5 m/s, Warthog up to 0.8 m/s (`src/mrs_navigation/config/nav2_<platform>.yaml`).

**Why teammate footprints?** The lidar can miss a low teammate up close. The Warthog's scan slice starts 0.49 m above the ground and its lowest beam passes over the 0.36 m Husky closer than about 2.6 m, so before `teammate_cloud` the Warthog drove into the Husky and pushed it. With it, each robot knows where the other one is from the team TF, even when its lidar sees nothing.

**Why keepout zones?** The Warthog (1.42 m wide) cannot pass the Husky in the 2.75 m corridor and does not fit the 1.2 m room doors. So the Warthog (scout) stays outdoors and the Husky (inspector) works the building alone. The zones are per site, in team-map coordinates, each listing the platforms it keeps out: `mrs_test_site` has one, `building` (x 10..30, y -10..14), for `w200`. The Husky has no zones.

**Why the relay?** Each robot's Nav2 works in that robot's own `map` frame, the map its SLAM started at its spawn pose. That frame is called `map` on the robot, the same name as the team map, but the two are different frames: for a robot that did not start at the team origin, the same coordinates are different places. A goal clicked in the team view must be converted first, and `goal_relay` does that through the team TF tree.

## 1. Rebuild the workspace

```bash
cd ~/mrs_ws
docker run --rm -u $(id -u):$(id -g) -e HOME=/tmp/h \
  -v ~/mrs_ws:/ws -w /ws mrs_ws:dev \
  bash -c 'source /opt/ros/jazzy/setup.bash && colcon build --symlink-install'
```

## 2. Start the simulation (terminal 1)

Same command as the previous two tutorials:

```bash
xhost +local:docker
docker run -it --rm --name mrs_sim --net=host --ipc=host \
  -u $(id -u):$(id -g) -e HOME=/tmp/h \
  -e DISPLAY=$DISPLAY -e LIBGL_ALWAYS_SOFTWARE=1 -e ROS_DOMAIN_ID=42 \
  -e RMW_IMPLEMENTATION=rmw_zenoh_cpp \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v ~/mrs_ws:/ws -w /ws mrs_ws:dev \
  bash -c 'source install/setup.bash && ros2 launch mrs_bringup multi_sim.launch.py'
```

Wait until Nav2 is up on both robots (about 20 s after launch). Look for this line twice:

```text
[<ns>.lifecycle_manager_navigation]: Managed nodes are active
```

## 3. Team view: send goals (terminal 2)

```bash
docker exec -it mrs_sim bash -c \
  'source install/setup.bash && rviz2 -d install/mrs_bringup/share/mrs_bringup/rviz/team_slam.rviz --ros-args -p use_sim_time:=true'
```

The toolbar has two **2D Goal Pose** buttons: the first sends to the Warthog (`/w200_0111/team_goal`), the second to the Husky (`/a200_1103/team_goal`). Hover a button to see its topic. Click a button, then click and drag on the map: the click sets the position, the drag sets the heading.

Try:

- **Husky:** along the corridor, e.g. x = 15, y = 1.5. It drives 12 m west from its start at the east end.
- **Warthog:** across the arena, e.g. x = -5, y = 10.

Terminal 1 shows the relay's progress:

```text
[goal_relay]: a200_1103: goal (15.00, 1.50) in map -> (12.00, 0.00, yaw -3.14) in its map
[goal_relay]: a200_1103: 8.4 m to go, 0 recoveries
[goal_relay]: a200_1103: goal succeeded
```

The Husky started at (27, 1.5) facing west, so team (15, 1.5) is 12 m straight ahead, (12, 0), in its own map, and a goal facing east (yaw 0 in the team map) is yaw -3.14 in its map.

A new goal for a robot replaces its current one. Warthog goals inside the building (x 10..30, y -10..14) fail by design: it is a keepout zone for the Warthog (see *Why keepout zones?* above).

## 4. Robot view: costmaps and plan (terminal 3, optional)

To see one robot's Nav2 (its SLAM map, global and local costmaps, plan and footprint), run RViz inside that robot's namespace, on its own TF tree:

```bash
docker exec -it mrs_sim bash -c 'source install/setup.bash && \
  rviz2 -d install/mrs_navigation/share/mrs_navigation/rviz/robot_nav.rviz --ros-args \
  -r __ns:=/a200_1103 -r /tf:=tf -r /tf_static:=tf_static -p use_sim_time:=true'
```

What to look for:

- **Teammate:** the other robot shows as lethal cells (its footprint plus a margin) in both costmaps, even right next to this robot where the lidar misses it. It moves as the teammate drives.
- **Keepout:** in the Warthog's view (`__ns:=/w200_0111`), the whole building is lethal in both costmaps. In the Husky's view there is no keepout.

Check the teammate cloud from a shell: the Warthog's holds the Husky's footprint (475 points), the Husky's the Warthog's (1224 points). An empty cloud (0) means the team TF of the other robot is missing:

```bash
ros2 topic echo --once /w200_0111/teammates sensor_msgs/msg/PointCloud2 --field width
```

Everything there is in the robot's own frames. Its **2D Goal Pose** button sends straight to that robot's Nav2 (`/<ns>/goal_pose`), in the robot's map, with no conversion.

## 5. Send goals from the command line

```bash
docker exec -it mrs_sim bash
source install/setup.bash && export ROS_DOMAIN_ID=42
ros2 topic pub --once /a200_1103/team_goal geometry_msgs/msg/PoseStamped \
  "{header: {frame_id: map}, pose: {position: {x: 15.0, y: 1.5}, orientation: {w: 1.0}}}"
```

Or skip the relay and call a robot's Nav2 action directly, in that robot's own map frame:

```bash
ros2 action send_goal --feedback /w200_0111/navigate_to_pose nav2_msgs/action/NavigateToPose \
  "{pose: {header: {frame_id: map}, pose: {position: {x: -5.0, y: 10.0}, orientation: {w: 1.0}}}}"
```

## 6. Stop

Press Ctrl+C in terminal 1, or run `docker rm -f mrs_sim`.

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| `ros2 topic list` in another terminal shows almost nothing | That terminal is not on Zenoh. The container must be started with `-e RMW_IMPLEMENTATION=rmw_zenoh_cpp` (step 2), or run `export RMW_IMPLEMENTATION=rmw_zenoh_cpp` there. |
| `Managed nodes are active` appears only once | One robot's Nav2 did not start. Look for `Failed to bring up` and the line above it. `transform from base_link to map did not become available` means that robot's SLAM is not running: `ros2 lifecycle get /<ns>/slam_toolbox` should say `active`. |
| `[goal_relay]: <ns>: Nav2 navigate_to_pose is not available` | That robot's Nav2 is not active yet. Wait for `Managed nodes are active`. |
| `Failed to create plan with tolerance` or `no valid path found` | The goal is inside an obstacle or its inflation, inside a keepout zone for that platform (the Warthog and the building), or unreachable for that footprint. Pick a point further from walls, or outside the zone. |
| A Warthog goal inside the building is accepted, then aborts after about 30 s | Expected: Nav2 accepts the goal, fails to plan into the keepout zone, runs its recoveries and gives up. `goal_relay` does not reject such goals up front yet. |
| `[<ns>.teammate_cloud]: <teammate>: no team TF` or `team TF older than 1.0 s` | The other robot is missing from the team tree on `/tf` (its SLAM or `tf_relay` is not running, or the sim is overloaded). Its footprint is left out until the TF comes back, so only the lidar sees it. |
| `Unable to transform robot pose into global plan's frame` | The SLAM `map -> odom` transform arrived too late. The tolerance is 0.5 s. If it keeps happening, the sim is overloaded: run `headless:=true` or drive one robot at a time. |
| The robot spins or backs up, then aborts | Recovery behaviors ran because it made no progress for 15 s, for example when blocked by the other robot. Send the goal again once the way is clear. |
| The robot stops short of an obstacle and will not go on | The collision monitor stopped it (`ros2 topic echo /<ns>/collision_monitor_state`). Nav2 replans; if it is boxed in, send a goal behind it. |
| Plans and costmaps appear in the wrong place in the team view | They are in the robot's own frames. Use the robot view (step 4). |

## Where things live (in mrs_ws)

| Path | What |
| --- | --- |
| `src/mrs_navigation/launch/nav2.launch.py` | Per-robot Nav2, same file for sim and hardware |
| `src/mrs_navigation/config/nav2.yaml` | Common Nav2 settings |
| `src/mrs_navigation/config/nav2_<platform>.yaml` | Footprint, speeds and inflation per platform |
| `src/mrs_navigation/mrs_navigation/goal_relay.py` | Team-map goal -> robot Nav2 action |
| `src/mrs_navigation/mrs_navigation/teammate_cloud.py`, `config/teammate_cloud.yaml` | Teammate footprints -> `teammates` (margin, rate, footprint size per platform) |
| `src/mrs_navigation/mrs_navigation/keepout_mask.py`, `config/keepout_mrs_test_site.yaml` | Keepout zones of the test site; another site passes its own file as `keepout_params_file` |
| `src/mrs_navigation/rviz/robot_nav.rviz` | Per-robot Nav2 view |
| `src/mrs_bringup/rviz/team_slam.rviz` | Team view with the goal tools |
