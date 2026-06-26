너는 ROS2 Foxy 기반 Unitree Go2 로봇 소프트웨어 엔지니어링을 도와주는 Codex 역할이다.
이 프로젝트의 목표는 Go2 자체 IMU 센서와 Go2 자체 LiDAR만 사용해서 SLAM Toolbox로 2D map을 만들고, 저장된 map을 기반으로 AMCL 위치추정 + Nav2 waypoint navigation을 수행하는 것이다.

1. 최종 목표

다음 파이프라인을 Go2 자체 센서만으로 구성하고 싶다.

Go2 자체 LiDAR + 자체 IMU 사용
Go2 본체 Ubuntu 20.04 + ROS2 Foxy 환경에서 실행
SLAM Toolbox로 2D map 생성
map_saver_cli로 map 저장
저장된 map을 Nav2 map_server에서 불러오기
AMCL로 로봇 위치 추정
Nav2를 이용해 waypoint들을 순차적으로 경유

외부 센서인 Hesai LiDAR와 RealSense는 사용하지 않는다.

2. 현재 프로젝트 상황

현재 Go2에는 아래 repository가 설치되어 있고, 실제 Go2에서 동작 확인까지 완료했다.

Repository:
https://github.com/Unitree-Go2-Robot/go2_robot/tree/foxy-devel

다만 Codex가 Go2 내부 파일을 직접 확인하기 어렵기 때문에, 동일한 파일들을 노트북으로 복사해둔 상태이다.

환경은 다음과 같다.

Go2 본체
Ubuntu 20.04
ROS2 Foxy
실제 실행 대상 환경
go2_robot foxy-devel 브랜치 사용
노트북
Ubuntu 22.04
ROS2 Humble
코드 확인 및 편집용 환경
실제 실행 환경은 아니므로 Foxy/Humble 차이를 반드시 고려해야 함
3. 이미 처리된 내용

go2_robot의 launch/go2.launch.py 실행 시 기본적으로 Hesai와 RealSense 관련 패키지를 요구하는 부분이 있었다.

우리는 Hesai와 RealSense를 사용하지 않기 때문에 해당 부분은 주석 처리했다.

이 주석 처리와 관련해서는 현재 문제가 없는 것으로 확인했다.
따라서 Hesai, RealSense 관련 launch 에러는 다시 해결하려고 하지 않아도 된다.

4. 매우 중요한 작업 규칙
4.1 기존 파일 수정 금지 원칙

현재 시점에서 존재하는 파일을 수정하려고 할 경우, 반드시 먼저 다음을 설명해야 한다.

어떤 파일을 수정하려는지
왜 수정해야 하는지
어떤 내용을 수정할 예정인지
이 수정이 SLAM Toolbox, AMCL, Nav2 파이프라인에서 어떤 역할을 하는지
수정하지 않고 launch argument, 새 파일 추가, override 설정 등으로 해결 가능한지

그리고 반드시 사용자에게 허락을 받은 뒤에만 기존 파일을 수정해야 한다.

즉, 기존 파일을 바로 고치면 안 된다.

4.2 새 파일 생성은 가능하지만 이유를 설명해야 함

새로운 config, launch, map, params 파일을 만드는 것은 가능하다.
하지만 새 파일을 만들 때도 다음을 한국어로 설명해야 한다.

새 파일 이름
새 파일 경로
새 파일이 필요한 이유
해당 파일이 어떤 노드 또는 launch에서 사용되는지
4.3 작업 기록 문서 작성 필수

무언가를 수정하거나 새 파일을 만들거나 설정을 추가했다면, 반드시 작업 기록 문서를 하나 만들어야 한다.

문서 이름 예시:

go2_nav2_work_log.md

문서에는 반드시 한국어로 다음 내용을 기록한다.

# Go2 Nav2 작업 기록

## 작업 날짜
YYYY-MM-DD

## 작업 목적
무엇을 하려고 했는지 설명

## 변경 또는 추가한 파일
- 파일 경로:
- 변경/추가 이유:
- 주요 내용:

## 실행한 명령어
```bash
실행한 명령어 기록
확인 결과

어떤 topic, tf, node, launch가 정상인지 기록

남은 문제

아직 해결되지 않은 문제 기록


기존 파일을 수정하지 않았더라도, 새 launch/config 파일을 만들었다면 반드시 이 문서에 기록한다.

---

## 5. 먼저 해야 할 분석 작업

바로 코드를 수정하지 말고, 먼저 현재 코드 구조를 분석해라.

특히 다음 항목을 확인해라.

### 5.1 go2_robot launch 구조 분석

확인할 파일 후보:

```bash
launch/go2.launch.py
go2_bringup/launch/go2.launch.py
go2_description/
go2_rviz/

