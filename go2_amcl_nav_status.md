# Go2 AMCL/Nav2 Current Status

Date: 2026-06-25

## 실행한 명령

```bash
ros2 launch go2_nav navigation.launch.foxy.py \
  use_sim_time:=false \
  slam:=false \
  rviz:=true \
  map:=$HOME/dy_ws/src/go2_nav/maps/go2_map_25_16_39.yaml \
  params_file:=$HOME/dy_ws/src/go2_nav/params/go2_nav_params_foxy.yaml
```

## 현재 확인된 패키지 상태

다음 명령으로 Nav2, AMCL, SLAM Toolbox, TF 도구가 설치되어 있음을 확인했다.

```bash
ros2 pkg list | grep -E "nav2|slam_toolbox|tf2_tools"
```

확인된 주요 패키지:

```text
nav2_amcl
nav2_behavior_tree
nav2_bringup
nav2_bt_navigator
nav2_controller
nav2_costmap_2d
nav2_dwb_controller
nav2_lifecycle_manager
nav2_map_server
nav2_msgs
nav2_navfn_planner
nav2_planner
nav2_recoveries
nav2_rviz_plugins
nav2_util
nav2_voxel_grid
nav2_waypoint_follower
slam_toolbox
tf2_tools
```

결론: 목적지 지정과 Nav2 이동 테스트에 필요한 핵심 패키지는 설치되어 있다.

## 현재 판단

AMCL 노드와 Nav2 localization 파이프라인은 동작 중인 것으로 보인다. 좌표가 계산되고 costmap이 그 좌표를 기준으로 처리하려 하기 때문이다.

하지만 아직 "AMCL이 지도에 정확히 수렴했다"고 확정할 수는 없다. 현재 확실한 문제는 로봇 위치 추정값 또는 센서 원점이 map 범위 밖에 있다는 점이다.

가능성이 높은 원인:

1. RViz에서 `2D Pose Estimate`로 초기 위치를 아직 지도 안에 맞춰 찍지 않았다.
2. AMCL 초기 pose가 실제 로봇 시작 위치와 다르다.
3. map yaml의 `origin`, `resolution`, `width`, `height` 기준으로 현재 좌표가 지도 밖이다.
4. `odom`, `base_link`, `/scan.header.frame_id` 중 실제 Go2 frame과 설정이 맞지 않는 부분이 있다.

## AMCL/Nav2 목적지 지정 방법

AMCL이 지도 안에서 정상 수렴했다고 가정하면 RViz에서 목적지를 지정한다.

1. RViz `Fixed Frame`을 `map`으로 설정한다.
2. `Map`, `TF`, `LaserScan`, `ParticleCloud` 표시를 켠다.
3. `2D Pose Estimate`로 로봇 현재 위치와 방향을 지도 위에 맞춰 찍는다.
4. `Nav2 Goal` 또는 `2D Goal Pose`로 목적지를 찍고 드래그해서 도착 방향을 정한다.
5. Nav2가 경로를 만들고 `/cmd_vel`을 publish하는지 확인한다.

CLI로 목적지를 보낼 수도 있다.

```bash
ros2 topic pub --once /goal_pose geometry_msgs/msg/PoseStamped "{
  header: {
    frame_id: 'map'
  },
  pose: {
    position: {x: 1.0, y: 0.5, z: 0.0},
    orientation: {z: 0.0, w: 1.0}
  }
}"
```

우선은 RViz의 `Nav2 Goal`을 사용하는 것이 가장 좋다. 지도 위에서 위치와 방향을 같이 지정할 수 있고, 경로 생성 여부를 바로 볼 수 있다.

## 다음 확인 명령

map 범위 확인:

```bash
ros2 topic echo /map --once
```

확인할 값:

```text
info.resolution
info.width
info.height
info.origin.position.x
info.origin.position.y
```

map 범위 계산:

```text
x_min = origin.x
x_max = origin.x + width * resolution
y_min = origin.y
y_max = origin.y + height * resolution
```

현재 로봇 pose 확인:

```bash
ros2 topic echo /amcl_pose --once
ros2 run tf2_ros tf2_echo map base_link
ros2 run tf2_ros tf2_echo map odom
```

센서와 odom frame 확인:

```bash
ros2 topic echo /scan --once
ros2 topic echo /odom --once
```

확인할 값:

```text
/scan.header.frame_id
/odom.header.frame_id
/odom.child_frame_id
```

Nav2 lifecycle와 이동 명령 확인:

```bash
ros2 lifecycle nodes
ros2 topic echo /cmd_vel
```

RViz에서 목적지를 찍었을 때 `/cmd_vel`이 나오면 Nav2는 이동 명령을 내고 있는 것이다. 이때 Go2가 움직이지 않으면 다음 확인 대상은 `cmd_vel` remap 또는 Go2 driver가 실제로 구독하는 velocity topic이다.

## 다음 단계

1. RViz에서 `2D Pose Estimate`로 로봇 위치를 map 안쪽에 다시 지정한다.
2. `/amcl_pose`와 `tf2_echo map base_link` 좌표가 map 범위 안에 들어오는지 확인한다.
3. 경고가 사라지면 RViz `Nav2 Goal`로 짧은 거리 목적지를 찍는다.
4. `/cmd_vel`이 publish되는지 확인한다.
5. `/cmd_vel`이 나오는데 로봇이 움직이지 않으면 Go2 driver의 velocity topic 연결을 확인한다.
