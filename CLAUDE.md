# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Workspace layout

`/mnt/c/acc` is **not a single repository** — it is a parent directory holding five independent git repos (편의상 서브모듈로 편입) for one ACC (Adaptive Cruise Control) system. Each subdirectory has its own `.git`, its own toolchain, and is usually worked on in isolation. Run git/build commands inside the relevant subdirectory, not at the top level.

| Subdir | Role | Language / Toolchain | Target HW |
|---|---|---|---|
| `ACC-CANDB/` | CAN 메시지 정본 (DBC + signal 명세) | `.dbc` + Markdown | n/a (진실의 원천) |
| `ACC-docs/` | Requirements, ASPICE/ISO 26262 artifacts, architecture sketches | StrictDoc (`.sdoc`), Markdown, Mermaid | n/a |
| `ACC-autosar/mbd/` | AUTOSAR Classic SWC models (ACC control logic) | MATLAB R2024b + Simulink + AUTOSAR Blockset + Embedded Coder + Stateflow | NXP MPC5606B (main ECU) |
| `ACC-Arduino/` | `acc_can_node` (CAN↔I2C 게이트웨이) + `acc_motor_node` (모터 4 + 엔코더) 2-보드 분리 | Arduino C++ / DFRobot MCP2515 / Motor Shield R3 ×2 | Arduino Uno ×2 |
| `ACC-RPI/` | HMI GUI, 카메라/LiDAR 퓨전, CAN 브릿지 | Python 3.14+ (`uv`), PyQt5, OpenCV, Ultralytics, python-can | Raspberry Pi 5 |

Team of 5, deadline 2026-04-30. Educational / portfolio project — not shipped, but follows **Automotive SPICE 4.0 + ISO 26262:2018** tailoring (see `ACC-docs/guide/README.md`).

## System architecture (big picture)

Three compute nodes talk over CAN @ 500 kbit/s. **진실의 원천 우선순위**: `ACC-CANDB/acc_db.dbc` (+ `ACC-CANDB/README.md`) → `ACC-docs/reqs/STK/SYS.sdoc` (SYS015/016/017/019/025/030/031) + `SAF.sdoc` (SAF004/010/015/018). 제안값과 DBC 가 다를 경우 DBC 우선.

```
[RPi 5 / SENSOR]  ── 0x110 SENSOR_FUSION    (20ms) ──►  [MPC5606B / ECU]  ── 0x210 MTR_CMD       (10ms) ──►  [Arduino / MTR]
   ▲           ── 0x111 SENSOR_HEARTBEAT (20ms) ──►        │                ◄── 0x300 MTR_SPD_FB    (10ms) ──
   │           ── 0x120 VEH_CTRL         (20ms) ──►        │                ◄── 0x310 MTR_HEARTBEAT (10ms) ──
   │           ── 0x510 ACC_CTRL         (50ms) ──►        │                                               
   ◄── 0x300 MTR_SPD_FB.GET_SPD_AVG (10ms, signal multicast) ─┤                                              
   └───────────── 0x520 ACC_STATUS      (50ms) ◄──           └── 0x410 ECU_HEARTBEAT (10ms, ASIL-B) ──► (both)
```

메시지 기능 분리 (DBC v3+): **`VEH_CTRL(0x120)`** 은 차량 직접 조작(브레이크/수동 accel, SAF004/015 ASIL-B 50ms 응답), **`ACC_CTRL(0x510)`** 은 ACC feature 버튼/setpoint, **`ACC_STATUS(0x520)`** 는 ACC 상태 피드백만. 차속은 `MTR_SPD_FB(0x300)` 의 `GET_SPD_AVG` 시그널이 signal-level multicast (AVG→ECU+SENSOR, 4륜→ECU만). 4륜 시그널은 DLC 8 포화 회피를 위해 int12 @ 0.3 cm/s.