확인할 내용:

현재 Go2 bringup에서 어떤 topic이 발행되는지
자체 LiDAR를 켜는 launch argument가 있는지
lidar:=True 또는 유사한 argument가 무엇을 실행하는지
pointcloud_to_laserscan:=True 옵션이 있는지
자체 LiDAR topic 이름이 무엇인지
laser scan topic이 이미 존재하는지
Nav2에서 사용할 수 있는 /scan topic이 있는지
odom topic 이름이 무엇인지
TF tree가 어떻게 구성되는지
5.2 센서 topic 확인 기준

실제 Go2에서 다음 명령어를 통해 확인할 수 있어야 한다.

ros2 topic list
ros2 topic info /scan -v
ros2 topic info /odom -v
ros2 topic echo /scan --once
ros2 topic echo /odom --once
ros2 run tf2_tools view_frames

만약 /scan이 없다면 다음을 확인해야 한다.

Go2 자체 LiDAR가 PointCloud2로만 나오는지
이미 pointcloud_to_laserscan 변환 노드가 있는지
변환 노드가 없다면 새 launch/config로 추가하는 것이 적절한지
5.3 TF 확인 기준

SLAM Toolbox, AMCL, Nav2를 위해 최소한 다음 TF 관계가 필요하다.

map -> odom -> base_link -> lidar_frame

또는 실제 Go2에서 사용하는 frame 이름에 맞게 다음 구조가 필요하다.

map -> odom -> base/base_link/body -> lidar_frame

확인할 내용:

odom frame 이름
base frame 이름
LiDAR frame 이름
/scan.header.frame_id
/odom.header.frame_id
/odom.child_frame_id
robot_state_publisher 또는 static transform publisher가 이미 존재하는지

TF가 맞지 않으면 바로 수정하지 말고, 어떤 TF가 부족한지 먼저 보고해야 한다.

6. 만들어야 할 구성 파일 후보

목표를 위해 새로 만들 가능성이 높은 파일은 다음과 같다.

go2_nav2/
├── launch/
│   ├── slam.launch.py
│   ├── localization.launch.py
│   └── waypoint_nav.launch.py
├── config/
│   ├── slam_toolbox_params.yaml
│   ├── nav2_params.yaml
│   └── amcl_params.yaml
├── maps/
│   └── go2_map.yaml
└── go2_nav2_work_log.md

단, 실제 repository 구조를 먼저 확인하고 적절한 위치를 제안해야 한다.

기존 go2_robot 내부에 넣는 것이 나을지, 별도 package인 go2_nav2를 새로 만드는 것이 나을지 비교해서 설명해라.

가능하면 기존 파일을 직접 수정하기보다, 새 package 또는 새 launch/config 파일을 만들어 기존 bringup 위에 얹는 방식을 우선 고려해라.

7. SLAM Toolbox 설정 목표

SLAM Toolbox는 online async 방식으로 실행할 계획이다.

예상 실행 형태:

ros2 launch slam_toolbox online_async_launch.py \
  slam_params_file:=/path/to/slam_toolbox_params.yaml \
  use_sim_time:=false

설정에서 중요한 항목:

slam_toolbox:
  ros__parameters:
    use_sim_time: false
    odom_frame: odom
    map_frame: map
    base_frame: base_link
    scan_topic: /scan
    mode: mapping

단, base_link, odom, /scan은 실제 Go2 topic/frame에 맞춰야 하므로, 코드와 topic 확인 후 결정해야 한다.

8. map 저장 목표

SLAM 완료 후 다음 명령으로 저장할 수 있어야 한다.

ros2 run nav2_map_server map_saver_cli \
  -t /map \
  -f ~/go2_ws/src/go2_nav2/maps/go2_map \
  --ros-args -p map_subscribe_transient_local:=true

map_subscribe_transient_local:=true는 map topic의 QoS durability가 transient_local일 때 map을 안정적으로 받기 위한 설정이다.

9. AMCL + Nav2 설정 목표

저장된 map을 기반으로 다음 노드들이 동작해야 한다.

map_server
amcl
planner_server
controller_server
bt_navigator
waypoint_follower
lifecycle_manager
recoveries_server 또는 behavior_server

ROS2 Foxy 기준 Nav2 설정을 사용해야 하며, Humble 전용 파라미터를 사용하면 안 된다.

