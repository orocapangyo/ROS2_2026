# 8장. Gazebo Harmonic과 URDF 기초

> **학습 목표**
> - 시뮬레이터(Gazebo)가 왜 필요한지, ROS 2와 어떻게 연결되는지 이해한다.
> - **Gazebo Harmonic**과 `ros_gz`(브리지/스폰)의 역할을 안다.
> - URDF로 간단한 이동 로봇 모델을 기술한다.
> - ROS 2 좌표계 규칙과 TF 프레임의 역할을 이해한다.
> - 로봇을 Gazebo에 띄우고 RViz로 본다.

> **이번 장의 산출물**
> - URDF 로봇 모델을 RViz에서 확인하고 Gazebo Harmonic에 spawn한다.
> - `base_link`, `odom`, `map`, 센서 프레임의 관계를 TF 트리로 확인한다.
> - `ros_gz` bridge로 Gazebo와 ROS 2 토픽을 연결한다.
>
> **공통 학습 흐름**: 개념 → 따라하기 → 코드 해설 → 실행 확인 → 버전/환경 체크 → 트러블슈팅 → 연습문제 → 마무리 점검

> ⚠️ **이 장은 1권의 마지막이자, 2권 시뮬레이션 프로젝트의 토대다.** 또한 gcamp(Foxy +
> Gazebo Classic) → Jazzy + Gazebo Harmonic 포팅이 본격적으로 시작되는 지점이다.

---

## 8.1 왜 시뮬레이터인가

실로봇은 비싸고, 망가지고, 환경 세팅이 번거롭다. 시뮬레이터는 같은 ROS 2 인터페이스를
그대로 쓰면서 **가상 로봇·가상 센서·가상 물리**를 제공한다. 코드를 시뮬레이터에서 검증한
뒤 실로봇(4부 TurtleBot3)에 거의 그대로 올릴 수 있다.

---

## 8.2 Gazebo Harmonic과 ros_gz

> 🔁 **Foxy → Jazzy 핵심 변화**
> - Foxy 시절의 **Gazebo Classic**(`gazebo`, `gazebo_ros`)은 지원이 끝나가고, Jazzy의
>   짝은 **Gazebo(구 Ignition) Harmonic**이다.
> - ROS 2와의 연결은 **`ros_gz`** 패키지군이 담당한다.
>   - `ros_gz_sim` — Gazebo 시뮬레이터 실행·로봇 스폰(`create`)
>   - `ros_gz_bridge` — ROS 토픽 ↔ Gazebo 토픽 변환(브리지)
> - 즉 gcamp의 `gazebo_ros` 기반 런치·플러그인은 **그대로 동작하지 않으며**, `ros_gz`
>   기준으로 재작성해야 한다.

설치(1장의 `ros-jazzy-desktop`에 보통 포함되지만, 명시 설치):

```bash
sudo apt install ros-jazzy-ros-gz
```

Gazebo만 단독 실행해 보기:

```bash
gz sim empty.sdf        # 빈 월드가 뜨면 정상
```

---

## 8.3 URDF — 로봇을 글로 기술하기

**URDF(Unified Robot Description Format)** 는 로봇의 **링크(link, 강체)** 와
**조인트(joint, 연결)** 를 XML로 적은 것이다. 모양·크기·관성·바퀴 회전축 등을 담는다.

가장 단순한 두 바퀴 로봇의 뼈대 — `my_robot_description/urdf/my_robot.urdf`:

```xml
<?xml version="1.0"?>
<robot name="my_robot">

  <!-- 본체 -->
  <link name="base_link">
    <visual>
      <geometry><box size="0.4 0.2 0.1"/></geometry>
    </visual>
    <collision>
      <geometry><box size="0.4 0.2 0.1"/></geometry>
    </collision>
    <inertial>
      <mass value="2.0"/>
      <inertia ixx="0.01" ixy="0" ixz="0" iyy="0.01" iyz="0" izz="0.01"/>
    </inertial>
  </link>

  <!-- 왼쪽 바퀴 -->
  <link name="left_wheel">
    <visual><geometry><cylinder radius="0.05" length="0.03"/></geometry></visual>
  </link>

  <!-- 본체-바퀴 연결(회전 조인트) -->
  <joint name="left_wheel_joint" type="continuous">
    <parent link="base_link"/>
    <child link="left_wheel"/>
    <origin xyz="0 0.12 0" rpy="1.5708 0 0"/>
    <axis xyz="0 0 1"/>
  </joint>

</robot>
```

