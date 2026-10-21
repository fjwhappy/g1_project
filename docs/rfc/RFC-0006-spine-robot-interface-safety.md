# RFC-0006: SPINE — Robot Hardware Interface and Safety Supervisor for the Unitree G1

- **Status:** Draft v2
- **Owner:** Member 6 (Embedded/robot systems engineer)
- **Backup:** Member 7 (DOJO)
- **Package:** `src/mf_spine` (C++17), `src/mf_description`
- **Implements:** `mindframe.robot.v1.RobotService` (real backend), all sensor streams in `streams.yaml`
- **Consumes:** `unitree_sdk2`, Unitree DDS topics, sensor drivers
- **Reference:** G1 developer guide <https://support.unitree.com/home/en/G1_developer>

## 1. Mission

SPINE is the only bridge to the real G1. It publishes robot state and sensor data, executes commands within safe limits, and owns the e-stop.

## 2. Career value

You gain deep hands-on skills in **real-time robot software**: the Unitree SDK, DDS, C++ real-time loops, sensor drivers and time sync, **safety supervisor design** (watchdogs, state machines, fault handling), and hardware bring-up. These skills fit **robotics software engineer (hardware integration), embedded and real-time engineer, and robot safety engineer** roles. Every humanoid company needs these skills. Your portfolio items are a documented, tested robot abstraction layer and hardware test reports.

## 3. Scope

**In scope:**
- PC2 setup and documentation.
- Network and DDS configuration.
- Sensor drivers (Livox, RealSense) with time sync.
- URDF and TF (`mf_description`).
- Mode state machine.
- Command arbitration and limits.
- Watchdog.
- E-stop.
- Battery, temperature, and fall monitoring.
- **Data recording sessions for other members.**

**Out of scope:** planning and perception algorithms.

## 4. Design

- **Mode state machine:** DAMP → STAND_UP → READY ⇄ WALKING / MANIPULATING. Any state can go to SAFE_STOP, then to DAMP on a fall or a critical fault.
- **Locomotion:** Use the Unitree high-level loco interface (velocity, stand, sit, damp). Use the arm SDK mode for the arm while the Unitree controller balances. **The v1 policy does not use low-level joint control of the legs.**
- **Arbitration priority:** e-stop > operator teleop > autonomous.
- **Limits:** Configure velocity, joint position, velocity, and torque limits, a hand force limit, and a self-collision check in `config/limits.yaml` (schema-validated).
- **Watchdog:** If no velocity or arm command arrives for 200 ms, the robot stops moving. A lost heartbeat from the workstation for 1 s puts the robot in SAFE_STOP.
- **Monitors:**
  - Fall: tilt and angular-rate thresholds.
  - Battery: CORTEX gets a warning at < 20%, and the robot sits at < 10%.
  - Motor temperature.

## 5. Independent development setup

SPINE is the only subsystem that needs the robot from day 1. To work safely and alone:

| Need | Substitute or safety measure |
|------|------------------------------|
| Logic testing | **Fake SDK backend** (`backend=fake`): a C++ class that replays recorded `lowstate` and accepts commands. Use it for unit tests of the state machine, limits, and watchdog without hardware. |
| Real tests | Gantry or harness, `robot_safe` limits, a second person on the remote e-stop |
| Clients | Use the contract test client from DOJO. No other subsystem is needed. |

## 6. Deliverables and validation

| ID | Deliverable | Verify command | Pass criteria |
|----|-------------|----------------|---------------|
| D1 | **Robot bring-up guide** `docs/hardware/` (PC2 OS/SDK versions, network, DDS, sensors, calibration) | Backup owner follows it on a clean PC2 image | Works without help |
| D2 | **Recorded dataset service:** session runbook + recorder; delivers `atlas-walks` and `iris-eval-household` raw recordings | `just data-check` | All `streams.yaml` topics at their rates; TF complete; time sync error ≤ 5 ms |
| D3 | `RobotService` (real) + streams | `just verify contract --backend real` (DOJO conformance suite) | 100% contract tests pass |
| D4 | Safety supervisor | `just verify safety` (fake backend, 50 fault-injection tests) + hardware test protocol | E-stop → motion stop ≤ 100 ms (measured). Watchdog stops in 100% of tests. All limits enforced. |
| D5 | Reliability | `just verify soak` | 2 h soak without a crash or memory growth; command latency p99 ≤ 5 ms |

## 7. Tracking (epics)

`SPINE-E1` bring-up · `E2` sensors + recording · `E3` RobotService · `E4` safety · `E5` reliability.

## 8. Milestones

| Week | Output |
|------|--------|
| 1 | SDK examples run, versions recorded; hand type confirmed (with GRIP) |
| 2 | First recording session (D2 v0) for IRIS and ATLAS |
| 4 | Teleop walking with watchdog + e-stop; mock `RobotService` |
| 6 | Arm/hand bridge on gantry; D3 PASS |
| 8 | D4 PASS |
| 10 | D1, D5 PASS |
| Ongoing | Monthly recording sessions on request (through issues) |

## 9. Handover

Keep the bring-up guide, the limit file with justification for each limit, the hardware test logs, and the fake backend for safe testing.

## 10. Risks

- **Firmware or SDK updates break the interface:** Pin versions and test any update on the fake backend first, then on the gantry.
- **Human error during tests:** Use the mandatory session runbook and two-person rule.
