# ACC — Adaptive Cruise Control (학습용 Full-Stack 프로젝트)

> 🏆 **SeSAC 팀별 프로젝트 1등 우수상 수상** (2026-04-30)

차량용 ACC(Adaptive Cruise Control) 시스템을, **요구공학(StrictDoc) → AUTOSAR Classic MBD → 임베디드 보드 → HMI/센서퓨전** 까지 한 번에 엮어 만든 학습 프로젝트입니다. 5인 팀, 약 4 개월(2026-01 ~ 2026-04) 진행. 양산 제품은 아니지만 **Automotive SPICE 4.0 + ISO 26262:2018** 의 일부를 tailoring 해서 적용했습니다.

---

## 시스템 한눈에 보기

3 개 컴퓨팅 노드가 **CAN @ 500 kbit/s** 로 연결됩니다.

```
┌──────────────────┐    SENSOR_FUSION / VEH_CTRL / ACC_CTRL    ┌──────────────────┐    MTR_CMD     ┌──────────────────┐
│   Raspberry Pi 5 │ ────────────────────────────────────────► │  NXP MPC5606B    │ ─────────────► │  Arduino Uno ×2  │
│  (HMI · Fusion)  │                                           │  (Main ECU,      │                │  (Motor Drivers) │
│  PyQt5 / OpenCV  │ ◄──────────────────────────────────────── │   AUTOSAR SWC)   │ ◄───────────── │  4 motors + Enc  │
│  YOLO + RPLiDAR  │   ACC_STATUS / MTR_SPD_FB / Heartbeats    │  Stateflow FSM   │  MTR_SPD_FB    │  Motor Shield ×2 │
└──────────────────┘                                           └──────────────────┘                └──────────────────┘
```

- **State machine**: `OFF → STANDBY → CRUISING / FOLLOWING → FAULT`
- **Cascaded 제어**: 외부 거리 PID(20 ms) → 내부 속도 PI(10 ms) → 4륜 PWM
- **Safety**: 노드별 독립 Heartbeat, 30 ms 타임아웃 → safe-state, ASIL-B 항목(SAF018) 등 ISO 26262 tailoring 반영

상세 아키텍처 / CAN 메시지 사양 / 요구사항 추적은 `ACC-docs/` (StrictDoc `.sdoc`) 와 `CLAUDE.md` 참고.

---

## 레포 구성 (메타 + 5 서브모듈)

이 레포는 **Git 메타레포** 입니다. 실제 코드/문서는 5 개의 독립 서브모듈에 들어 있습니다.

| 서브모듈 | 역할 | 주요 스택 | 타깃 HW |
|---|---|---|---|
| [`ACC-CANDB/`](ACC-CANDB) | CAN 메시지 정본 (DBC + 시그널 명세) | Vector CANdb++, Markdown | — |
| [`ACC-docs/`](ACC-docs) | 요구사항(STK/SYS/SWR/SAF/HWR), ASPICE/ISO 26262 산출물, 아키텍처 | StrictDoc, Mermaid | — |
| [`ACC-autosar/`](ACC-autosar) | AUTOSAR Classic SWC 5종 (제어 로직) | MATLAB R2024b, Simulink, AUTOSAR Blockset, Embedded Coder, Stateflow | NXP MPC5606B |
| [`ACC-Arduino/`](ACC-Arduino) | CAN↔I2C 게이트웨이 + 모터 4개·엔코더 (2-보드 분리) | Arduino C++, DFRobot MCP2515, Motor Shield R3 ×2 | Arduino Uno ×2 |
| [`ACC-RPI/`](ACC-RPI) | HMI GUI · 카메라/LiDAR 퓨전 · CAN 브릿지 | Python 3.14+, PyQt5, OpenCV, Ultralytics YOLO, python-can | Raspberry Pi 5 |

> 각 서브모듈은 자체 `CLAUDE.md` / `README.md` 를 갖고 독립적으로 빌드·커밋됩니다.