> 💡 실무에서는 반복을 줄이려 **xacro**(매크로)를 쓴다. 바퀴처럼 좌우 대칭인 부품을 매크로로
> 정의하면 코드가 짧아진다. 입문에서는 먼저 순수 URDF로 구조를 이해하자.

---

## 8.4 [따라하기] RViz로 모델 확인

물리 시뮬 전에, 모델이 제대로 생겼는지 RViz로 본다. `robot_state_publisher`가 URDF를 읽어
링크들의 좌표(TF)를 발행한다.

```python
# launch/display.launch.py (요지)
from launch import LaunchDescription
from launch_ros.actions import Node
import os
from ament_index_python.packages import get_package_share_directory


def generate_launch_description():
    urdf = os.path.join(
        get_package_share_directory("my_robot_description"),
        "urdf", "my_robot.urdf")
    with open(urdf, "r") as f:
        robot_desc = f.read()

    return LaunchDescription([
        Node(package="robot_state_publisher", executable="robot_state_publisher",
             parameters=[{"robot_description": robot_desc}]),
        Node(package="joint_state_publisher_gui",
             executable="joint_state_publisher_gui"),
        Node(package="rviz2", executable="rviz2"),
    ])
```

RViz에서 `RobotModel` 디스플레이를 추가하고 Fixed Frame을 `base_link`로 두면 로봇이 보인다.

---

### 좌표계와 TF 기초

ROS 2에서 로봇 위치 문제의 절반은 좌표계 문제다. 특히 14장 SLAM과 15장 Nav2는
`map → odom → base_link → 센서 프레임` 연결이 맞아야 정상 동작한다. 여기서 먼저 기본
규칙을 잡고 간다.

ROS의 대표 좌표 규칙은 다음과 같다.

```text
x: 로봇 전방
y: 로봇 왼쪽
z: 위쪽
```

주요 프레임은 역할이 다르다.

| 프레임 | 의미 | 주로 쓰는 곳 |
|---|---|---|
| `base_link` | 로봇 본체 중심 | URDF, 센서 장착 기준 |
| `base_footprint` | 바닥에 투영한 로봇 중심 | 2D 주행 로봇, Nav2 |
| `base_scan` | 라이다 위치 | `/scan` 데이터 기준 |
| `camera_link` | 카메라 본체 위치 | 영상·비전 처리 기준 |
| `odom` | 바퀴 오도메트리 기준의 연속 좌표 | 짧은 시간 주행 추정 |
| `map` | SLAM/Localization이 보정한 전역 지도 좌표 | Nav2 목표 좌표 |

`robot_state_publisher`는 URDF의 link/joint를 읽어 `base_link → base_scan` 같은 정적
관계를 TF로 발행한다. 반대로 `map → odom`은 SLAM이나 localization 노드가 만들고,
`odom → base_link`는 오도메트리 노드가 만든다.

RViz에서 Fixed Frame을 고를 때는 목적을 기준으로 정한다.

- URDF 모양만 볼 때: `base_link`
- 라이다와 로봇 모델 정합을 볼 때: `odom` 또는 `base_link`
- SLAM/Nav2를 볼 때: `map`

정적 센서를 임시로 붙여 볼 때는 `static_transform_publisher`로 프레임을 만들 수 있다.

```bash
ros2 run tf2_ros static_transform_publisher \
  0.12 0 0.18 0 0 0 base_link camera_link

ros2 run tf2_ros tf2_echo base_link camera_link
ros2 run tf2_tools view_frames
```

`Pose`와 `Odometry` 메시지의 `header.frame_id`도 이 기준을 따른다. 예를 들어 15장에서
Nav2 목표를 보낼 때 `goal.pose.header.frame_id = "map"`으로 지정하는 이유는, 목표 좌표가
로봇 기준이 아니라 지도 기준이기 때문이다.

---