Heartbeat 배치 (SYS025): 각 노드별 독립 `<NODE>_HEARTBEAT` 메시지 — `0x111 SENSOR_HEARTBEAT` (20ms), `0x310 MTR_HEARTBEAT` (10ms), `0x410 ECU_HEARTBEAT` (10ms, SAF018 ASIL-B). 모든 HB 메시지 공통 구조: `HB_<node>` (uint8 순환 카운터) + `ERR_<node>` (uint8, 0=Normal).

- **Main ECU (MPC5606B, AUTOSAR Classic)**: ASW 응용 SWC 5개 (`AccStateMachine`, `DistanceControl`, `MotorControl`, `Diagnostics`, `HmiManager`). State machine: `OFF → STANDBY → CRUISING/FOLLOWING → FAULT`. **Cascaded 제어 (v0.6, 2026-04-23)**: `DistanceControl`(20ms, 바깥) 은 FOLLOWING 에서 거리 PID 로 `TargetSpeedCmS` 산출 (CRUISE 에선 `SetSpeedCmS` passthrough), `MotorControl`(10ms, 안쪽) 은 `TargetSpeedCmS` 와 `MTR_SPD_FB.GET_SPD_AVG`(대표 속도) 로 속도 PI (+ FF Lookup, 현재 FF=0) 를 수행해 PWM 출력 (SWR034, SAF008 클램핑). STANDBY 에서는 VEH_CTRL 의 `SET_ACCEL_PWM` (int8) 을 그대로 4륜에 pass-through (PI 적분기 리셋). **CAN I/O 는 응용 SWC 가 mobilgene Rte 를 통해 DBC 시그널과 직결** (Pattern B, 2026-04-25 `experiment/pattern-b-direct-com` 진행 중) — 구 `CanCommunication` 게이트웨이 SWC 는 폴더 보존 상태로 빌드에서 제외. AUTOSAR SWC 매핑은 `ACC-autosar/CLAUDE.md` + `ACC-autosar/RUNBOOK.md` (Pattern B 통합 진행 기록) 와 각 `create_swc_*.m` 헤더 주석 참조.
- **Arduino 2 보드 (acc_can_node + acc_motor_node)** 는 *dumb* motor controller. `acc_can_node` 는 CAN↔I2C 게이트웨이(MCP2515 + Wire Master), `acc_motor_node` 는 Motor Shield R3 ×2 스택 + 엔코더 1개(Wire Slave). I2C 분리 이유는 Arduino Uno 핀 충돌 (D10~D13 을 CAN SPI 와 모터쉴드가 동시 요구). DBC 4륜 PWM 은 `can_handler.cpp` 어댑터에서 L/R pair (avg) 로 축소 전달. 30 ms `MTR_CMD` 타임아웃 + 30 ms `ECU_HEARTBEAT` 타임아웃 → 모터 정지 safe state (SAF018 ASIL-B).
- **Raspberry Pi 5 (SENSOR)** 가 HMI (PyQt5) + 센서 퓨전 (카메라 YOLO + RPLiDAR) + CAN 브릿지(python-can) 를 모두 담당. 퓨전이 RPi 쪽이라 ECU 는 필터링 없이 가공된 `VEH_DIST(mm)` + `VEH_DET` 만 수신. 현재 `acc_can/`, `acc_fusion/` 모듈은 실동작 코드가 들어와 있고 (과거 skeleton 상태 아님) `acc_hmi/` 와 공용 `common/`, `interfaces/` 모듈로 분리되어 있다.
- **Requirements IDs** (`STK001`…`STK024`, `SYS001`…`SYS037`, `SWR001`…`SWR033`, `SAF001`…`SAF018`, `HWR001`…`HWR012`) are the traceability backbone — code comments reference them, and the AUTOSAR architecture doc maps them to SWCs/interfaces/data types. Preserve these IDs in comments when editing; don't invent new ones.

## Working in `ACC-autosar/mbd/` (MATLAB AUTOSAR MBD)

The `.slx` and `.sldd` files are binary **generated artifacts**. The `.m` scripts are the source of truth — re-generate rather than hand-editing the model.

