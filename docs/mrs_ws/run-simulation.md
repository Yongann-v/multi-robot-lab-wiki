# Run the mrs_ws two-robot simulation

Part of [mrs_ws](https://github.com/GSU-CARL/mrs_ws), the lab's Warthog + Husky multi-robot inspection system (ROS 2 Jazzy, Gazebo Harmonic). The source of this page is `docs/tutorials/01-run-simulation.md` in that repo.

You will build the dev image and the workspace, then spawn the Warthog (`w200_0111`) and the Husky (`a200_1103`) in the `mrs_test_site` world. Then you check their topics, lidars and TF, and drive both robots.

Time: about 20 min the first time (mostly the image download), under 1 min after that.

## What runs

- Gazebo Harmonic with `mrs_sim/worlds/mrs_test_site.sdf`: a walled outdoor arena with a concrete slab, and an indoor corridor building to the east.
- One stock Clearpath stack per robot, generated from the real robot.yaml files in `mrs_description/config/robots/` plus their `.sim.yaml` overlays. Each stack runs under its hardware namespace.
- `mrs_slam/tf_relay`, which republishes each robot's `/<ns>/tf` into the team `/tf` with `<ns>/` frame prefixes.

| Robot | Namespace | Spawn (x, y) | Role |
| --- | --- | --- | --- |
| Warthog W200 | `w200_0111` | (0, 0) | scout |
| Husky A200 | `a200_1103` | (0, 3) | inspector |

## 1. One-time setup

You need Docker and membership in the `docker` group (`groups` should list `docker`). The commands below assume the repo is at `~/mrs_ws`. Change the paths if yours is elsewhere.

Clone the repo (you need read access to `GSU-CARL/mrs_ws`) and build the dev image:

```bash
git clone git@github.com:GSU-CARL/mrs_ws.git ~/mrs_ws
cd ~/mrs_ws
docker build -f docker/Dockerfile.dev -t mrs_ws:dev docker
```

Build the workspace inside the container. Always build there, never on the host: the `install/` symlinks point to container paths (`/ws/...`).

```bash
docker run --rm -u $(id -u):$(id -g) -e HOME=/tmp/h \
  -v ~/mrs_ws:/ws -w /ws mrs_ws:dev \
  bash -c 'source /opt/ros/jazzy/setup.bash && colcon build --symlink-install'
```

Rerun this build after pulling changes or editing launch, config or world files that are not symlinked.

## 2. Start the simulation (terminal 1)

```bash
xhost +local:docker   # let the container open windows on your display
docker run -it --rm --name mrs_sim --net=host --ipc=host \
  -u $(id -u):$(id -g) -e HOME=/tmp/h \
  -e DISPLAY=$DISPLAY -e LIBGL_ALWAYS_SOFTWARE=1 -e ROS_DOMAIN_ID=42 \
  -e RMW_IMPLEMENTATION=rmw_zenoh_cpp \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v ~/mrs_ws:/ws -w /ws mrs_ws:dev \
  bash -c 'source install/setup.bash && ros2 launch mrs_bringup multi_sim.launch.py'
```

Gazebo opens with the arena. The Warthog spawns after about 5 s and the Husky about 5 s later. Look for two `Entity creation successful` lines in the log.

What the flags do:

| Flag | Why |
| --- | --- |
| `-e ROS_DOMAIN_ID=42` | `--net=host` puts sim topics on the lab network. The real robots use domain 0, so this keeps sim commands from reaching a powered-on robot with the same namespace. |
| `-e RMW_IMPLEMENTATION=rmw_zenoh_cpp` | The sim runs on Zenoh: `multi_sim.launch.py` starts the Zenoh router and every node uses it. Set on the container so that `docker exec` terminals (ros2 CLI, RViz, teleop) use Zenoh too; without it they see no topics. On Fast DDS, startup sometimes hung one robot's SLAM, collision monitor or Nav2. |
| `-e LIBGL_ALWAYS_SOFTWARE=1` | CPU rendering. Needed while the NVIDIA driver/library versions on the host mismatch (`nvidia-smi` fails). Drop it once `nvidia-smi` works. |
| `-u $(id -u):$(id -g) -e HOME=/tmp/h` | Files the container writes stay owned by you. Generated robot files go to `/tmp/h/.ros/mrs/<robot_id>/` inside the container. |
| `--name mrs_sim` | Lets terminal 2 attach with `docker exec`. |

Launch arguments (append to the `ros2 launch` line):

| Argument | Default | Use |
| --- | --- | --- |
| `headless:=true` | `false` | No GUI, for servers, SSH sessions or CI |
| `world:=<name>` | `mrs_test_site` | Another world in `mrs_sim/worlds/` |
| `fleet:=<path>` | `mrs_description/config/fleet.yaml` | A different robot list |
| `spawn_period:=<s>` | `5.0` | Seconds between robot spawns |
| `rmw:=<rmw>` | `rmw_zenoh_cpp` | Middleware for everything the launch starts. Match it with `RMW_IMPLEMENTATION` in your other terminals |

## 3. Check and drive (terminal 2)

Attach to the running container:

```bash
docker exec -it mrs_sim bash
source install/setup.bash && export ROS_DOMAIN_ID=42
```

Nothing outside the two namespaces except the shared topics:

```bash
ros2 topic list | grep -vE '^/(a200_1103|w200_0111)/'
# expect: /clock /parameter_events /rosout /tf /tf_static
```

Lidar point clouds (about 10-17 Hz with CPU rendering):

```bash
ros2 topic hz /w200_0111/sensors/lidar3d_0/points
ros2 topic hz /a200_1103/sensors/lidar3d_0/points
```

TF through the relay (team tree, prefixed frames):

```bash
ros2 run tf2_ros tf2_echo a200_1103/odom a200_1103/lidar3d_0_laser --ros-args -p use_sim_time:=true
ros2 run tf2_ros tf2_echo w200_0111/base_link w200_0111/lidar3d_0_laser --ros-args -p use_sim_time:=true
# Warthog lidar should be about 0.81 m above base_link, the real tower height
```

Drive (Ctrl+C stops publishing, and the robot brakes after a short timeout):

```bash
ros2 topic pub -r 10 /w200_0111/cmd_vel geometry_msgs/msg/TwistStamped "{twist: {linear: {x: 0.5}}}"
ros2 topic pub -r 10 /a200_1103/cmd_vel geometry_msgs/msg/TwistStamped "{twist: {angular: {z: 0.5}}}"
```

Watch odometry move:

```bash
ros2 topic echo /w200_0111/platform/odom/filtered --field pose.pose.position
```

## 4. Stop

Press Ctrl+C in terminal 1. The container is removed automatically (`--rm`). If it hangs, run `docker rm -f mrs_sim`.

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| `Authorization required` / `could not connect to display` | Run `xhost +local:docker` on the host first, and check that `echo $DISPLAY` is set. |
| Gazebo GUI crashes with `Failed to create OpenGL context` | Use `LIBGL_ALWAYS_SOFTWARE=1`, or run without the GUI using `headless:=true`. |
| `Unable to find image 'mrs_ws:dev'` | Build the image (step 1). |
| `Package 'mrs_bringup' not found` | Build the workspace inside the container (step 1). |
| `can't copy '.../build/...': doesn't exist` during build | Stale symlinks after a file was deleted or renamed. Delete that package's folders under `build/` and `install/` and rebuild. |
| `system.ros2.namespace ... must equal robot_id` | A robot.yaml namespace and its `fleet.yaml` robot_id differ. They must match, because the namespace is the robot's identity on hardware too. |
| Only one robot appears | Check terminal 1 for the first `[ERROR]`. Generator output for each robot is in `/tmp/h/.ros/mrs/<robot_id>/generate.log` inside the container. |
| `Couldn't open joystick /dev/input/ps4` | Harmless in sim: there is no gamepad attached. |
| `The IMU seems to be in free fall` at startup | Harmless: the robots drop a few cm when they spawn. |
| Topics from another sim or a real robot show up | Someone else on the LAN uses the same `ROS_DOMAIN_ID`. Pick another number (0-101) in both terminals. |

## Where things live (in mrs_ws)

| Path | What |
| --- | --- |
| `src/mrs_description/config/fleet.yaml` | Robot list, spawn poses, roles |
| `src/mrs_description/config/robots/<robot_id>.yaml` | Real Clearpath robot.yaml, used unchanged on hardware |
| `src/mrs_description/config/robots/<robot_id>.sim.yaml` | Sim-only overlay: swaps hardware-only packages for Clearpath's Hesai XT32 |
| `src/mrs_bringup/launch/multi_sim.launch.py` | The launch file used here ([details](https://github.com/GSU-CARL/mrs_ws/blob/main/src/mrs_bringup/README.md)) |
| `src/mrs_sim/tools/make_test_site.py` | Edit this, then run it to regenerate the world |