## 8.5 [따라하기] Gazebo에 스폰하기

Gazebo Harmonic을 띄우고, 그 안에 로봇을 생성(spawn)한다 — `ros_gz_sim` 기준.

```python
# launch/spawn.launch.py (요지)
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch_ros.actions import Node
from ament_index_python.packages import get_package_share_directory
import os


def generate_launch_description():
    gz_share = get_package_share_directory("ros_gz_sim")
    gz_sim = IncludeLaunchDescription(
        PythonLaunchDescriptionSource(
            os.path.join(gz_share, "launch", "gz_sim.launch.py")),
        launch_arguments={"gz_args": "empty.sdf"}.items(),
    )
    spawn = Node(
        package="ros_gz_sim", executable="create",
        arguments=["-topic", "robot_description", "-name", "my_robot"],
        output="screen",
    )
    return LaunchDescription([gz_sim, spawn])
```

> 📌 위 구조가 gcamp의 Foxy `spawn_entity.launch.py`를 대체하는 **Jazzy 정답 패턴**이다.
> `ros_gz_sim`의 `create`가 Classic 시절의 `spawn_entity.py` 역할을 한다.

### 브리지 — ROS와 Gazebo 잇기

Gazebo 안의 토픽(예: `/cmd_vel`)을 ROS에서 쓰려면 `ros_gz_bridge`를 띄운다.

```python
    bridge = Node(
        package="ros_gz_bridge", executable="parameter_bridge",
        arguments=["/cmd_vel@geometry_msgs/msg/Twist@gz.msgs.Twist"],
    )
```

`@타입@gz타입` 형식으로 ROS 메시지와 Gazebo 메시지를 매핑한다.

---

## 코드 해설 · 실행 확인 · 버전 체크

- **코드 해설 포인트**: URDF link/joint, `robot_state_publisher`, TF 프레임, `ros_gz_sim create`, `parameter_bridge` 연결을 해설한다.
- **실행 확인 포인트**: RViz 모델, TF 트리, `gz sim`, `/clock`, `/cmd_vel`, `/scan` 토픽을 확인한다.
- **버전/환경 체크**: Gazebo Classic의 `gazebo_ros` 방식과 Harmonic의 `ros_gz` 방식을 구분한다.

## 8.6 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| `gz sim` 실행 안 됨 | ros_gz 미설치 | `sudo apt install ros-jazzy-ros-gz` |
| 로봇이 안 보임 | URDF 오류/Fixed Frame 잘못 | RViz Fixed Frame=`base_link`, URDF 문법 확인 |
| 라이다/카메라가 엉뚱한 위치에 보임 | 센서 프레임 TF 누락/오프셋 오류 | `tf2_echo`, `view_frames`로 `base_link → 센서` 확인 |
| 스폰 실패 | Foxy(`spawn_entity.py`) 잔존 | `ros_gz_sim`의 `create`로 교체 |
| cmd_vel이 Gazebo에 안 전달 | 브리지 없음/매핑 오타 | `ros_gz_bridge` 추가, `@` 매핑 확인 |
| 화면이 느림(맥 VM) | GPU 가속 제한 | 단순 월드 사용, 부록 A의 VM 설정 참고 |

---

## 8.7 연습문제

1. `my_robot.urdf`에 오른쪽 바퀴와 캐스터를 추가해 RViz에서 확인하라.
2. 본체 색을 바꾸는 `<material>`을 추가하라.
3. `empty.sdf` 대신 다른 월드 파일로 Gazebo를 띄워 보라.
4. `/cmd_vel` 브리지를 만들고 `ros2 topic pub`로 로봇을 움직여 보라.
5. `static_transform_publisher`로 `base_link → camera_link`를 만들고 RViz TF 디스플레이에서 확인하라.

---

## 8.8 마무리 점검

- [ ] Gazebo Harmonic과 `ros_gz_sim`/`ros_gz_bridge`의 역할을 안다.
- [ ] URDF의 link/joint 구조로 간단한 로봇을 기술했다.
- [ ] ROS 좌표축 규칙과 `map → odom → base_link → 센서` TF 흐름을 설명할 수 있다.
- [ ] RViz로 모델을, Gazebo로 물리 시뮬을 띄웠다.
- [ ] Foxy(Classic) → Jazzy(Harmonic) 스폰·브리지 차이를 안다.

