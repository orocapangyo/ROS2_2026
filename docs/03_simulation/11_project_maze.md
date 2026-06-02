# 11장. 프로젝트 3 — 미로 탈출(Maze Escape)

> **학습 목표 / 통합 개념**: 액션(5장) + 토픽(3장) + Odometry + OpenCV
> - 액션으로 "미로를 빠져나가라"는 목표를 주고 진행을 피드백받는다.
> - 라이다·오도메트리를 결합해 벽 따라가기(wall-following)를 구현한다.
> - 막히거나 위험할 때 목표를 취소한다.
> - OpenCV로 카메라 영상에서 출구 마커를 인식한다.

> **이번 장의 산출물**
> - Maze Escape 액션 서버/클라이언트를 작성한다.
> - LaserScan, Odometry, OpenCV 출구 마커 인식을 하나의 프로젝트로 통합한다.
>
> **공통 학습 흐름**: 개념 → 따라하기 → 코드 해설 → 실행 확인 → 버전/환경 체크 → 트러블슈팅 → 연습문제 → 마무리 점검

2권 시뮬레이션의 정점. 토픽·서비스·액션을 한 프로젝트에서 통합한다.

---

## 11.1 무엇을 만드나

미로 월드에서 로봇이 벽을 따라 이동해 출구로 나간다. 구성:

```text
액션:  /escape_maze   (커스텀)   목표=출발, 피드백=진행거리, 결과=탈출 여부
구독:  /scan          LaserScan  벽까지 거리(좌/전/우)
구독:  /odom          Odometry   현재 위치·이동 거리
발행:  /cmd_vel        Twist      주행 명령
```

---

## 11.2 커스텀 액션 정의 (6장 방식)

`my_robot_interfaces/action/EscapeMaze.action`:

```text
bool start                 # goal: 시작 신호
---
bool escaped               # result: 탈출 성공 여부
float64 total_distance     # result: 이동 총거리
---
float64 traveled           # feedback: 현재까지 이동거리
string state               # feedback: 현재 상태(직진/회전 등)
```

빌드 후 `ros2 interface show my_robot_interfaces/action/EscapeMaze`로 확인.

---

## 11.3 벽 따라가기 알고리즘

가장 단순하고 견고한 전략은 **오른손 법칙(우벽 추종)**: 오른쪽 벽을 일정 거리로 유지하며
따라간다.

```text
- 오른쪽이 너무 멀다     → 오른쪽으로 약간 회전(벽에 붙는다)
- 오른쪽이 적당하고 전방 열림 → 직진
- 전방이 막혔다           → 왼쪽으로 회전
```

라이다 `ranges`에서 전방·우측 인덱스를 뽑아 위 규칙을 적용한다.

---

## 11.4 [따라하기] 액션 서버 (요지)

5.3절 서버 구조에 위 제어 루프를 넣고, 오도메트리로 이동거리를 누적해 피드백한다.

```python
import math
import rclpy
from rclpy.node import Node
from rclpy.action import ActionServer
from sensor_msgs.msg import LaserScan
from nav_msgs.msg import Odometry
from geometry_msgs.msg import Twist
from my_robot_interfaces.action import EscapeMaze


class MazeServer(Node):
    def __init__(self):
        super().__init__("maze_server")
        self.scan_ = None
        self.start_pos_ = None
        self.traveled_ = 0.0
        self.cmd_pub_ = self.create_publisher(Twist, "cmd_vel", 10)
        self.create_subscription(LaserScan, "scan", self.on_scan, 10)
        self.create_subscription(Odometry, "odom", self.on_odom, 10)
        self.server_ = ActionServer(
            self, EscapeMaze, "escape_maze", execute_callback=self.execute)

    def on_scan(self, msg):
        self.scan_ = msg

    def on_odom(self, msg):
        p = msg.pose.pose.position
        if self.start_pos_ is None:
            self.start_pos_ = (p.x, p.y)
        self.traveled_ = math.dist((p.x, p.y), self.start_pos_)

    def execute(self, goal_handle):
        feedback = EscapeMaze.Feedback()
        rate = self.create_rate(10)
        while rclpy.ok():
            if goal_handle.is_cancel_requested:
                goal_handle.canceled()
                return EscapeMaze.Result()

            cmd = Twist()
            if self.scan_ is not None:
                front = self.scan_.ranges[len(self.scan_.ranges)//2]
                right = self.scan_.ranges[len(self.scan_.ranges)//4]
                if front < 0.5:                      # 전방 막힘 → 좌회전
                    cmd.angular.z = 0.6
                    feedback.state = "turn_left"
                elif right > 0.8:                    # 우벽 멀다 → 우회전
                    cmd.angular.z = -0.4
                    cmd.linear.x = 0.15
                    feedback.state = "follow_right"
                else:                                # 직진
                    cmd.linear.x = 0.2
                    feedback.state = "forward"
            self.cmd_pub_.publish(cmd)

            feedback.traveled = self.traveled_
            goal_handle.publish_feedback(feedback)

            if self.escaped():                       # 탈출 판정(아래)
                break
            rate.sleep()

        goal_handle.succeed()
        result = EscapeMaze.Result()
        result.escaped = True
        result.total_distance = self.traveled_
        return result

    def escaped(self):
        # 예: 사방이 충분히 열리면 출구로 간주(월드에 맞게 조정)
        if self.scan_ is None:
            return False
        return min(self.scan_.ranges) > 1.5
```

