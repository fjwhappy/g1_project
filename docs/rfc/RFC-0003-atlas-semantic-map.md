# RFC-0003: ATLAS — LiDAR Mapping, Localization, and Semantic World Model

- **Status:** Draft v2
- **Owner:** Member 3 (SLAM/mapping engineer)
- **Backup:** Member 4 (STRIDE)
- **Package:** `src/mf_atlas` (C++ + Python)
- **Implements:** `mindframe.world.v1.WorldModelService`, streams `/mf/world/map_grid`, `/mf/world/pose`, TF `map→odom`
- **Consumes:** LiDAR, IMU, odometry streams

## 1. Mission

ATLAS builds the map, localizes the robot in it, and keeps a semantic world model of rooms, places, and objects. CORTEX uses the world model to understand language.

## 2. Career value

You gain hands-on **LiDAR-inertial SLAM** on a moving humanoid, localization, map management, and **semantic scene graphs**. These skills fit **SLAM engineer, localization engineer (robotics, AV, AR), and spatial AI engineer** roles. Your portfolio items are a public humanoid LiDAR dataset with ground truth, and a SLAM comparison study.

## 3. Scope

**In scope:**
- Offline mapping.
- Online localization and relocalization.
- 2D occupancy grid.
- Scene graph (rooms, places, aliases, approach poses).
- Object memory.
- Map labeling tool (CLI; HUB adds the UI).

**Out of scope:** object detection (IRIS).

## 4. Design

| Function | Baseline |
|----------|----------|
| Mapping | FAST-LIO2 → global map PCD (loop closure with an optional backend, for example HBA or a GTSAM pose graph) |
| Localization | FAST-LIO2 odometry + scan-to-map ICP/NDT correction → `map→odom` TF |
| Relocalization | Global descriptor (Scan Context) + ICP refinement |
| 2D grid | Height band 0.1–1.5 m projection |
| Scene graph | `world.yaml` (rooms, places, aliases, approach poses) + in-memory object store with `last_seen` decay |

## 5. Independent development setup

| Need | Substitute |
|------|-----------|
| Robot walking data | **Recorded MCAP** of the G1 walking through the site (SPINE records this in week 2). You can also use public LiDAR-IMU datasets for early tuning. |
| Ground truth | Motion-capture runs if available. Otherwise, use surveyed floor markers (tape crosses measured with a laser distance meter) and AprilTags. |
| Objects | Fixture objects in `world.yaml`. IRIS mock for upserts. |

## 6. Deliverables and validation

| ID | Deliverable | Verify command | Pass criteria |
|----|-------------|----------------|---------------|
| D1 | **Dataset** `atlas-walks` v1: ≥ 10 recorded walks (≥ 30 min total) with ground-truth checkpoints | `just data-check` | Ground truth for every checkpoint; calibration file included |
| D2 | Map builder: `just map <bag>` → PCD + 2D grid + map metadata | `just verify map` | Map consistency: checkpoint error ≤ 5 cm after loop closure |
| D3 | Online localization + relocalization | `just verify loc` (replay all test walks) | Pose error p95 ≤ 5 cm / 3°. Relocalization ≤ 10 s from any start. Zero lost-track events. |
| D4 | `WorldModelService` + scene graph + labeling CLI | `just verify world` (contract + query tests) | All contract tests pass. Query p95 ≤ 50 ms. |
| D5 | Site package `config/sites/<site>/` (map, grid, world.yaml) for the lab and the demo home | `just verify site <site>` | Schema-valid; approach poses are collision-free on the grid |

## 7. Tracking (epics)

`ATLAS-E1` data + ground truth · `E2` mapping · `E3` localization · `E4` world model service · `E5` site packages.

## 8. Milestones

| Week | Output |
|------|--------|
| 2 | Mock `WorldModelService` serving fixture `world.yaml` |
| 4 | D1 v0 (3 walks) + FAST-LIO2 baseline result card |
| 7 | D2 PASS |
| 10 | D3 PASS |
| 12 | D4 PASS |
| 16 | D5 for the lab and the demo home |

## 9. Handover

Keep a mapping runbook (`docs/runbooks/build_map.md`), versioned site packages, and parameter files with comments. A new owner rebuilds the map from the recorded bag with one command.

## 10. Risks

- **Body sway while walking hurts odometry:** Use tight LiDAR-IMU coupling, deskew, and extrinsic calibration checks.
- **Furniture moves:** Objects decay by `last_seen`. CORTEX re-observes before it picks.
