# RFC-0007: DOJO — Simulation, Benchmarking, and Test Framework

- **Status:** Draft v2
- **Owner:** Member 7 (Simulation/QA engineer)
- **Backup:** Member 6 (SPINE)
- **Package:** `src/mf_dojo` (Python, MuJoCo, optional Isaac Sim)
- **Implements:** `mindframe.robot.v1.RobotService` (sim backend), sensor streams (sim), conformance test suite, end-to-end benchmark harness
- **Consumes:** all service contracts (as test client)

## 1. Mission

DOJO gives everyone a robot they can use without the robot. It also proves, with numbers, whether each subsystem and the whole platform work.

## 2. Career value

You build **robot simulation** (MuJoCo, Isaac Sim / Isaac Lab), **sim-to-real evaluation**, scene generation with domain randomization, **contract and conformance testing**, and CI for robotics. These skills fit **robot simulation engineer, robotics QA and test engineer, and evaluation engineer for embodied AI** roles. Evaluation is a fast-growing area for robot foundation models. Your portfolio items are an open humanoid household benchmark and a sim-vs-real study.

## 3. Scope

**In scope:**
- G1 simulation (MuJoCo first).
- Household scenes with randomization.
- Sim `RobotService` and sensor streams.
- **Contract conformance suite** for all 6 services.
- **End-to-end task benchmark** harness.
- Sim-to-real comparison.
- Robot session test protocol.

**Out of scope:** algorithms of other subsystems.

## 4. Design

- **Simulator:**
  - MuJoCo (`unitree_mujoco`) for control fidelity and fast CI.
  - Isaac Sim (`unitree_sim_isaaclab`) optional, for photorealistic camera data for IRIS.
- **Locomotion in sim:** A velocity-tracking policy that responds like the Unitree controller (use an open G1 RL policy). Its velocity response is published for STRIDE.
- **Scenes:** `dojo_home` (kitchen and living room, 8 × 6 m), a procedural generator (furniture layout, cup types, lighting), and ground-truth object poses. Scenes export a site package in the ATLAS `world.yaml` format.
- **Conformance suite:** For each service, a black-box test that uses only the contract (pytest + generated stubs). The tests check streaming semantics, `reasons.yaml` codes, cancellation timing, and health and info. Every subsystem CI runs it against its mock and its real image.
- **Task benchmark:** `dojo bench task --n 50 --seed 0` sends instructions to `TaskService`, checks the final world state (cup on the target table and upright), and writes a result card with failure attribution by subsystem (`SkillResult.reason`).

## 5. Independent development setup

DOJO depends on no other subsystem. It uses its own scripted mocks for the services it tests, and it uses the contracts.

## 6. Deliverables and validation

| ID | Deliverable | Verify command | Pass criteria |
|----|-------------|----------------|---------------|
| D1 | Sim `RobotService` + streams | `just verify contract --backend sim` | 100% conformance. Streams match `streams.yaml` rates. |
| D2 | Scenes + generator (`dojo_home` + 50 random variants) | `just verify scenes` | All scenes load. Ground truth exported. Site packages are schema-valid. |
| D3 | **Conformance suite** for all 6 services | Runs in every package CI | Covers 100% of RPCs. Catches all faults in a seeded fault list (mutation test). |
| D4 | **Task benchmark harness** + dashboard feed | `just verify bench-harness` (against mocks with known success rates) | Measured rates match the configured rates within ± 3% |
| D5 | Sim fidelity report: sim vs. real for walking velocity response, arm tracking, camera noise | `just verify fidelity` | Gaps measured and documented. Sim parameters updated. |
| D6 | Robot session protocol + checklists | Used in every robot session | Zero sessions without a checklist |

## 7. Tracking (epics)

`DOJO-E1` sim robot · `E2` scenes · `E3` conformance suite · `E4` benchmark harness · `E5` fidelity · `E6` test protocol.

## 8. Milestones

| Week | Output |
|------|--------|
| 2 | G1 standing and walking in MuJoCo; conformance suite skeleton |
| 3 | D1 basic (enough for STRIDE and GRIP to start) |
| 5 | D3 for all services (runs on mocks) |
| 8 | D2 PASS; cameras + LiDAR in sim |
| 10 | D4 PASS |
| 16 | G3 sim integration runs |
| 20 | D5 |

## 9. Handover

Keep the scene format documentation, the benchmark definitions with fixed seeds, and the nightly CI configuration.

## 10. Risks

- **Sim walking does not match the real controller:** Use D5 to measure the gap, and update the sim with real logs from SPINE.
- **Isaac Sim needs much GPU power:** Isaac Sim is optional and not in the CI path.
