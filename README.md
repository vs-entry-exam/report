# 입사 과제 **최종 결과 보고서** 



## 1. 과제 개요 (Introduction)

### 1) 과제 목표 요약

* **Step 1 — RAG Q\&A 웹앱:** 문서 업로드 → 임베딩/인덱싱 → **문서 기반 질의응답**을 제공하는 RAG 웹 서비스 구축. (Next.js + FastAPI + ChromaDB) 
* **Step 2 — AGV/Cobot 시뮬레이션:** Gazebo 상에서 **AGV(Nav2+SLAM)**, \*\*Cobot(MoveIt2)\*\*를 구동하고, **VLA 학습용 주행/조작 데이터 수집 기반** 마련. 
* **Step 3 — 3D 맵핑/환경 인식:** 시뮬레이션 3D LiDAR → **OctoMap** 생성 및 RViz 시각화 파이프라인 구축. 

### 2) 최종 결과물 요약 (한두 문장씩)

* **Step 1:** 3-Tier 구조(웹/백엔드/벡터DB)로 RAG 체인을 구성해 **출처가 포함된 한국어 답변**을 제공. 
* **Step 2:** **AGV: SLAM Toolbox + Nav2**, **Cobot: MoveIt2** 연동 시나리오를 구현하고, rosbag 기반 **데이터 수집 파이프라인의 골격**을 설계. 
* **Step 3:** **OctoMap 서버**로 3D 점유 격자 지도를 생성하여 **3D 환경 인식의 출발점**을 확보. 3D 정보를 2D 네비게이션으로 확장 적용할 **통합 방안**을 제시. 



## 2. 프로젝트 관리 (Project Management)

### 1) 7일 간트 차트 (예시)

**기간:** 2025-09-15  \~ 2025-09-22 (7일)

| 작업                 |  15 |  16 |  17 |  18 |  19 |  20 |  21 |  22 |
| -------------------- | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| Step1 설계/구현       |  ■  |  ■  |     |     |     |     |     |     |
| Step2 AGV(Nav2/SLAM) |     |  ■  |  ■  |     |  ■  |     |     |     |
| Step2 Cobot(MoveIt)  |     |     |  ■  |  ■  |  ■  |     |     |     |
| Step3 3D/OctoMap     |     |     |     |     |     |  ■  |  ■  |     |
| 문서/영상 정리        |     |  ■  |     |     |  ■  |     |  ■  |  ■  |

### 2) 개발 일지 

| 일자 | 작업 내용 | 문제/리스크 | 해결/의사결정 |
| -- | ----- | ------ | ------- |
| 15  | step1 설계 및 개발, Ubuntu 22.04, ROS2 Humble setting |       |        |
| 16  | robot pkgs build, agv tutorial | 빌드 에러: 메모리 부족, 실로봇 부재로 scan 데이터 부재, lidar에 로봇 몸체가 걸리는 문제, tf 불일치로 scan 토픽을 slam 및 nav2 에서 활용 불가      | 병렬 빌드 패키지 수 제한, gazebo plugin 으로 lidar 구동, urdf 수정, gazebo와 rviz에 활용하는 urdf 통일       |
| 17  | agv gazebo + slam + nav2 통합, cobot tutorial      | gazebo에서 cobot 스폰시 충돌 발생,       | urdf 수정, gazebo와 rviz에 활용하는 urdf 통일       |
| 18  | cobot gazebo + moveit(motion planning) 통합    | tf 불일치      | urdf 수정, gazebo와 rviz에 활용하는 urdf 통일       |
| 19  | agv/cobot 최종 점검     | 환경 변수 문제      | bashrc에 ws기준 source install/setup.bash 명확하게 수정       |
| 20  | gazebo에서 3D lidar 설정     | 3D lidar 스캔시 빈 공간      | lidar 플러그인 sample 수를 올려서 더 촘촘하게 스캔       |
| 21  | octomap 적용     | octomap_binary 토픽을 받아줄 rviz display 부재      | 플러그인 설치       |
| 22  | 문서 정리     |       |        |


## 3. 주요 기술 결정 및 구현 내용 (Technical Deep Dive)

### 1) 공통 아키텍처/선택 이유

* **Step 1 (RAG 웹앱)**

  * **3-Tier**(Next.js 프론트 / FastAPI 백엔드 / ChromaDB): 확장성과 모듈성, LLM/임베딩 **교체 용이성** 확보.
  * **LLM·임베딩 가변 구성**(OpenAI ↔ Ollama): 비용/성능 **트레이드오프 대응** 및 오프라인/로컬 실행 지원.
  * **Chunking 파라미터 명시**(예: size 1000, overlap 150): 검색-재구성 품질과 지연시간의 **균형 최적화**.

* **Step 2 (AGV/Cobot)**

  * **AGV:** SLAM Toolbox + Nav2(맵/코스트맵/플래너/컨트롤러) 표준 스택 → **재현성**과 **튜닝 생태계** 활용.
  * **Cobot:** MoveIt2를 통해 **계획→실행 파이프라인** 확립(조인트 상태/트라젝토리 확보).

