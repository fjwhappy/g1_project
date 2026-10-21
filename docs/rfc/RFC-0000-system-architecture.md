# RFC-0000: MINDFRAME System Architecture (v2)

- **Status:** Draft v2. This version replaces v1.
- **Owner:** Tech lead
- **Reviewers:** All members
- **Created:** 2025-10-18
- **v2 changes:**
  - Contracts now use protobuf/gRPC.
  - Every subsystem can develop and validate alone, with mocks, datasets, and the simulator.
  - Every subsystem has a clear validated output.
  - The integration process is now defined.

## 1. Summary

MINDFRAME is a modular "robot brain" for the Unitree G1 EDU humanoid. It accepts a natural-language task, plans it as a sequence of skills, and executes the skills safely on the real robot or in simulation.

The platform has **8 subsystems**. Each subsystem has one owner. The subsystems connect only through versioned **contracts** in `contracts/`.

## 2. Goals

1. Execute multi-step household fetch-and-place tasks from one instruction.
2. **Independence:** each member can build, test, and validate their subsystem without waiting for any other member.
3. **Reproducibility:** each subsystem output can be reproduced with one command (`just verify`) and checked against fixed metrics.
4. **Continuity:** standard contracts, tools, and documents let a new member take over any subsystem.
5. **Safety:** one component (SPINE) controls all motion and can always stop it.
6. **One code base** for the simulator and the real robot.

## 3. Non-goals (version 1)

- Opening doors or drawers, stairs, two-handed manipulation, multiple robots, and online learning.

## 4. Reference task

> "Go to the kitchen, take the cup on the table, and put it on the table in the living room."

The expected plan, using only skills from `contracts/skills.yaml`:

```text
1. navigate_to(place_id="kitchen_table")
2. dock(place_id="kitchen_table")
3. find_object(query="cup", place_id="kitchen_table")   -> $cup
4. pick(object_id=$cup)
5. navigate_to(place_id="living_room_table")
6. dock(place_id="living_room_table")
7. place(place_id="living_room_table")
8. say(text="The cup is on the living room table.")
```

## 5. Hardware assumptions

Confirm each item in week 1, and record the results in `docs/hardware/`.

| Item | Assumption |
|------|-----------|
| Robot | Unitree G1 EDU+, 7-DoF arms, waist joint |
| Onboard computer | PC2 (Jetson Orin), `192.168.123.164` |
| Motion control computer | Unitree internal, `192.168.123.161`. Do not modify. |
| Sensors | Livox MID-360 LiDAR, RealSense D435i head camera |
| Hand | Dex3-1, or another hand. **Decide in week 1.** GRIP depends on this. |
| Off-board compute | One GPU workstation (≥ 24 GB VRAM), wired to the robot network |

## 6. Architecture

```text
┌──────────────────── Interaction ─────────────────────┐
│ HUB  operator console · deploy · data · monitoring   │
└──────────────────────────┬───────────────────────────┘
                           │ TaskService (gRPC)
┌────────────────────── Cognition ─────────────────────┐
│ CORTEX  instruction → plan → validate → execute      │
└──────┬──────────────┬───────────────┬───────────┬────┘
       │ Navigation   │ Manipulation  │Perception │ WorldModel   (gRPC)
┌──────▼─────┐ ┌──────▼─────┐ ┌───────▼────┐ ┌────▼───────┐
│  STRIDE    │ │   GRIP     │ │   IRIS     │ │  ATLAS     │
└──────┬─────┘ └──────┬─────┘ └─────▲──────┘ └────▲───────┘
       │ RobotService (gRPC)         │ sensor streams (ROS 2)
┌──────▼──────────────▼──────────────┴─────────────┴───────┐
│ RobotService + streams  ← implemented by EITHER:         │
│   SPINE (real G1 via unitree_sdk2)  or  DOJO (simulator) │
└──────────────────────────────────────────────────────────┘
```

| Subsystem | Implements | Uses (consumes) |
|-----------|-----------|-----------------|
| SPINE | `RobotService`, sensor streams (real) | Unitree SDK |
| DOJO | `RobotService`, sensor streams (sim); benchmark harness | All services (as test client) |
| ATLAS | `WorldModelService`, `/mf/world/*` streams | Sensor streams |
| IRIS | `PerceptionService` | Camera streams, `WorldModelService` |
| STRIDE | `NavigationService` | `RobotService`, `WorldModelService`, streams |
| GRIP | `ManipulationService` | `RobotService`, `PerceptionService` |
| CORTEX | `TaskService` | All skill services, `WorldModelService` |
| HUB | Operator console, deploy, data, monitoring | `TaskService`, `RobotService` (e-stop) |

### 6.1 Compute placement

| Robot PC2 (safety-critical, low latency) | GPU workstation (heavy compute) |
|-----------------------------------------|---------------------------------|
| SPINE, STRIDE, ATLAS localization, GRIP motion execution, recorder | CORTEX + LLM, IRIS models, GRIP grasp model, HUB console |

Rule: everything needed to stop or balance the robot runs on PC2. If the network fails, the robot stops and stands.

## 7. Contracts

The full definitions are in [`contracts/`](../../contracts/README.md). This RFC only summarizes them.

