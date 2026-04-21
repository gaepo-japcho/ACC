# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Workspace layout

`/mnt/c/acc` is **not a single repository** — it is a parent directory holding four independent git repos for one ACC (Adaptive Cruise Control) system. Each subdirectory has its own `.git`, its own toolchain, and is usually worked on in isolation. Run git/build commands inside the relevant subdirectory, not at the top level.

| Subdir | Role | Language / Toolchain | Target HW |
|---|---|---|---|
| `ACC-docs/` | Requirements, ASPICE/ISO 26262 artifacts, architecture sketches | StrictDoc (`.sdoc`), Markdown, Mermaid | n/a |
| `ACC-autosar/mbd/` | AUTOSAR Classic SWC models (ACC control logic) | MATLAB R2024b + Simulink + AUTOSAR Blockset + Embedded Coder + Stateflow | NXP MPC5606B (main ECU) |
| `ACC-Arduino/acc_motor_node/` | Motor-node firmware (front + rear) | Arduino C++ / MCP2515 CAN | Arduino Uno |
| `ACC-RPI/` | HMI GUI, camera/LiDAR fusion, CAN bridge | Python 3.14+ (`uv`), PyQt5, OpenCV, Ultralytics | Raspberry Pi 5 |

Team of 5, deadline 2026-04-30. Educational / portfolio project — not shipped, but follows **Automotive SPICE 4.0 + ISO 26262:2018** tailoring (see `ACC-docs/guide/README.md`).

## System architecture (big picture)

Three compute nodes talk over CAN:

```
[RPi 5]  ── 0x110 RASPI_SENSOR (20ms) ──►  [MPC5606B main ECU]  ── 0x300/0x301 MTR_CMD (10ms) ──►  [Arduino front/rear]
   ▲          0x520 GUI_STATUS (50ms)            │                  ◄── 0x400/0x401 FEEDBACK (10ms) ──
   └──────────  0x510 GUI_CTRL (50ms)  ──────────┘                  ◄── 0x410/0x411 HEARTBEAT (50ms) ──
```

- **Main ECU (MPC5606B, AUTOSAR Classic)** runs 5 SWCs: `CanCommunication`, `AccStateMachine`, `DistanceControl`, `Diagnostics`, `HmiManager`. State machine: `OFF → STANDBY → CRUISING/FOLLOWING → FAULT`. See `ACC-docs/autosar/ACC_AUTOSAR_Architecture_Sketch.md`.
- **Arduinos (front/rear)** are *dumb* motor controllers. Same source tree, `BOARD_FRONT` / `BOARD_REAR` macro in `config.h` selects identity (CAN IDs, front=LF/RF, rear=LR/RR). 30 ms command timeout → motor stop (ASIL-B).
- **Raspberry Pi 5** currently hosts the HMI (PyQt5) and will host camera/LiDAR fusion (Ultralytics YOLO). Sensor fusion is done on RPi so the ECU does no filtering.
- **Requirements IDs** (`SWR001`…`SWR031`, `SYS*`, `STK*`) are the traceability backbone — code comments reference them, and the AUTOSAR architecture doc maps them to SWCs/interfaces/data types. Preserve these IDs in comments when editing; don't invent new ones.

## Working in `ACC-autosar/mbd/` (MATLAB AUTOSAR MBD)

The `.slx` and `.sldd` files are binary **generated artifacts**. The `.m` scripts are the source of truth — re-generate rather than hand-editing the model. Required order inside MATLAB R2024b:

```matlab
cd ACC-autosar/mbd
addpath(pwd)

build_acc_dictionary                      % Step 1: creates ACC_Types.sldd
create_swc_AccStateMachine                % Step 2A: creates AccStateMachine.slx
                                          %   (auto-calls register_acc_autosar_datatypes
                                          %    and apply_acc_autosar_interfaces)
map_swc_AccStateMachine                   % Step 2B: Simulink ↔ AUTOSAR mapping
slbuild('AccStateMachine')                % Step 2C: verify Rte_*.h/.c generation

% Force rebuild after edits:
create_swc_AccStateMachine('Force', true)
```

Only `AccStateMachine` is implemented so far; the other four SWCs follow the same `create_swc_*` + `map_swc_*` pattern. `register_acc_autosar_datatypes.m` and `apply_acc_autosar_interfaces.m` are internal helpers — do not call directly. R2022a and earlier lack the required `autosar.api.create` APIs; R2025a may break `mapInport`/`mapOutport` signatures. Target MCU has no FPU — `float32_t` is only allowed inside PID IRVs.

See `ACC-autosar/mbd/README.md` for the full SWR → dictionary mapping and troubleshooting table.

## Working in `ACC-Arduino/acc_motor_node/`

Two sibling sketches (`acc_motor_front/`, `acc_motor_rear/`) share almost-identical source; the *only* intended difference is `#define BOARD_FRONT` vs `#define BOARD_REAR` in `config.h`, which switches CAN IDs and log strings. When fixing bugs in shared logic (`can_handler.cpp`, `encoder.cpp`, `motor_driver.cpp`) apply the change to **both** sketches.

Pin map is tight — Arduino Uno has only PWM pins 3/5/6/9/10/11, and MCP2515 consumes 2, 10, 11; encoder INT1 consumes 3. Available PWM pins for motor EN: 5, 6, 9. See the ASCII pin table in `config.h` before touching pins.

Build: open the `.ino` in Arduino IDE with MCP_CAN library installed.

## Working in `ACC-RPI/`

Python 3.14+, managed with `uv`. Top-level `main.py` is a stub; the real entry point is the HMI:

```bash
cd ACC-RPI
uv sync                                 # install deps from uv.lock

cd acc_hmi
python main.py                          # simulation mode (no CAN hardware)
python main.py --can                    # real CAN mode (requires python-can + MCP2515/USB-CAN)
```

`acc_hmi/acc_state.py` is a **simulated** state machine for standalone GUI testing — in production, state transitions happen on the MPC5606B and the RPi only renders. Keep the two in sync when requirements change. `acc_can/` and `acc_fusion/` are skeletons (only `__init__.py`).

## Conventions and gotchas

- **Commit scope:** each subdirectory is its own repo — do not try to commit across them in one operation.
- **Requirement traceability:** `SWR###`, `SYS###`, `STK###` tags in comments are load-bearing (they map to `ACC-docs/reqs/*.sdoc` and ASPICE audit). When you delete a feature, delete the SWR reference too; when you add one, cite the requirement you're implementing.
- **Safety vs ergonomics:** this is a safety-themed project with ASIL-A/B items (unintended accel, brake failure). The architecture doc and requirements reflect ISO 26262 decisions — don't "simplify" away timeouts, heartbeats, override checks, or FAULT-state handling without checking `ACC-docs/autosar/ACC_AUTOSAR_Architecture_Sketch.md`.
- **Binary artifacts:** `.slx`, `.sldd`, `.docx`, `.xlsx`, `.pdf` in `ACC-docs/` and `ACC-autosar/` are regenerated from scripts or authored in GUI tools. Never hand-edit binaries; edit the `.m` script or the `.sdoc` source instead.
- **Korean text:** requirements, comments, and docs are mostly in Korean. Preserve the original language when editing nearby text.
