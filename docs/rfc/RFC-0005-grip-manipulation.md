# RFC-0005: GRIP — Grasping and Placing Household Objects with a Humanoid Arm

- **Status:** Draft v2
- **Owner:** Member 5 (Manipulation engineer)
- **Backup:** Member 2 (IRIS)
- **Package:** `src/mf_grip` (Python + C++)
- **Implements:** `mindframe.manipulation.v1.ManipulationService`
- **Consumes:** `RobotService` (arm, hand), `PerceptionService` (cloud, surface)

## 1. Mission

GRIP picks an object from a surface, carries it safely, and places it upright on another surface. The robot stands still during manipulation.

## 2. Career value

You gain hands-on **humanoid arm kinematics** (Pinocchio), motion planning (MoveIt 2 or cuRobo), dexterous-hand grasping, **force and contact sensing**, and **learning-based grasping and imitation learning** (for example ACT or Diffusion Policy with teleoperation data). These skills fit **robot manipulation engineer, robot learning engineer, and humanoid robotics engineer** roles. Your portfolio items are a pick-and-place benchmark and a learned-policy comparison.

## 3. Scope

**In scope:**
- Grasp generation.
- IK and trajectories.
- Hand control.
- Grasp verification.
- Place.
- Carry pose.
- Optional (stretch goal): a learned policy.

**Out of scope:** object detection (IRIS), and walking (STRIDE).

## 4. Design

- **Decide in week 1:** the hand type (Dex3-1, Inspire, or other), recorded as an ADR.
- **v1 heuristic grasp:**
  - Side grasp for cylinders: object axis from the cloud (PCA), approach from the robot side, pre-grasp offset 10 cm.
  - Top grasp for boxes.
- **Kinematics:**
  - Pinocchio with the G1 URDF (`mf_description`).
  - 7-DoF arm IK, with optional waist DoF.
  - Collision check against the table box and the robot body.
- **Execution:**
  - Stream joint targets at 50–100 Hz through `RobotService.StreamArmCommand`.
  - Close the hand with `SetHand`, using the force fraction.
- **Verify:** Combine `contact_detected`, the hand joint error, and a re-observation by IRIS.
- **Place:** Choose the free-area point closest to the robot. Lower until contact (current spike), open the hand, and retreat.
- **Stretch goal:** Collect 200 teleop demos with HUB's teleop and an XR or joystick setup. Train ACT or Diffusion Policy, and compare it with the heuristic on the same benchmark.

## 5. Independent development setup

| Need | Substitute |
|------|-----------|
| Robot arm + hand | **DOJO sim** (MuJoCo G1 with the selected hand model; the robot stands on a fixed base or is balanced by the controller) |
| Object poses and clouds | DOJO ground-truth objects through the IRIS mock (`backend=sim_gt`), with configurable noise to simulate real perception |
| Real arm (no walking) | SPINE `robot_safe` on a gantry, available from week 6 |

## 6. Deliverables and validation

| ID | Deliverable | Verify command | Pass criteria |
|----|-------------|----------------|---------------|
| D1 | Kinematics package: IK, reachability map at table heights 0.6–0.9 m | `just verify kin` | IK success rate is consistent with the reachability map. Self-collision test passes. |
| D2 | `Pick` in sim (`grip-bench-v1`: 10 cups/bottles × 20 poses, with perception noise σ = 1 cm) | `just verify pick` | Success ≥ 90% |
| D3 | `Place` + carry pose in sim | `just verify place` | Upright and stable ≥ 95%. Object kept while the base walks 10 m ≥ 95%. |
| D4 | Real-robot acceptance (5 real cups × 10 trials, robot_safe) | `just verify pick --backend real` | Pick ≥ 85%. Place ≥ 95%. Zero unsafe contacts. |
| D5 (stretch) | Learned policy vs. heuristic report | `just verify pick --policy learned` | Same benchmark, documented comparison |

## 7. Tracking (epics)

`GRIP-E1` hand decision + model · `E2` kinematics · `E3` pick · `E4` place + carry · `E5` real tuning · `E6` learned policy (stretch).

## 8. Milestones

| Week | Output |
|------|--------|
| 1 | Hand ADR |
| 2 | Mock `ManipulationService` |
| 5 | D1 PASS |
| 10 | D2 PASS |
| 13 | D3 PASS |
| 18–20 | D4 on robot |
| 24 | D5 (if stretch) |

## 9. Handover

Keep the grasp parameters per object class in `config/grasps.yaml`, the reachability maps in DVC, and the demo dataset in DVC with a datasheet.

## 10. Risks

- **Arm motion upsets balance:** Use speed and payload limits. Test new motion on a gantry first.
- **Small arm workspace:** This depends on STRIDE docking accuracy (±3 cm). The waist DoF gives a fallback.
