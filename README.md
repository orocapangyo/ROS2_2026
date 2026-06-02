# ROS 2로 배우는 로봇 프로그래밍 실습 (2026)

> **기준 환경**: ROS 2 **Jazzy Jalisco** (LTS, ~2029) · **Ubuntu 24.04** · **Gazebo Harmonic**
> 표윤석 박사의 『ROS 2로 시작하는 로봇 프로그래밍』을 이론 뼈대로 삼고,
> [gcamp_ros2_basic](https://github.com/sumilee-pcu/gcamp_ros2_basic)의 Gazebo 실습과
> [turtlebot3](https://github.com/sumilee-pcu/turtlebot3) 패키지를 실습 본체로 결합한 **실습 중심 교재**입니다.

---

## 이 책의 차별점

기존 교재가 *개념과 API* 중심이라면, 이 책은 **빌드되고 굴러가는 엔드투엔드 프로젝트**까지 끌고 갑니다.

- 모든 통신 예제를 **Python과 C++** 양쪽으로 제시
- **Gazebo 시뮬레이션 3대 프로젝트**(주차 / 로봇 복제 / 미로 탈출)로 개념을 통합
- **좌표계/TF**를 URDF·SLAM·Nav2 전반의 연결 개념으로 명시
- **OpenCV 출구 마커 인식**을 Maze 프로젝트에 결합해 센서 융합 입문 제공
- **TurtleBot3 실로봇·SLAM·Nav2**까지 연결
- 원본 실습이 **Foxy(EOL)** 기반이던 것을 **Jazzy로 전면 포팅·검증**

> ⚠️ **포팅 주의**: 원본 gcamp 예제는 ROS 2 Foxy + Gazebo Classic 기반입니다.
> 본 교재의 모든 코드는 Jazzy + Gazebo Harmonic에서 재작성·검증하며, 변경점은 각 장의
> **"Foxy → Jazzy 차이"** 박스에 정리합니다.

---

## 대상 독자

- 대학 학부 3~4학년 / 로봇 입문 개발자
- Linux 기본 명령과 Python 기초 문법을 아는 수준
- 선수지식으로 C++을 권장하나, Python 트랙만으로도 완주 가능

---

## 구성 — 2권 분권

전체 ~360페이지를 **권당 약 180페이지로 2권 분권**합니다. 각 장은 *개념 → 따라하기(실습 단계)
→ 트러블슈팅 → 연습문제* 흐름으로 보강해 분량과 교육효과를 함께 확보합니다.

| 권 | 부제 | 포함 | 레벨 | 목표 |
|---|---|---|---|---|
| **1권** | ROS 2 기초편 | 1~8장 | L1~L3 진입 | 통신 4종 + 시뮬레이션 기초까지 자체 완결 |
| **2권** | ROS 2 실전편 | 9~18장 | L3~L4 | 프로젝트 3종 + TurtleBot3 실로봇 + 고급기능 |

---

## 📘 1권 — ROS 2 기초편 (목표 ~180p)

### 1부. ROS 2 시작하기
- [1장. ROS 2 개요와 개발 환경 구축](docs/01_intro/01_overview_and_setup.md) ✅ *초안* — 목표 12p
  - ROS 1과 ROS 2의 차이 · DDS · 배포판 선택(왜 Jazzy인가)
  - Ubuntu 24.04 / ROS 2 Jazzy 설치 · 환경 변수 · 동작 확인
- [2장. 워크스페이스와 패키지, 빌드 시스템](docs/01_intro/02_workspace_and_colcon.md) — 목표 18p
  - `colcon` 빌드 · `ament` · 워크스페이스 오버레이
  - Python 패키지 vs C++ 패키지 생성

### 2부. ROS 2 핵심 통신 (Python & C++)
- [3장. 토픽 — 발행/구독](docs/02_basics/03_topic.md) — 목표 26p
- [4장. 서비스 — 요청/응답](docs/02_basics/04_service.md) — 목표 24p
- [5장. 액션 — 목표/피드백/결과](docs/02_basics/05_action.md) — 목표 26p
- [6장. 파라미터와 커스텀 인터페이스](docs/02_basics/06_parameter_interface.md) — 목표 22p
- [7장. 런치 시스템](docs/02_basics/07_launch.md) — 목표 20p

### 3부. 시뮬레이션 기초
- [8장. Gazebo Harmonic과 URDF·좌표계/TF 기초](docs/03_simulation/08_gazebo_urdf.md) — 목표 28p

> **1권 소계 ≈ 176p**

---

## 📗 2권 — ROS 2 실전편 (목표 ~180p)

### 4부. 시뮬레이션 통합 프로젝트 ★
- [9장. 프로젝트 1 — 주차(Parking): LaserScan 구독 + Twist 발행](docs/03_simulation/09_project_parking.md) — 목표 22p
- [10장. 프로젝트 2 — 로봇 복제(Spawn): Gazebo 서비스](docs/03_simulation/10_project_spawn.md) — 목표 18p
- [11장. 프로젝트 3 — 미로 탈출(Maze Escape): 액션 + Odometry + OpenCV 출구 마커](docs/03_simulation/11_project_maze.md) — 목표 26p

### 5부. 실로봇 — TurtleBot3
- [12장. TurtleBot3 구성과 Bringup](docs/04_turtlebot3/12_bringup.md) — 목표 14p
- [13장. 텔레오퍼레이션과 시뮬레이션](docs/04_turtlebot3/13_teleop_sim.md) — 목표 14p
- [14장. Cartographer로 SLAM](docs/04_turtlebot3/14_slam.md) — 목표 20p
- [15장. Nav2 자율주행](docs/04_turtlebot3/15_nav2.md) — 목표 24p

### 6부. 심화
- [16장. QoS와 통신 신뢰성](docs/05_advanced/16_qos.md) — 목표 14p
- [17장. Lifecycle 노드와 Component](docs/05_advanced/17_lifecycle_component.md) — 목표 16p
- [18장. 로깅 · CLI · rqt 디버깅](docs/05_advanced/18_logging_cli_rqt.md) — 목표 14p

> **2권 소계 ≈ 182p**

### 부록
- [부록 A. 맥(Apple Silicon)에서 ROS 2 Jazzy 실습 환경 구축](docs/06_appendix/A_mac_setup.md)
- [부록 B. ROS 2 명령어 치트시트 & 트러블슈팅 총정리](docs/06_appendix/B_cli_troubleshooting.md)

---

## 산출물 (DOCX)

원고는 마크다운으로 집필하고, 자작 변환기로 DOCX를 생성합니다(신국판 152×225mm).

- 장별 DOCX: [`docx/`](docx/) — 18개 장 + 부록 2편
- 합본 DOCX: [`docx_volumes/`](docx_volumes/) — `ROS2교재_1권_기초편.docx`, `ROS2교재_2권_실전편.docx`
- 변환기: [`tools/md2docx.js`](tools/md2docx.js) — `node tools/md2docx.js <md> <docx>`

> 현재 분량(신국판 기준): 1권 ≈ 59p, 2권 ≈ 62p. 목표(권당 ~180p) 대비 추가 보강 진행 중
> (따라하기 단계 세분화·워크드 예제·그림 추가). 그림/스크린샷은 저작권 안전을 위해 직접 제작.

---

## 실습 코드

장별 완성 코드는 [`code/`](code/) 디렉터리에 ROS 2 패키지 단위로 정리합니다.
(원본: [gcamp_ros2_basic](https://github.com/sumilee-pcu/gcamp_ros2_basic) → Jazzy 포팅본)

## 라이선스 / 출처

- 본문: 집필 중 (저자: 이수미)
- 실습 코드 원본: gcamp_ros2_basic, ROBOTIS turtlebot3 (각 원저작권 표기 준수)
- 이론 참고: 표윤석·임태훈, 『ROS 2로 시작하는 로봇 프로그래밍』, 루비페이퍼
