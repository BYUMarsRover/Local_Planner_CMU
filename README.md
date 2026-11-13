<img src="img/header.jpg" alt="Header" width="100%"/>

# CMU local planner: BYU Mars Rover fork

This is BYU Mars Rover's fork of CMU's [autonomous_exploration_development_environment](https://github.com/HongbiaoZ/autonomous_exploration_development_environment), branch `humble`, based on upstream commit `158e67b`. CMU's original project page is [cmu-exploration.com](https://www.cmu-exploration.com).

We plan to use CMU's `terrain_analysis`, `localPlanner` and `pathFollower` as the rover's local planner. Upstream assumes a 360° lidar. The rover only has a ZED 2i stereo camera, which looks forward. This fork:

1. **Simulates the ZED 2i** instead of the Velodyne, so we can see how the planner behaves with a forward-only view.
2. **Patches the planner** so it only drives where the camera can see, and turns in place when it's blocked.
3. **Adds a Hanksville world** built from our GeoTIFF elevation data, with ground-truth terrain following.

All planner patches are **off by default**. Without the ZED arguments below, `localPlanner` and `pathFollower` behave like upstream. The one default that differs from upstream is the simulated sensor: it is now the ZED (see [Sensor](#sensor-zed-2i-in-place-of-the-velodyne)).

## Quick start

### 1. Install and build

Follow CMU's ROS 2 Humble setup (dependencies, then `download_environments.sh` for the stock worlds), cloning this fork instead of upstream:

```bash
cd ~
git clone -b humble https://github.com/BYUMarsRover/Local_Planner_CMU.git autonomous_exploration_development_environment
cd autonomous_exploration_development_environment
./src/vehicle_simulator/mesh/download_environments.sh
```

### 2. Generate the Hanksville terrain (once)

The terrain mesh and heightmap are not in Git, because `src/vehicle_simulator/mesh/*` is ignored like CMU's other meshes. Generate them from the GeoTIFF in the [marsrover-ros2](https://github.com/BYUMarsRover) repo. This needs `numpy` and `rasterio`.

```bash
cd ~/mars_ws/src/autonomy/autonomy/maps
python3 dem_to_gazebo.py hanksville.tif --name hanksville --size 500 --force \
    --out ~/autonomous_exploration_development_environment/src/vehicle_simulator
```

- `--force` is needed because `world/hanksville.world` is already committed. The script rewrites it from the same inputs.
- To confirm you got the same tile as everyone else, this should print nothing:
  ```bash
  git diff src/vehicle_simulator/world/hanksville.world
  ```

The output goes to `src/vehicle_simulator/mesh/hanksville/`:

| File | Purpose |
|---|---|
| `meshes/terrain.obj` | Terrain mesh, used for collision and drawing |
| `heightmap.bin` | Ground-truth heights for `vehicleSimulator` |
| `preview/pointcloud.ply` | Overview map for `visualization_tools` |
| `origin.json` | Local origin, for GPS ↔ map conversion |

### 3. Build

Generate the terrain **before** building, so `install/` links the new mesh folder.

```bash
cd ~/autonomous_exploration_development_environment
colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=Release
source install/setup.bash
```

### 4. Run

Hanksville, ZED mode:

```bash
ros2 launch vehicle_simulator system_hanksville.launch \
    twoWayDrive:=false useFovCheck:=true rotateWhenBlocked:=true
```

Notes:
- Add `gazebo_gui:=true` to open the Gazebo window. RViz opens on its own after about 8 s.
- Set goals with the **Waypoint** tool in RViz.
- The same ZED arguments work with every world: `system_garage.launch`, `system_forest.launch`, and so on.

## Launch arguments

Pass arguments as `name:=value` on the `ros2 launch` line. They reach every included launch file.

> **Write decimal parameters with a decimal point** (`fovHalfAngle:=36.0`, not `36`). ROS 2 rejects an integer for a `double` parameter, and the node crashes at startup: `InvalidParameterTypeException ... setting it to {integer} is not allowed`. The robot then just sits still. `system_hanksville.launch` converts its own position, height and slope arguments for you, but nothing converts planner parameters.

### ZED mode (planner)

| Argument | Default | ZED value | Node | What it does |
|---|---|---|---|---|
| `twoWayDrive` | `true` | `false` | `localPlanner`, `pathFollower` | Never drive in reverse. The camera can't see behind the rover. |
| `useFovCheck` | `false` | `true` | `localPlanner` | Only accept paths that lie entirely inside the camera's horizontal view |
| `fovHalfAngle` | `55.0` | `55.0` (2.1 mm lens), `36.0` (4 mm lens) | `localPlanner` | Half of the camera's horizontal field of view, in degrees |
| `rotateWhenBlocked` | `false` | `true` | `pathFollower` | When no valid path exists, turn in place instead of stopping |

**Use all three switches together.** On its own, `useFovCheck` blocks most goals that are behind the rover, so it just stops. `rotateWhenBlocked` is what gets it turning to look for a way out. `twoWayDrive:=false` makes sure that turn happens facing forward.

We don't yet know which ZED 2i lens the rover has. The 2.1 mm lens sees about 110° × 70° and the 4 mm lens about 72° × 44°.

### Simulator

| Argument | Default | What it does |
|---|---|---|
| `sensor_xacro` | `zed_ray.urdf.xacro` | Simulated sensor. Use `lidar.urdf.xacro` for the original VLP-16. |
| `gazebo_gui` | `false` | Open the Gazebo window (`gzclient`). The simulation runs either way. |
| `vehicleX`, `vehicleY`, `vehicleYaw` | `0.0` | Start pose, in the map frame |
| `useTerrainMap` | `true` | Use `mesh/<world>/heightmap.bin` for the rover's height and tilt when the file exists |
| `terrainMapInclRadius` | `0.75` | Half-span (m) over which ground slope is averaged for tilt |
| `maxIncl` | `20.0` (Hanksville), `30.0` (other worlds) | Steepest slope in degrees. Logs a warning when the ground is steeper. |

## Troubleshooting

- **The robot doesn't move after a waypoint.** Scroll up in the launch terminal for a red `process has died` line. The line above it says why. The usual cause is an integer passed to a decimal parameter (see the note under [Launch arguments](#launch-arguments)).
- **Odd behavior after a crashed launch** (jitter, flickering maps, two of everything in `ros2 node list`). A crashed launch can leave nodes running, and the next launch then fights with them. Check before every launch, and stop leftovers:
  ```bash
  COLUMNS=200 ps -eo pid,cmd | grep -E "gzserver|pathFollower|terrainAnalysis|vehicleSimulator" | grep -v grep
  pkill -f gzserver; pkill -f gzclient; pkill -f vehicleSimulator; pkill -f localPlanner
  pkill -f pathFollower; pkill -f terrainAnalysis; pkill -f sensorScanGeneration
  pkill -f visualizationTools; pkill -f static_transform_publisher
  pkill -f robot_state_publisher; pkill -f rviz2; pkill -f joy_node
  ros2 daemon stop
  ```
- **`xacro ... No such file` for `zed_ray.urdf.xacro`, or a missing Hanksville mesh.** A new file only gets linked into `install/` by a **successful** build. If `colcon build` printed `Failed`, nothing was installed. Fix the first `error:` line and rebuild.
- **`No terrain map at .../hanksville/heightmap.bin`.** Generate the terrain (Quick start, step 2), then rebuild.


### Example commands

```bash
# Stock CMU behavior with the ZED sensor (reverses into unseen space)
ros2 launch vehicle_simulator system_garage.launch

# Stock CMU behavior with the original 360° lidar
ros2 launch vehicle_simulator system_garage.launch sensor_xacro:=lidar.urdf.xacro

# ZED mode, 4 mm lens, forest
ros2 launch vehicle_simulator system_forest.launch \
    twoWayDrive:=false useFovCheck:=true rotateWhenBlocked:=true fovHalfAngle:=36.0

# ZED mode, Hanksville, Gazebo window, different start point
ros2 launch vehicle_simulator system_hanksville.launch gazebo_gui:=true \
    twoWayDrive:=false useFovCheck:=true rotateWhenBlocked:=true vehicleX:=-26 vehicleY:=40
```

## What changed from upstream

### Sensor: ZED 2i in place of the Velodyne

**New file: `src/vehicle_simulator/urdf/zed_ray.urdf.xacro`.** It's a Gazebo ray sensor shaped like the ZED's depth cloud. It reuses the Velodyne plugin and still publishes on `/velodyne_points`, so no subscribers changed. Every lens-dependent value is a property at the top of the file:

| Property | Value | Meaning |
|---|---|---|
| `hfov_deg`, `vfov_deg` | 110, 70 | Field of view, 2.1 mm lens. For the 4 mm lens use 72, 44. |
| `min_range`, `max_range` | 0.3 m, 6.0 m | Minimum depth, and the trusted depth the rover's cloud will be cropped to |
| `tilt_deg` | 0 | Camera pitched down. Faked by shifting the vertical ray angles. |
| `res_deg` | 1.0 | Angle between rays: 111 × 71 ≈ 7,900 rays |
| `hz` | 10 | Requested rate. About 7 Hz is typical on our machines, because Gazebo ray-casts on the CPU. |
| `noise` | 0.02 m | Gaussian range noise |

Changing `fovHalfAngle` only changes the planner. To simulate the 4 mm lens, also edit `hfov_deg`, `vfov_deg` and `min_range` here.

**Changed: `src/vehicle_simulator/launch/vehicle_simulator.launch`.** Adds the `sensor_xacro` argument, plus 180 s spawn timeouts, because the larger worlds are slow to load.

**Changed: `src/vehicle_simulator/src/vehicleSimulator.cpp`.**
- **The sensor turns with the robot.** Upstream posed the lidar with yaw = 0. That's harmless for a 360° sensor, but a forward camera has to face where the robot faces.
- **`scanHandler` registers each scan with the vehicle's full orientation:** roll, then pitch, then yaw, then translation. Upstream only applied the terrain's roll and pitch.

### Planner: only drive where the camera can see

Upstream treats space with no points as free. With a forward-only camera, everything behind and beside the rover has no points. So for a goal behind the rover, upstream chooses a backward path through space it has never seen, and `pathFollower` reverses along it, through walls if they're there.

**`src/local_planner/src/localPlanner.cpp`:**
- **New `pathInFov()`.** The planner scores 343 pre-drawn paths at 36 rotations (10° apart). For each candidate, this function rotates and scales the path's sample points into the vehicle frame and measures each point's bearing from the camera at (`sensorOffsetX`, `sensorOffsetY`). It rejects the path if any bearing exceeds `fovHalfAngle`.
  - It only checks the stretch of path the obstacle check uses: up to `pathRange`, and just past the goal.
  - It skips points under the rover's own footprint.
- **Scoring and `/free_paths` use the check.** With `useFovCheck:=true`, only in-view paths can be selected, and RViz's green FreePaths fan shrinks to the camera's wedge.
- **The blocked path carries a turn direction.** When no path is free, upstream publishes a 1-point path at (0, 0), which means stop. It now publishes the point at (0, ±1): 1 m to the left (+1) or right (−1).
  - The side is chosen from the goal direction **once**, when the rover first becomes blocked, and kept until a path is found again. This stops it wobbling left and right when the goal is straight ahead behind a wall.
  - It is still a 1-point path, so code that detects "blocked" by path size still works.

**`src/local_planner/src/pathFollower.cpp`:** with `rotateWhenBlocked:=true`, a 1-point path no longer forces the yaw rate to 0. `pathFollower`'s existing steering turns toward the sideways point at `maxYawRate`, while its speed stays 0. The result is that the rover spins in place until the camera finds an open path.

**`src/local_planner/launch/local_planner.launch`:** adds the `useFovCheck`, `fovHalfAngle` and `rotateWhenBlocked` arguments, with defaults that keep upstream behavior.

### Hanksville world and ground-truth terrain

- **`world/hanksville.world`, `launch/system_hanksville.launch`.** A 500 m × 500 m tile of the Hanksville GeoTIFF at 1 m resolution.
  - The local origin is the tile center: 38.42152, −110.78948, elevation 1377.22 m (UTM 12N, EPSG:26912).
  - The map frame is x = east, y = north, z = up, relative to that origin. It uses grid north, which is about 1° from true north here.
- **`include/vehicle_simulator/terrain_map.h` plus `vehicleSimulator.cpp`.** While the rover is on the heightmap, its height and tilt come from the elevation data instead of from a plane fitted to `/terrain_map`. Upstream's fit fails on steep banks, so the simulated rover drove into hills.
  - Off the map, or in worlds without a heightmap, the upstream method is used.
  - The launch output says which one is active: `Loaded terrain map ...`, or `No terrain map at ...`.

## Known limitations

| Limitation | Effect | Settings involved |
|---|---|---|
| Obstacles beside the rover are forgotten once they leave the camera's view | It may cut corners around obstacles it has just passed. `clearDyObs` assumes a 360° sensor: it hides remembered points the sensor "should" see but doesn't, and it only checks vertical angle. Points beyond 4 m also decay after 2 s. **Not fixed yet.** | `terrain_analysis.launch`: `clearDyObs`, `minDyObsVFOV`/`maxDyObsVFOV`, `decayTime`, `noDecayDis` |
| Turning in place has no limit | If every direction is blocked, the rover spins forever. On the rover, the state machine should take over. | `rotateWhenBlocked` |
| Turning in place doesn't check the robot's swept corners | The rover could clip something right beside it while spinning | `checkRotObstacle` (off upstream) |
| The simulated cloud is sparse and evenly spaced in angle | The real ZED is far denser, and its error grows with distance | `zed_ray.urdf.xacro` |
| Scans are registered with the latest pose | Walls look slightly thick while the rover turns. The rover's bridge should use the pose at the cloud's timestamp. | `use_gazebo_time` (false) |
| The tilt is approximate | Shifting ray angles matches a tilted camera along the center column only | `tilt_deg` |

## Credit

Simulator, planner and visualization tools are from CMU's [Autonomous Exploration Development Environment](https://www.cmu-exploration.com) (BSD license). Please use their project page for the original instructions and the other planners (TARE, FAR, DSV).