> 📌 `escaped()` 판정은 미로 월드 형태에 따라 조정해야 한다. 출구 지점에 마커를 두고
> 오도메트리 좌표로 판정하는 방법도 좋다.

---

## 11.5 [따라하기] 클라이언트로 목표 보내기

5.4절 클라이언트 구조 그대로, `EscapeMaze` 목표를 보내고 피드백·결과를 출력한다.

```bash
ros2 action send_goal /escape_maze my_robot_interfaces/action/EscapeMaze \
  "{start: true}" --feedback
```

피드백으로 `traveled`(이동거리)와 `state`(직진/회전)가 실시간으로 흐른다. 막히면 `Ctrl+C`
또는 취소 요청으로 즉시 멈춘다.

---

## 11.6 OpenCV로 출구 마커 인식하기

라이다와 오도메트리만으로도 미로 주행은 가능하지만, "출구"라는 의미 있는 장면을 인식하려면
카메라가 유용하다. 이 절에서는 빨간 출구 마커를 예로 들어 `sensor_msgs/Image`를 OpenCV
영상으로 바꾸고, HSV 색 마스크로 마커를 찾는다.

설치:

```bash
sudo apt install -y ros-jazzy-cv-bridge python3-opencv
```

의존성:

```xml
<!-- package.xml -->
<depend>sensor_msgs</depend>
<depend>cv_bridge</depend>
<exec_depend>python3-opencv</exec_depend>
```

카메라 토픽은 Gazebo/Harmonic 월드 구성에 따라 이름이 다를 수 있다. 먼저 토픽을 확인한다.

```bash
ros2 topic list | grep image
ros2 topic info /camera/image_raw
```

빨간색 마커 검출 노드는 다음 흐름을 갖는다.

```python
import cv2
import rclpy
from rclpy.node import Node
from cv_bridge import CvBridge
from sensor_msgs.msg import Image
from std_msgs.msg import Bool


class ExitMarkerDetector(Node):
    def __init__(self):
        super().__init__("exit_marker_detector")
        self.bridge = CvBridge()
        self.seen_pub = self.create_publisher(Bool, "exit_marker_seen", 10)
        self.create_subscription(Image, "camera/image_raw", self.on_image, 10)

    def on_image(self, msg):
        frame = self.bridge.imgmsg_to_cv2(msg, "bgr8")
        hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)

        lower1 = (0, 80, 80)
        upper1 = (10, 255, 255)
        lower2 = (170, 80, 80)
        upper2 = (180, 255, 255)
        mask = cv2.inRange(hsv, lower1, upper1) | cv2.inRange(hsv, lower2, upper2)

        contours, _ = cv2.findContours(mask, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
        area = max((cv2.contourArea(c) for c in contours), default=0.0)
        seen = area > 1200.0

        self.seen_pub.publish(Bool(data=seen))
        if seen:
            self.get_logger().info(f"출구 마커 감지: area={area:.0f}")


def main():
    rclpy.init()
    rclpy.spin(ExitMarkerDetector())
    rclpy.shutdown()
```

Maze 액션 서버는 `exit_marker_seen`을 구독해 `escaped()` 판정에 반영한다.

```python
from std_msgs.msg import Bool

# __init__ 안에서:
self.exit_seen_ = False
self.create_subscription(Bool, "exit_marker_seen", self.on_exit_seen, 10)

def on_exit_seen(self, msg):
    self.exit_seen_ = msg.data

def escaped(self):
    return self.exit_seen_
```

이제 미로 출구 근처에 빨간 표식을 두면, 주행 로직은 라이다로 벽을 따라가고 탈출 판정은
OpenCV 마커 인식으로 수행한다. 이 구조가 센서 융합의 가장 작은 형태다.

> 🔁 **Foxy → Jazzy 메모**: `cv_bridge`·OpenCV 연동 API는 거의 동일하다. 다만 의존 패키지
> 이름·버전(예: `python3-opencv`)을 Jazzy 기준으로 설치하고, 카메라 토픽은 Harmonic
> 브리지로 노출한다.

---

## 코드 해설 · 실행 확인 · 버전 체크

- **코드 해설 포인트**: action goal/feedback/result, LaserScan/Odometry 제어 루프, OpenCV HSV 마스크, cancel 처리를 해설한다.
- **실행 확인 포인트**: goal 전송, feedback 관찰, `exit_marker_seen` 토픽, cancel, 미로 탈출 동작을 확인한다.
- **버전/환경 체크**: Jazzy 기준 custom action 빌드, `cv_bridge`, Gazebo bridge topic 구성을 점검한다.

