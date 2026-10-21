# MINDFRAME Contracts

This folder is the **single source of truth** for how subsystems talk to each other. If it is not in here, it is not an interface.

| File | What it defines | Format |
|------|-----------------|--------|
| `proto/mindframe/common/v1/` | Shared types: header, result, poses, service info | Protobuf |
| `proto/mindframe/robot/v1/` | `RobotService` — SPINE (real) and DOJO (sim) | Protobuf + gRPC |
| `proto/mindframe/world/v1/` | `WorldModelService` — ATLAS | Protobuf + gRPC |
| `proto/mindframe/perception/v1/` | `PerceptionService` — IRIS | Protobuf + gRPC |
| `proto/mindframe/navigation/v1/` | `NavigationService` — STRIDE | Protobuf + gRPC |
| `proto/mindframe/manipulation/v1/` | `ManipulationService` — GRIP | Protobuf + gRPC |
| `proto/mindframe/task/v1/` | `TaskService` — CORTEX | Protobuf + gRPC |
| `skills.yaml` | Skill catalog that CORTEX may plan with (JSON Schema args) | YAML |
| `reasons.yaml` | Allowed `SkillResult.reason` strings | YAML |
| `streams.yaml` | High-rate ROS 2 topics, types, rates, frames | YAML |

## Why two transports

- **gRPC + protobuf** for commands, queries, and skills. These are low-rate, typed, and language-neutral. A Python, C++, or TypeScript service works the same way.
- **ROS 2 topics** for high-rate sensor data (camera, LiDAR, TF, joints). ROS drivers, Nav2, and RViz/Foxglove already use them.

## Commands

```bash
buf lint                       # style rules (STANDARD)
buf breaking --against '.git#branch=main,subdir=contracts'   # no breaking changes
buf generate                   # stubs → contracts/gen/ (python, cpp, ts)
```

## Change rules

1. Make only additive changes in `v1`. Do not change or reuse field numbers. Mark removed fields `reserved`.
2. A breaking change needs a new `v2` package, an ADR, and approval from all affected owners.
3. Every field has a comment with its unit and frame.
4. Every change must keep all mocks and contract tests passing (CI checks this).

Validated: `buf lint` passes, and all files compile with `protoc` (Python + gRPC).