### Clone

```bash
git clone --recurse-submodules https://github.com/gaepo-japcho/ACC.git
# 또는 이미 클론한 경우
git submodule update --init --recursive
```

---

## 했던 것 (요약)

### 1. 요구공학 / 추적성
- **5 계층 요구사항**: Stakeholder(STK) → System(SYS) → Software(SWR) / Safety(SAF) / Hardware(HWR), 약 110 개 ID
- StrictDoc 으로 `.sdoc` 작성 → HTML 자동 생성, 코드 주석에 `SWR###` 태그를 박아 **양방향 추적성** 확보
- ASPICE SWE.1~SWE.6 / SUP.1 일부 산출물 작성 (`ACC-docs/ASPICE/`)
- ISO 26262 HARA 결과로 ASIL-A/B 항목 식별 → 의도치 않은 가속, 브레이크 실패, ECU heartbeat 손실 등에 safe-state 정의

### 2. AUTOSAR Classic MBD
- **ACC Composition** 1 개 안에 **5 SWC** 가 묶여 동작:

  | SWC | Runnable (주기) | 역할 |
  |---|---|---|
  | `AccStateMachine` | `RE_FsmStep` (10 ms) | Stateflow FSM — `OFF/STANDBY/CRUISING/FOLLOWING/FAULT` 전이, setpoint 산출 |
  | `DistanceControl` | `RE_PidStep` (20 ms) | 외부 거리 PID — `TargetSpeedCmS` 산출 (FOLLOWING) / passthrough (CRUISE) |
  | `MotorControl` | `RE_MotorCtrlStep` (10 ms) | 내부 속도 PI + FF — 4륜 PWM (`SET_PWM_LF/RF/LR/RR`) |
  | `HmiManager` | `RE_HmiStep` (50 ms) | 4 boolean 버튼 edge-detect, 운전자 요청 분기 |
  | `Diagnostics` | `RE_FaultDetect` (10 ms) | 노드 HB freeze 검출, FaultActive/FaultCode 집계 |

- **Composition 인터페이스**: bus-binding 8 개 (CAN ↔ SWC 직결: `MTR_SPD_FB`, `SENSOR_FUSION`, `VEH_CTRL`, `SENSOR_HB`, `MTR_HB`, `ACC_CTRL` 수신 / `MTR_CMD`, `ACC_STATUS`, `ECU_HB` 송신) + intra-ECU assembly 7 개 (`IF_CtrlOutput`, `IF_AccStatus`, `IF_DriverReq`, `IF_DiagStatus` 등)
- **별도 게이트웨이 SWC 없음** — 응용 SWC 가 mobilgene Rte 를 통해 DBC 시그널과 직접 매핑 (35 매핑)
- 모델 자체는 `.m` 스크립트로 **재생성 가능한 코드** 로 관리 (`rebuild_all.m`) — `.slx`/`.sldd` 는 generated artifact 취급
- Embedded Coder 로 C 코드 자동 생성, MPC5606B 상용 BSP 와 통합

### 3. 임베디드 (Arduino)
- 처음에는 앞/뒤 Arduino 분리였으나, **CAN 게이트웨이 ↔ 모터 드라이버 분리** 로 리팩터링
- 이유: Uno 핀맵에서 MCP2515 (SPI D10–D13) 와 Motor Shield (D11–D13) 가 충돌 → I2C(0x10) 로 분리
- DBC 4륜 PWM 은 게이트웨이에서 좌/우 평균으로 축소해 모터 노드에 전달 (어댑터 패턴)

### 4. HMI / 센서퓨전 (Raspberry Pi)
- PyQt5 기반 클러스터 UI (속도, 차간거리, ACC 상태, 경고)
- USB 카메라 + YOLO(Ultralytics) 로 차량/보행자 검출 → RPLiDAR 거리와 융합
- python-can 으로 CAN 송수신, **버스 미연결 시 자동 simulation fallback** — 데모/개발 편의