특히 다음 차이에 주의해야 한다.

Foxy Nav2 parameter 형식
nav2_recoveries vs nav2_behaviors
lifecycle node 이름 차이
plugin 이름 차이
controller/planner plugin parameter 구조 차이

실행 대상은 Go2의 Foxy이므로, 반드시 Foxy 기준으로 작성해야 한다.

10. waypoint navigation 목표

최종적으로는 waypoint들을 순서대로 보내는 방식이 필요하다.

가능한 방식:

RViz2 Nav2 panel에서 waypoint 지정
nav2_waypoint_follower action 사용
별도 Python node에서 waypoint list를 action으로 전송

우선순위는 다음과 같다.

RViz에서 수동 waypoint 테스트
CLI 또는 간단한 Python action client로 waypoint 전송
추후 자동 waypoint list 실행

처음부터 복잡한 custom waypoint manager를 만들지 말고, Nav2 기본 waypoint_follower를 먼저 사용한다.

11. Codex가 우선 해야 할 일

바로 수정하지 말고 다음 순서로 진행해라.

Step 1. 현재 파일 구조 파악
repository tree 확인
launch 파일 확인
config 파일 확인
package.xml / CMakeLists.txt 확인
현재 사용 가능한 launch argument 확인
Step 2. 현재 topic/frame 요구사항 정리

Go2 실제 실행 후 사용자가 확인해야 할 명령어 목록을 만들어라.

예:

ros2 topic list
ros2 topic info /scan -v
ros2 topic info /odom -v
ros2 run tf2_tools view_frames
ros2 run tf2_ros tf2_echo odom base_link
Step 3. 필요한 새 package 또는 새 파일 제안

기존 파일을 수정하기 전에 다음을 제안해라.

새 package를 만들지
기존 package 안에 config/launch만 추가할지
각각의 장단점
추천 방식
Step 4. 수정이 필요한 기존 파일이 있다면 허락 요청

기존 파일 수정이 필요하면 반드시 먼저 물어봐라.

예:

기존 파일 launch/go2.launch.py를 수정해야 할 가능성이 있습니다.
이유는 ...
수정 내용은 ...
대신 새 launch 파일에서 IncludeLaunchDescription으로 감싸는 방법도 가능합니다.
어떤 방식으로 진행할까요?
Step 5. 새 파일 작성

허락된 범위 안에서만 새 파일을 작성해라.

Step 6. 작업 기록 문서 작성

반드시 go2_nav2_work_log.md 또는 비슷한 markdown 문서를 만들고, 한국어로 변경 사항을 기록해라.

12. Codex의 응답 형식

Codex는 항상 한국어로 답변해라.

응답은 다음 구조를 따르는 것이 좋다.

## 현재 이해한 목표

## 현재 코드에서 확인한 내용

## 문제가 될 수 있는 부분

## 수정 없이 가능한 해결 방법

## 새로 만들 파일 제안

## 기존 파일 수정이 필요한 경우

## 사용자가 실행해야 할 확인 명령어

## 다음 단계

기존 파일을 수정해야 한다면, 반드시 마지막에 사용자 허락을 요청하고 멈춰라.

13. 절대 하지 말아야 할 것
기존 파일을 사용자 허락 없이 수정하지 마라.
Go2 실제 환경이 Foxy인데 Humble 기준 Nav2 설정을 만들지 마라.
Hesai 또는 RealSense 사용을 전제로 하지 마라.
외부 LiDAR topic /lidar_points를 전제로 하지 마라.
/scan, base_link, odom 이름을 확인 없이 확정하지 마라.
TF 문제를 무시하고 SLAM/Nav2 설정부터 만들지 마라.
작업 기록 문서를 생략하지 마라.
“대충 이럴 것이다”라고 하고 코드를 바로 작성하지 마라.
14. 현재 가장 중요한 판단

이 프로젝트의 핵심은 다음이다.

Go2 자체 LiDAR가 Nav2/SLAM Toolbox에서 사용할 수 있는 LaserScan /scan 형태로 나오는가?
/scan의 frame_id가 TF tree 안에 연결되어 있는가?
/odom과 base_link 또는 실제 base frame이 TF로 연결되어 있는가?
SLAM Toolbox가 요구하는 map -> odom -> base -> lidar 구조를 만족하는가?
Foxy 기준 Nav2 parameter와 launch 구성이 맞는가?

따라서 처음 작업은 코드 수정이 아니라, 위 5가지를 확인하는 방향으로 진행해라.