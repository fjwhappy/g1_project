# Project MINDFRAME — Robot Brain Platform for Unitree G1 EDU

MINDFRAME turns a natural-language instruction into safe robot actions.

> Example: "Go to the kitchen, take the cup on the table, and put it on the table in the living room."

## Start here

| If you want to … | Read |
|------------------|------|
| Understand the system | [RFC-0000](RFC-0000-system-architecture.md) |
| See the interfaces | [`contracts/`](../../contracts/README.md) |
| Learn the team rules and tools | [Engineering Standards](../standards/ENGINEERING_STANDARDS.md) |
| Join or leave the team | [Onboarding and Handover](../standards/ONBOARDING.md) |
| Integrate and deploy | [Integration and Deployment](../INTEGRATION_AND_DEPLOYMENT.md) |
| Prepare the final report | [Final reports](../final/README.md) |

## Members, projects, and outputs

| # | Project | Full title | Implements | Main validated output | Career track |
|---|---------|-----------|-----------|------------------------|--------------|
| 1 | [**CORTEX**](RFC-0001-cortex-task-brain.md) | Language-Driven Task Planning and Execution | `TaskService` | LLM planner + executor, with a 300-instruction benchmark | AI agent / embodied AI engineer |
| 2 | [**IRIS**](RFC-0002-iris-perception.md) | Open-Vocabulary 3D Object Perception | `PerceptionService` | 3D object finder, with a labeled RGB-D dataset | Computer vision / 3D perception engineer |
| 3 | [**ATLAS**](RFC-0003-atlas-semantic-map.md) | LiDAR Mapping, Localization, and Semantic World Model | `WorldModelService` | Map + localization + scene graph, with a humanoid LiDAR dataset | SLAM / spatial AI engineer |
| 4 | [**STRIDE**](RFC-0004-stride-navigation.md) | Humanoid Navigation and Precision Docking | `NavigationService` | Nav + docking, with a navigation benchmark suite | Navigation / locomotion engineer |
| 5 | [**GRIP**](RFC-0005-grip-manipulation.md) | Grasping and Placing with a Humanoid Arm | `ManipulationService` | Pick/place, with a pick-and-place benchmark | Manipulation / robot learning engineer |
| 6 | [**SPINE**](RFC-0006-spine-robot-interface-safety.md) | Robot Hardware Interface and Safety Supervisor | `RobotService` (real) | Safe robot abstraction layer, bring-up guide, datasets | Robotics systems / real-time / safety engineer |
| 7 | [**DOJO**](RFC-0007-dojo-simulation-evaluation.md) | Simulation, Benchmarking, and Test Framework | `RobotService` (sim) | Simulator, conformance suite, task benchmark | Robot simulation / evaluation engineer |
| 8 | [**HUB**](RFC-0008-hub-platform-operator.md) | Robot Software Platform and Operator Console | Console, CI, deploy | Platform, release system, operator UI | Robotics platform / DevOps / full-stack engineer |

## How the plan meets each requirement

| Requirement | How |
|-------------|-----|
| **Work independently** | Every service has a mock, and the contracts freeze at week 3. Each member develops against mocks, recorded datasets, or the DOJO simulator (RFC-0000 §8). Only SPINE needs the robot early, and SPINE has a fake backend for safe work. |
| **Clear, reproducible output** | Each RFC lists deliverables D1…Dn. Each one has a `just verify` command and numeric pass criteria. It produces a result card that the backup owner must reproduce. |
| **Good career** | Each project matches a strong job market and ends with a public portfolio item: a dataset, a benchmark, or a tool (§2 of each RFC). |
| **Continuity** | Protobuf contracts, a standard package layout, standard commands, ADRs, weekly logs, a glossary, and onboarding and handover checklists (Engineering Standards). |
| **Trackable** | Each deliverable is an epic. Epics belong to milestones and gates. Result cards feed a nightly scorecard in Grafana, and every member writes a weekly log. |
| **Integration and deploy** | Integration only swaps mocks for real services in a release manifest. It climbs the L0–L4 ladder, then deploys with `mf deploy` (Integration and Deployment). |

## Backup pairs

| Primary | Backup | | Primary | Backup |
|---------|--------|-|---------|--------|
| CORTEX | HUB | | GRIP | IRIS |
| IRIS | GRIP | | SPINE | DOJO |
| ATLAS | STRIDE | | DOJO | SPINE |
| STRIDE | ATLAS | | HUB | CORTEX |

The backup owner reviews the partner's PRs and reproduces the partner's result card at each gate.

## Timeline and gates (24 weeks + 2 weeks reporting)

| Gate | Week | Pass criteria |
|------|------|---------------|
| G0 Contracts frozen | 3 | Contracts v1.0.0; mock for every service; CI green |
| G1 Walking skeleton | 5 | Reference plan runs end-to-end on mocks (L0) |
| G2 Standalone skills | 12 | Every subsystem passes its own `just verify` |
| G3 Sim integration | 16 | Reference task ≥ 80% in 50 sim runs (L2) |
| G4 Robot skills | 20 | Every skill passes real-robot acceptance (L3) |
| G5 Release 1.0 | 24 | Reference task ≥ 8/10 on robot (L4); handover complete |
| Reports | 25–26 | Final report + presentation per member |

## Status

All documents are drafts. Change them by pull request. Interface changes go to `contracts/` and follow the contract rules.