> **1권 끝 / 2권 예고** — 여기까지가 1권 "기초편"이다. 2권 "실전편" 9장부터 지금 만든
> 시뮬레이션 토대 위에서 **주차·복제·미로 탈출** 세 프로젝트를 완성한다.

---

## 8.9 URDF 링크·조인트 더 깊이

8.3의 최소 예제를 실제로 굴리려면 알아야 할 세 가지가 있다.

- **`<visual>` vs `<collision>` vs `<inertial>`**: 보이는 형상 / 충돌 계산용 형상 / 질량·관성.
  시뮬에서 물리가 동작하려면 `<inertial>`(질량·관성)이 **반드시** 있어야 한다(없으면 로봇이
  바닥으로 꺼지거나 튄다).
- **조인트 타입**: `continuous`(무한 회전, 바퀴), `revolute`(각도 제한 회전, 관절),
  `prismatic`(직선 이동), `fixed`(고정, 센서 장착) — 바퀴는 `continuous`.
- **`<origin>`의 rpy**: 부모 기준 자식의 위치(xyz)·회전(roll-pitch-yaw, 라디안). 바퀴를
  옆으로 눕히려 `1.5708`(=π/2)을 준 것이다.

## 8.10 xacro로 반복 줄이기

좌우 바퀴처럼 똑같은 부품은 xacro 매크로로 한 번만 정의한다.

```xml
<xacro:macro name="wheel" params="prefix y_offset">
  <link name="${prefix}_wheel">
    <visual><geometry><cylinder radius="0.05" length="0.03"/></geometry></visual>
  </link>
  <joint name="${prefix}_wheel_joint" type="continuous">
    <parent link="base_link"/><child link="${prefix}_wheel"/>
    <origin xyz="0 ${y_offset} 0" rpy="1.5708 0 0"/>
    <axis xyz="0 0 1"/>
  </joint>
</xacro:macro>

<xacro:wheel prefix="left"  y_offset="0.12"/>
<xacro:wheel prefix="right" y_offset="-0.12"/>
```

런치에서 xacro를 펼쳐 `robot_description`으로 넘긴다(`xacro` 명령 또는 `Command` 사용).

## 8.11 흔한 함정 — Foxy 자료를 그대로 따라 할 때

> ⚠️ 인터넷의 ROS 2 Gazebo 튜토리얼 상당수가 아직 **Foxy/Classic** 기준이다. 다음이 보이면
> Jazzy에서 그대로 안 되니 §8.2·8.5로 치환한다.
> - `gazebo_ros`, `libgazebo_ros_*` 플러그인 → `ros_gz` / `gz-sim-*-system`
> - `spawn_entity.py` → `ros_gz_sim`의 `create`
> - `<gazebo>` 태그 안의 Classic 플러그인 → Harmonic용 시스템 플러그인

## 8.12 연습문제 해설(요약)

- **1번** 8.10의 xacro `wheel` 매크로로 left/right를 만들고, 캐스터는 작은 `sphere` 링크 +
  `fixed`(또는 `continuous`) 조인트로 추가.
- **2번** `<material name="blue"><color rgba="0 0 1 1"/></material>`를 `<visual>` 안에 넣는다.
- **3번** `gz_args`를 다른 `.sdf`로 바꾸면 된다(예: `gz_args: "shapes.sdf"`).
- **4번** `ros_gz_bridge`로 `/cmd_vel` 매핑 후 `ros2 topic pub /cmd_vel geometry_msgs/msg/Twist
  "{linear: {x: 0.2}}"`. 로봇이 움직이면 브리지가 ROS→Gazebo로 명령을 전달한 것이다.

---

### 참고 자료
- Gazebo Harmonic 공식 — https://gazebosim.org/docs/harmonic
- `ros_gz` — https://github.com/gazebosim/ros_gz
- ROS 2 Jazzy — URDF / robot_state_publisher 튜토리얼
- 대조 코드(MIT): `gcamp_ros2_basic` gcamp_gazebo, `ROS-2-from-Scratch` ch11~13