| Contract | Transport | Purpose |
|----------|-----------|---------|
| 6 gRPC services (`mindframe.*.v1`) | gRPC + protobuf | Commands, queries, and skills |
| `skills.yaml` | YAML + JSON Schema | Skills that CORTEX may plan with |
| `reasons.yaml` | YAML | Standard failure reasons |
| `streams.yaml` | ROS 2 topics | Camera, LiDAR, IMU, TF, joint states, map, pose |

Every skill RPC has the same shape: one request, then a stream of `progress` events, then exactly one `result` with `SkillResult{code, reason, detail}`. Every server also hosts the standard gRPC health service and `ServiceInfoService` (version, commit, backend = real, sim, replay, or mock).

## 8. Independence model: how each member works alone

Every subsystem must be developed and validated with **substitutes** for its dependencies. No member waits for another member.

| Substitute | Who provides it | Used for |
|-----------|-----------------|----------|
| **Mock services** (`just mock` in each package) | Each owner for their own service. A scripted version is ready in week 2. | Calling another subsystem before it exists |
| **Recorded datasets** (MCAP + labels, DVC-versioned) | SPINE records them. The data user labels them. | Perception, mapping, and localization work offline |
| **DOJO simulator** implementing `RobotService` + streams | DOJO | Control work (STRIDE, GRIP) and end-to-end tests without the robot |
| **Fixture world model** (`config/sites/*/world.yaml`) | ATLAS | Navigation and planning without live mapping |

### 8.1 Dependency matrix (what each member uses until integration)

| Owner | Develops against | Validates on |
|-------|------------------|--------------|
| CORTEX | Mock skills + fixture world | Instruction benchmark (text only) |
| IRIS | Recorded RGB-D dataset | Labeled 3D object dataset |
| ATLAS | Recorded LiDAR/IMU dataset | Ground-truth trajectories |
| STRIDE | DOJO sim + fixture map | Sim navigation benchmark |
| GRIP | DOJO sim + fixture object poses | Sim pick-and-place benchmark |
| SPINE | Real robot (on a gantry) | Hardware acceptance tests |
| DOJO | Its own sim + mock services | Contract conformance + sim fidelity tests |
| HUB | Mock `TaskService` + mock `RobotService` | Platform tests (build, deploy, rollback) |

### 8.2 Bootstrap (weeks 1–3): the only shared dependency

The contracts are the only thing that all members need from each other. They are frozen at **Gate G0 (week 3)**.

| Week | Shared bootstrap task | Owner |
|------|------------------------|-------|
| 1 | Contracts v1 draft (`contracts/`, already started) | Tech lead + all |
| 1 | Repo skeleton, CI, package template | HUB |
| 2 | Scripted mock for every service | Each owner (template from HUB) |
| 2 | First recorded dataset from the robot (walk through rooms, look at tables and cups) | SPINE |
| 3 | DOJO `RobotService` sim (basic) | DOJO |
| 3 | **G0: contracts frozen at v1.0.0** | Tech lead |

After G0, members only make additive contract changes, and each one still needs a reviewed PR.

## 9. Output and validation model

Each subsystem delivers a **versioned release** with four parts:

1. **Artifact:** a Docker image `mindframe-<name>:vX.Y.Z` that implements its contract.
2. **Contract conformance:** contract tests pass against the image (run by DOJO `conformance` suite).
3. **Benchmark result card:** `just verify` reproduces metrics on a pinned dataset or sim scene and prints PASS or FAIL against the targets in the subsystem RFC.
4. **Demo:** `just demo` replays a recorded scenario, and there is a short video.

A subsystem is "done" for a milestone only when all four parts exist and the backup owner has reproduced them.

## 10. Safety model

1. Only `RobotService` can command motion. On the real robot, only SPINE implements it.
2. Hardware e-stop (remote) and software e-stop (`RobotService.EStop`) are always available.
3. SPINE enforces limits and a 200 ms command watchdog.
4. `robot_safe` profile: reduced speeds, operator plan approval, gantry for new motion code.
5. Every real-robot session follows the session runbook and records MCAP data.

## 11. Integration

See [INTEGRATION_AND_DEPLOYMENT.md](../INTEGRATION_AND_DEPLOYMENT.md). Integration happens in four stages:

1. All mocks.
2. Real services in the simulator.
3. Real services on the robot, one skill at a time.
4. The full task on the robot.

## 12. Milestones and gates

| Gate | Week | Pass criteria (all from result cards) |
|------|------|---------------------------------------|
| G0 Contracts frozen | 3 | Contracts v1.0.0 released; every service has a mock; CI green |
| G1 Walking skeleton | 5 | CORTEX runs the reference plan against all mocks in CI |
| G2 Standalone skills | 12 | Each subsystem passes its own `just verify` benchmark |
| G3 Sim integration | 16 | Reference task ≥ 80% over 50 randomized sim runs with all real services |
| G4 Robot integration | 20 | Each skill passes its real-robot acceptance test |
| G5 Release 1.0 | 24 | Reference task ≥ 8 of 10 on real robot; zero safety incidents; all handover docs complete |

## 13. Standards

Engineering standards are in [ENGINEERING_STANDARDS.md](../standards/ENGINEERING_STANDARDS.md). They cover tools, the package layout, the result card, tracking, and onboarding.

## 14. Open questions

1. Hand type: Dex3-1, Inspire, or none. Decide in week 1. GRIP depends on it.
2. LLM: cloud API, a local model, or both. Proposal: both, behind one `LLMBackend` interface.
3. Locomotion: the Unitree built-in controller (proposal for v1), or a custom RL policy (v2 research).