* **Step 3 (3D 맵핑)**

  * **OctoMap** 선택: 3D 점유 격자 기반으로 **메모리 효율**과 **질의속도** 확보, 3D 장애물 표현 용이.

### 2) 핵심 기능 구현 방식

* **Step 1 — RAG 체인**

  * 업로드(PDF/MD/TXT) → 임베딩 → **ChromaDB 인덱싱** → 질의시 **관련 문서 retrieval + LLM 생성** → **출처(문서/페이지) 표기**.
  * 환경변수로 **LLM/임베딩/프롬프트 파일**을 스위칭 가능(운영/개발 분리 용이).

* **Step 2 — Nav2/SLAM & MoveIt 연동**

  * **AGV:** Gazebo 월드 로드 → SLAM Toolbox로 **지도 생성** → Nav2로 **목표점 네비게이션**. 주요 토픽(`/scan`, `/odom`, `/tf`, `/cmd_vel` 등) **수집 기준 정립**.
  * **Cobot:** Gazebo(myCobot 280) + MoveIt2로 **플래닝 결과를 시뮬레이션에 반영**, **조인트/트라젝토리** 로깅 준비.

* **Step 3 — OctoMap 파이프라인**

  * Gazebo 3D LiDAR(레이 센서) → **PointCloud 발행** → `octomap_server`로 **3D 점유맵 생성** → RViz 시각화.



## 4. 프로젝트별 회고 및 개선 방향 (Retrospective & Improvements)

### Step 1: RAG Q\&A 웹 애플리케이션

* **미비점**

  * 사용자 인증/권한 관리 부재 
  * 스트리밍 응답/진행률 표시 등 **UX 디테일 부족** 

* **개선 방향**

  * **JWT 기반 인증** 및 사용자별 문서 스코프 분리 
  * **WebSocket 스트리밍**, 업로드/인덱싱 **프로그레스 바** 도입 

### Step 2: AGV/Cobot 시뮬레이션

* **미비점**

  * **데이터 파이프라인 자동화 미완**(rosbag → 학습 포맷 변환) 
  * Omni-drive 특성 반영 불충분(컨트롤러/플래너 튜닝 필요) 
  * 웹 기반 시뮬레이션 미구현

* **개선 방향**

  * **자동 로깅/전처리 스크립트** 제작(토픽 리스트/주기/압축/메타데이터) 
  * Nav2 **컨트롤러 교체/파라미터 튜닝**(경로 추종/속도 제한/곡률 보상 등) 
  * WebRTC 기반 저지연 스트리밍 + rosbridge(WebSocket) + React 대시보드(조작·상태 모니터)로 시뮬레이터 화면/토픽을 웹에 실시간 노출하고 기본 원격 제어까지 지원

### Step 3: 3D 맵핑 및 환경 인식

* **미비점**

  * OctoMap이 **시각화 중심**으로만 활용, **플래닝 연동 부재** 
  * 대규모 환경에서 **성능 저하 가능성** 

* **개선 방향**

  * **3D 코스트맵 레이어 통합**으로 공중장애물 회피까지 지원 
  * **해상도/깊이 파라미터 최적화**, 멀티스레딩으로 처리량 개선 

## 5. 결론 및 제언 (Conclusion & Contribution)

### 1) 과제 요약 및 의의

* 본 과제는 **(i)** 문서 지식 접근성을 높이는 **RAG 웹앱**, **(ii)** 실로봇에 가까운 **AGV/Cobot 시뮬레이션 기반**, **(iii)** 3D **환경 인식 파이프라인**을 구현 했흡니다.

### 2) 참고 링크 (README/코드/영상)

* **Step 1 (RAG 웹앱)**

  * [시연 영상](https://www.youtube.com/watch?v=r_hEyD05s1g)
  * [GitHub](https://github.com/vs-entry-exam/step1)

* **Step 2 (AGV/Cobot)**

  * [AGV 시연](https://www.youtube.com/watch?v=KWQHvcB6-xM)
  * [Cobot 시연](https://www.youtube.com/watch?v=ChGDlB8bcLQ)
  * [GitHub](https://github.com/vs-entry-exam/step2)

* **Step 3 (3D 맵핑)**

  * [GitHub](https://github.com/vs-entry-exam/step3)

---

## 부록. 실행/재현 가이드 (필요 시 첨부)

### Step 1 — RAG 웹앱 (요약)

```bash
# Linux/macOS
chmod +x scripts/dev.sh
./scripts/dev.sh --all   # 백엔드(FastAPI) + 프론트(Next.js) 동시 실행
```

### Step 2 — 시뮬레이션 런치

```bash
# AGV: Gazebo + SLAM + Nav2
source ~/step2/install/setup.bash
ros2 launch myagv_pro_bringup sim_slam_nav2.launch.py

# Cobot: Gazebo + MoveIt2
source ~/step2/install/setup.bash
ros2 launch mycobot_280_bringup mycobot_gazebo_moveit.launch.py
```

### Step 3 — 3D 맵 & OctoMap

```bash
source ~/step3/install/setup.bash
ros2 launch agv_pro_gazebo agv_pro_gazebo.launch.py
# RViz에서 OctoMap/Occupancy 관련 토픽 추가
```