**Preferred: one-shot WSL rebuild** (as of 2026-04-23, all 6 SWCs pass `slbuild`):

```bash
cd /mnt/c/acc/ACC-autosar
matlab.exe -batch "run('$(wslpath -w ./mbd/rebuild_all.m)')"
```

`rebuild_all.m` turns on diary, cleans orphans (root `<SWC>.slx`, shadow `ACC_Types.sldd`, stale `slprj`), regenerates the dictionary, then runs `create_swc_* → map_swc_* → (build_logic_* for DistanceControl) → slbuild` for all 6 SWCs. Log lands at `mbd/build_log_YYYYMMDD_HHMMSS.txt`.

**Per-SWC from MATLAB GUI** (required order):

```matlab
cd ACC-autosar/mbd
addpath(genpath(pwd))                     % root + all SWC subfolders

build_acc_dictionary                      % Step 1: creates ACC_Types.sldd
cd AccStateMachine
create_swc_AccStateMachine                % Step 2A (auto-calls helpers)
map_swc_AccStateMachine                   % Step 2B: Simulink ↔ AUTOSAR mapping
slbuild('AccStateMachine')                % Step 2C: verify Rte_*.h/.c
cd ..

% DistanceControl has an extra logic-injection step:
cd DistanceControl
create_swc_DistanceControl
map_swc_DistanceControl
build_logic_DistanceControl               % ControlLogic MATLAB Function (PID)
slbuild('DistanceControl')
cd ..

% Force rebuild after edits: create_swc_AccStateMachine('Force', true)
```

All 6 SWCs (`AccStateMachine`, `DistanceControl`, `HmiManager`, `Diagnostics`, `CanCommunication`, `MotorControl`) now build successfully. Inner logic status varies — Hmi/Diag are fully implemented in `create_swc_*`, `DistanceControl` has outer-loop 거리 PID (20ms) via `build_logic_DistanceControl.m` (gains are `TODO_TUNE`), `MotorControl` has inner-loop 속도 PI (10ms) via `build_logic_MotorControl.m` (gains `TODO_TUNE`, FF_Lookup=0 — 벤치 데이터 대기), and AccStateMachine/CanComm still ship skeletons that need Stateflow chart / CAN decode logic respectively. `register_acc_autosar_datatypes.m` and `apply_acc_autosar_interfaces.m` are internal helpers — do not call directly. R2022a and earlier lack the required `autosar.api.create` APIs; R2025a may break `mapInport`/`mapOutport` signatures. Target MCU has no FPU — `float32_t` is only allowed inside PID/PI IRVs.

**Interface source of truth**: `apply_acc_autosar_interfaces.m` (SR/CS/MS definitions). Every `map_swc_*` / `build_logic_*` / `create_swc_*` DataElement name and type must match. v0.5.2 (2026-04-22) renamed `FwdDistanceCm(uint8)` → `FwdDistanceMm(uint16 mm, 0~12000, DBC VEH_DIST)`, dropped `RelVelocityCmS` from `IF_SensorData` (ECU internal), and dropped `TargetSpeedCmS` from `IF_MotorCmd`. **v0.6 (2026-04-23, cascaded)**: DC Provide Port 를 `PP_MotorCmd` → `PP_CtrlOutput` 로 복귀, MC 에 `RP_SensorData` 포트 추가 (EgoSpeedCmS inner PI feedback), MC inner-loop 속도 PI 도입 (SWR034). DC 주기 10ms → 20ms, MC 주기 10ms 유지.

See `ACC-autosar/mbd/README.md` for the full SWR → dictionary mapping and troubleshooting table.

## Working in `ACC-Arduino/`

**두 개의 독립 스케치** (`acc_can_node/`, `acc_motor_node/`) 이며 `BOARD_FRONT/REAR` 같은 매크로 분기는 없다. 역할 분리:

