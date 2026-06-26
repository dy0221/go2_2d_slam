# Go2 Nav2 작업 기록

## 작업 날짜
2026-06-26

## 작업 목적
Go2 자체 LiDAR와 자체 odometry를 사용해 SLAM Toolbox로 2D map을 만들 때, 직선 구조물이 지도에서 휘어지는 문제를 줄이기 위해 센서 frame, `/scan`, `/odom` publish 상태를 확인하고 Go2 본체의 odometry 입력 경로를 수정했다.

## 변경 또는 추가한 파일

- 파일 경로:
  - Go2 본체: `~/dy_ws/src/go2_driver/include/go2_driver/go2_driver.hpp`
  - Go2 본체: `~/dy_ws/src/go2_driver/src/go2_driver/go2_driver.cpp`
- 변경 이유:
  - 기존 `/odom`은 `/utlidar/robot_pose` callback에서 최초 1회만 publish되는 구조였다.
  - 실제 Go2에는 `/utlidar/robot_odom`이 약 150 Hz로 정상 publish되고 있었으므로, 이 odometry를 `/odom`으로 republish하는 구조가 더 적절하다고 판단했다.
  - Nav2와 SLAM 관련 노드가 연속적인 odometry stream을 사용할 수 있게 하기 위함이다.
- 주요 내용:
  - `robot_pose_sub_`를 `robot_odom_sub_`로 변경했다.
  - 구독 타입을 `geometry_msgs::msg::PoseStamped`에서 `nav_msgs::msg::Odometry`로 변경했다.
  - 구독 topic을 `/utlidar/robot_pose`에서 `/utlidar/robot_odom`으로 변경했다.
  - callback을 `publish_pose_stamped(...)`에서 `publish_odom(...)`으로 변경했다.
  - `/utlidar/robot_odom` 메시지를 받아 `/odom`으로 publish하고, 같은 pose로 `odom -> base_link` TF를 publish하도록 했다.

- 파일 경로:
  - Go2 본체: `~/dy_ws/src/go2_driver/launch/go2_driver.launch.py`
- 변경 또는 확인 이유:
  - `/scan`에서 `range_min: 0.0`, `range_max: .inf`가 확인되어 `pointcloud_to_laserscan`의 range 파라미터를 명시하는 방안을 검토했다.
  - launch 파일 수정 중 Python 문법 오류가 발생했으며, `parameters=[{...}],` 블록의 괄호 닫힘을 수정해야 함을 확인했다.
- 권장 주요 내용:
  - `pointcloud_to_laserscan` 파라미터는 아래 형태로 유지하는 것을 권장한다.

```python
parameters=[{
    "target_frame": "base_link",
    "min_height": -0.3,
    "max_height": 0.5,
    "range_min": 0.15,
    "range_max": 10.0,
    "use_inf": True,
}],
```

## 실행한 명령어

```bash
ros2 topic hz /odom
ros2 topic list | grep odom
ros2 topic hz /utlidar/robot_odom
ros2 topic echo /utlidar/robot_odom --once
ros2 topic echo /scan --once
ros2 run tf2_ros tf2_echo base_link radar
ros2 run tf2_ros tf2_echo odom base_link
colcon build --packages-select go2_driver
source install/setup.bash
ros2 launch go2_bringup go2.launch.py
```

## 확인 결과

- `/utlidar/robot_odom`은 약 150 Hz로 정상 publish되고 있었다.

```text
average rate: 약 150 Hz
header.frame_id: odom
child_frame_id: base_link
```

- `/odom` 수정 후 `/odom`도 약 150 Hz로 정상 publish되는 것을 확인했다.

```text
average rate: 약 149~150 Hz
```

- `/scan.header.frame_id`는 `base_link`로 확인됐다.

```text
header.frame_id: base_link
scan_time: 0.033333335
range_min: 0.0
range_max: .inf
```

- `base_link -> radar` TF는 존재했다.

```text
Translation: [0.289, 0.000, -0.047]
Rotation RPY degree: [180.000, 15.091, 180.000]
```

- RViz에서 `Fixed Frame=base_link` 기준으로 `/scan`을 봤을 때 scan은 정상으로 확인됐다.

- `odom -> base_link` TF는 존재하지만 roll, pitch, z가 포함되어 있어 `Fixed Frame=odom`에서는 scan 평면이 살짝 기울어진 것처럼 보일 수 있음을 확인했다.

## 남은 문제

- 지도에서 직선이 휘어지는 문제가 `/odom` 연속 publish 수정 후 얼마나 개선되는지 재테스트가 필요하다.
- `/scan`의 `range_min`, `range_max`가 아직 원하는 값으로 반영됐는지 확인해야 한다.
- Go2 보행 중 `base_link`에 roll/pitch/z 흔들림이 들어가므로, 문제가 계속되면 2D SLAM용 `base_footprint` frame 도입을 검토해야 한다.

## 다음 작업 순서

1. `go2_bringup`을 정상 launch한다.
2. `/odom`이 약 150 Hz로 계속 publish되는지 확인한다.
3. `/scan`의 `range_min`, `range_max`, `frame_id`를 확인한다.
4. 조이스틱을 끄고 `/cmd_vel`로 저속 이동한다.

```bash
ros2 service call /switch_joystick go2_interfaces/srv/SwitchJoystick "flag: false"
```

5. SLAM Toolbox를 실행하고 천천히 mapping한다.
6. 그래도 직선이 휘면 `base_footprint` 기반 2D frame 구조를 추가한다.
