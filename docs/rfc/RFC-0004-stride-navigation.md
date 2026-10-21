# RFC-0004: STRIDE — Humanoid Navigation and Precision Docking

- **Status:** Draft v2
- **Owner:** Member 4 (Navigation engineer)
- **Backup:** Member 3 (ATLAS)
- **Package:** `src/mf_stride` (Nav2 config + C++/Python)
- **Implements:** `mindframe.navigation.v1.NavigationService`
- **Consumes:** `RobotService.StreamBaseVelocity`, `WorldModelService.GetPlace`, map, pose, and LiDAR streams

## 1. Mission

STRIDE moves the robot to a named place without collisions. It then docks at a surface accurately enough for manipulation.

## 2. Career value

You gain hands-on **Nav2**, motion control for a legged base with delayed velocity response, controller tuning (MPPI), and **systematic navigation benchmarking**. You can also try **RL locomotion** (Isaac Lab, unitree_rl_gym). These skills fit **robot navigation engineer, motion planning engineer, and legged locomotion engineer** roles. Your portfolio items are a navigation benchmark suite and a docking controller with measured accuracy.

## 3. Scope

**In scope:**
- Global and local planning.
- Velocity command generation.
- Docking.
- Recovery behaviors.
- Velocity response model of the G1 controller.
- Optional (stretch goal): an RL locomotion policy behind `RobotService`.

**Out of scope:** the map (ATLAS), and balance and walking (the Unitree controller through SPINE).

## 4. Design

- **Nav2:** Smac 2D global planner and MPPI controller. Footprint radius about 0.35 m (measure it). Limits v1: vx ≤ 0.5 m/s, vy ≤ 0.3 m/s, vyaw ≤ 0.6 rad/s.
- **Bridge:** Nav2 `cmd_vel` → `RobotService.StreamBaseVelocity` at 20 Hz.
- **Velocity response model:** Identify the delay and acceleration limits from logged data (SPINE week-4 teleop data). Use the model in Nav2 and in DOJO.
- **Dock:** Approach pose → closed-loop alignment using the LiDAR line fit to the table edge (no camera needed) → standoff 0.35 m ± 3 cm.

## 5. Independent development setup

| Need | Substitute |
|------|-----------|
| Robot | **DOJO sim** `RobotService` (MuJoCo, G1 walking controller) + LiDAR stream |
| Map and places | Fixture site `dojo_home` (grid + `world.yaml`), the ATLAS mock service |
| Velocity response | Log the response of the simulated controller. After week 4, refit to SPINE's real teleop logs. |

## 6. Deliverables and validation

| ID | Deliverable | Verify command | Pass criteria |
|----|-------------|----------------|---------------|
| D1 | Velocity response model + identification notebook | `just verify vel-model` | Model predicts logged velocity with RMSE ≤ 0.05 m/s |
| D2 | `NavigateTo` in sim | `just verify nav` (`nav-bench-v1`: 100 randomized start/goal pairs in 3 sim layouts) | Success ≥ 95%. Zero collisions. Time ≤ 1.5× optimal. |
| D3 | `Dock` in sim | `just verify dock` (50 trials) | Final standoff 0.35 ± 0.03 m, yaw ± 3°, in ≥ 95% |
| D4 | Dynamic obstacle handling | `just verify obstacles` | Stops or replans for a person crossing in 100% of 30 trials |
| D5 | Real-robot acceptance (at integration) | `just verify nav --backend real` (20 runs, robot_safe) | Success ≥ 95%. Dock error within 5 cm / 5°. |
| D6 (stretch) | RL velocity-tracking policy in sim | `just verify rl-policy` | Tracks commands better than D1 baseline |

## 7. Tracking (epics)

`STRIDE-E1` velocity model · `E2` Nav2 setup · `E3` docking · `E4` benchmark suite · `E5` real-robot tuning · `E6` RL (stretch).

## 8. Milestones

| Week | Output |
|------|--------|
| 2 | Mock `NavigationService` (simulated delay, scripted failures) |
| 5 | D1 from sim; Nav2 running in DOJO |
| 9 | D2 PASS |
| 12 | D3, D4 PASS |
| 18–20 | D5 on robot |

## 9. Handover

Keep tuned parameters in `config/` with a comment for every changed value, the benchmark suite in `test/bench/`, and the tuning log in ADRs.

## 10. Risks

- **Sim-to-real gap in walking response:** Refit D1 to real logs. Keep speed limits conservative.
- **Narrow passages:** Inflate obstacles conservatively and allow in-place rotation.