- `acc_can_node/` — DFRobot MCP2515 CAN Shield 장착. ECU 와 CAN 으로 통신하고 I2C Master 로 `acc_motor_node` 에 명령을 내림. `config.h` 에 CAN ID / 주기 / 타임아웃 상수.
- `acc_motor_node/` — Motor Shield R3 ×2 스택(좌/우 pair) + 엔코더 1개(D2 INT0, D4 polling). Wire Slave (`0x10`). CAN 코드 없음.

공통 로직이 아니므로 `can_handler`/`motor_driver`/`encoder` 를 양쪽에 복사하지 말 것. `i2c_handler` 는 양쪽에 각각 있으나 Master/Slave 구현이 다름.

DBC 의 4륜 `SET_PWM_LF/RF/LR/RR` 는 `acc_can_node/can_handler.cpp` 어댑터에서 좌/우 2채널 평균으로 축소되어 I2C 전달된다 (의도된 운용 모드 — 확장 시 어댑터만 교체). `MTR_CMD` E2E P01 (RC/CRC) 자리는 DBC 에 예약되어 있으나 현재 송신 0 / 수신 skip (SAF010 TODO). 30 ms 타임아웃(SWR018)이 유일한 활성 safe-state 경로.

Build: 각 `.ino` 를 Arduino IDE 에서 열고 `acc_can_node` 쪽은 **DFRobot_MCP2515** 라이브러리 설치 (MCP_CAN 아님, 헷갈리지 말 것). 상세 핀맵/배선은 `ACC-Arduino/CLAUDE.md`.

## Working in `ACC-RPI/`

Python 3.14+, managed with `uv`. 모듈 구조: `acc_hmi/` (PyQt5 GUI), `acc_can/` (DBC codec + CanInterface, 자동 sim fallback), `acc_fusion/` (camera + LiDAR + YOLO), `common/` (로거 등), `interfaces/`, `config.toml`. 과거처럼 skeleton (`__init__.py` 뿐) 이 아니다.

```bash
cd ACC-RPI
uv sync                                 # 루트 pyproject.toml 기반 설치

# 전체 통합 실행 (acc_can + acc_fusion + acc_hmi — 루트 main.py 는 아직 TODO 조립 단계)
python -m main

# HMI 단독 실행 (권장 — 현재 가장 안정)
python -m acc_hmi.main
```

`acc_hmi/main.py` 는 `acc_can.CanInterface` 를 기동하고 GUI 에 주입한다. `python-can` 부재 또는 CAN 버스 open 실패 시 `acc_can` 이 자동으로 simulation 모드로 fallback — 별도 `--can` 플래그 없음 (과거 문서에 있던 `--can` 서술은 stale).

`acc_hmi/acc_state.py` 는 **시뮬레이션 용** 상태머신 (standalone GUI 테스트). 실제 차량에서는 MPC5606B 가 상태를 관리하고 RPi 는 렌더링만 — 요구사항이 바뀌면 `../ACC-autosar/mbd/AccStateMachine/` 과 함께 수정.

## Conventions and gotchas

- **Commit scope:** each subdirectory is its own repo — do not try to commit across them in one operation.
- **Requirement traceability:** `SWR###`, `SYS###`, `STK###` tags in comments are load-bearing (they map to `ACC-docs/reqs/*.sdoc` and ASPICE audit). When you delete a feature, delete the SWR reference too; when you add one, cite the requirement you're implementing.
- **Safety vs ergonomics:** this is a safety-themed project with ASIL-A/B items (unintended accel, brake failure). The requirements reflect ISO 26262 decisions — don't "simplify" away timeouts, heartbeats, override checks, or FAULT-state handling without checking `ACC-docs/reqs/STK/SYS/SAF.sdoc`.
- **Binary artifacts:** `.slx`, `.sldd`, `.docx`, `.xlsx`, `.pdf` in `ACC-docs/` and `ACC-autosar/` are regenerated from scripts or authored in GUI tools. Never hand-edit binaries; edit the `.m` script or the `.sdoc` source instead.
- **Korean text:** requirements, comments, and docs are mostly in Korean. Preserve the original language when editing nearby text.