## 11.7 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| 로봇이 벽에 박음 | 임계값/회전속도 부적절 | front/right 임계, `angular.z` 튜닝 |
| traveled가 0 | `/odom` 미수신 | 브리지·토픽 확인 |
| 무한 회전 | 규칙 충돌 | 전방·우측 조건 우선순위 점검 |
| 취소가 안 됨 | `is_cancel_requested` 미검사 | 루프에 취소 체크 추가 |
| `create_rate` 멈춤 | 단일 스레드 spin | 멀티스레드 executor 사용 검토 |
| OpenCV 마커가 감지되지 않음 | 카메라 토픽/HSV 범위/조명 문제 | `ros2 topic echo exit_marker_seen`, HSV 범위와 면적 임계값 조정 |

---

## 11.8 연습문제

1. 좌벽 추종(왼손 법칙)으로 바꿔 동작을 비교하라.
2. 피드백 `state`를 화면 대신 별도 토픽으로도 발행해 `rqt`로 관찰하라.
3. 5초간 이동거리 변화가 없으면 "막힘"으로 보고 자동 취소하라.
4. OpenCV로 빨간 출구 마커를 인식해 탈출 판정에 사용하라.

---

## 11.9 마무리 점검

- [ ] 커스텀 액션으로 장시간 작업을 목표·피드백·결과로 모델링했다.
- [ ] 라이다+오도메트리로 벽 따라가기를 구현했다.
- [ ] OpenCV 출구 마커 인식을 `escaped()` 판정에 연결했다.
- [ ] 취소로 안전하게 중단할 수 있다.
- [ ] 토픽·서비스·액션을 한 프로젝트에서 통합해 보았다.

> **3부(시뮬레이션) 끝 / 다음 예고** — 12장부터 **실로봇 TurtleBot3**다. 시뮬에서 검증한
> 개념을 실제 하드웨어와 표준 패키지(SLAM·Nav2)로 옮긴다.

---

## 11.10 [워크드 예제] 막힘 감지 자동 취소

미로에서 로봇이 같은 자리를 맴돌면 사람이 개입해야 한다. 이동거리 변화가 일정 시간 없으면
"막힘"으로 보고 자동 취소하는 안전장치를 서버 루프에 넣는다.

```python
import time

def execute(self, goal_handle):
    feedback = EscapeMaze.Feedback()
    last_dist = 0.0
    last_progress_t = time.time()
    rate = self.create_rate(10)
    while rclpy.ok():
        if goal_handle.is_cancel_requested:
            goal_handle.canceled(); return EscapeMaze.Result()

        # ... (11.4의 벽 따라가기 제어) ...

        # 막힘 감지: 5초간 10cm도 못 나가면 중단
        if self.traveled_ - last_dist > 0.1:
            last_dist = self.traveled_
            last_progress_t = time.time()
        elif time.time() - last_progress_t > 5.0:
            self.get_logger().warn("5초간 진전 없음 → 막힘으로 판단, 중단")
            goal_handle.abort()
            return EscapeMaze.Result()

        feedback.traveled = self.traveled_
        goal_handle.publish_feedback(feedback)
        rate.sleep()
```

`succeed`(성공)·`canceled`(취소)·`abort`(실패) 세 종료 상태를 구분해 쓰는 것이 견고한 액션
설계의 핵심이다.

## 11.11 멀티스레드 실행자 주의

`create_rate(...).sleep()`을 `execute` 콜백 안에서 쓰면, 단일 스레드 실행자에서는 콜백이
스레드를 점유해 `/scan`·`/odom` 콜백이 갱신되지 않는 교착이 생길 수 있다. 해결:

```python
from rclpy.executors import MultiThreadedExecutor
from rclpy.callback_groups import ReentrantCallbackGroup
# 액션 서버·구독을 ReentrantCallbackGroup에 두고 MultiThreadedExecutor로 spin
```

> 📌 "액션은 도는데 `traveled`가 0에서 안 변한다"면 이 문제일 확률이 높다. 멀티스레드
> 실행자로 바꾸면 제어 루프와 센서 콜백이 동시에 돈다.

## 11.12 연습문제 해설(요약)

- **1번** 우/좌를 바꾼다: `right` 인덱스(`n//4`)를 `left`(`3n//4`)로, 회전 부호를 반대로.
- **2번** `feedback.state`를 별도 `String` 토픽으로도 발행하고 `ros2 topic echo`/`rqt`로 관찰.
- **3번** 위 11.10의 막힘 감지 + `abort()`가 정답.
- **4번** `cv_bridge`로 영상을 받아 HSV 변환 후 빨간색 마스크의 면적이 임계 이상이면
  `exit_marker_seen=True`를 발행하고, Maze 서버의 `escaped()`에서 이 값을 사용한다.

---

### 참고 자료
- `nav_msgs/Odometry`, `cv_bridge` 문서
- 대조 코드(MIT): `gcamp_ros2_basic` py_action_pkg `maze_*`, `robot_controller`, `odom_sub`