---

## 결과 / 회고

- ✅ HIL 데모로 CRUISE/FOLLOW/FAULT 전이 검증 완료
- ✅ ECU Heartbeat 손실 / MTR_CMD 타임아웃 → safe-state 동작 검증
- ✅ 요구사항 ID 가 코드 주석까지 살아 있어, 변경 영향 분석이 실제로 가능했음
- ⚠️ **남은 TODO** (학습 프로젝트 한계):
  - `MTR_CMD` E2E (RC/CRC) 미적용 — DBC 자리만 예약 (SAF010)
  - 거리/속도 PID 게인 HIL 튜닝 미세조정 (`TODO_TUNE`)
  - `Diagnostics` SWC 가 stub 상태 (DTC 저장/보고 일부만 구현)

학습한 점을 한 줄씩:

| 영역 | 배운 것 |
|---|---|
| 요구공학 | StrictDoc + 코드 주석 ID 태깅이 추적성을 살아있게 만든다 — "문서를 위한 문서" 를 피하는 가장 실용적인 방법 |
| AUTOSAR | `.slx` 를 손으로 만지면 안 되는 이유를 몸으로 배움. `.m` 스크립트로 모델을 **재생성 가능** 하게 짜야 팀 협업이 가능 |
| 안전 | "타임아웃 → safe-state" 같이 단순한 규칙이 ASIL-B 의 9 할. 화려한 진단보다 **확실한 fallback** 이 우선 |
| CAN 설계 | 메시지 분리(`VEH_CTRL` vs `ACC_CTRL`) 와 signal-level multicast 가 ECU/SENSOR 양쪽 부하를 동시에 낮춤 |
| 통합 | DBC 를 **단일 진실의 원천(Single Source of Truth)** 으로 두고 모든 노드가 codec 자동 생성 — 사람이 비트 시프팅 하지 않게 |

---

## 팀

**5 인 팀**, SeSAC 부트캠프, 2026-01 ~ 2026-04.

| 멤버 | 주 기여 영역 | 주요 책임 |
|---|---|---|
| **Park JongBeum** ([@parkjbdev](https://github.com/parkjbdev)) | ACC-docs · ACC-autosar · ACC-CANDB · ACC-RPI | 시스템/PM — 요구사항(SYS/SWR/SAF/HWR), HARA, 아키텍처, AUTOSAR Composition, CAN DBC, 일정 관리 |
| **BAE-Jungwoo** ([@BAE-Jungwoo](https://github.com/BAE-Jungwoo)) | ACC-autosar | SW — AUTOSAR SWC 구현 (제어 로직), Embedded Coder 통합 |
| **Wis3754** ([@Wis3754](https://github.com/Wis3754)) | ACC-autosar · ACC-Arduino · ACC-CANDB · ACC-RPI | SW/HW 통합 — AUTOSAR · 보드 브링업 · CAN 시그널 정합 |
| **Yurim Kim** ([@yurimdl1](https://github.com/yurimdl1)) | ACC-RPI · ACC-docs | SW — HMI(PyQt5) · 카메라/LiDAR 센서퓨전 · YOLO 통합 |
| **dahee** ([@ekgml4573](https://github.com/ekgml4573)) | ACC-Arduino · ACC-autosar | HW/SW — Arduino 모터 노드 · CAN 게이트웨이 · 엔코더 |

> **독립성 원칙 (ISO 26262 Part 2, Independence Level I1)**: 안전 산출물(SAF·ASIL 표시 SWR)은 작성자 ≠ 리뷰어 원칙으로 cross-review 진행.

GitHub Org: [gaepo-japcho](https://github.com/gaepo-japcho)

---

## 라이선스 / 면책

학습·포트폴리오 목적으로 공개합니다. 양산/실차 적용을 의도하지 않았으며, 사용에 따른 결과는 사용자 책임입니다.
