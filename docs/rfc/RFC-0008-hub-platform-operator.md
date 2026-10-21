# RFC-0008: HUB — Robot Software Platform and Operator Console

- **Status:** Draft v2
- **Owner:** Member 8 (Platform/DevOps + full-stack engineer)
- **Backup:** Member 1 (CORTEX)
- **Packages:** `src/mf_hub`, `apps/operator_ui`, `deploy/`, `tools/`, CI
- **Implements:** operator console, gateway (HTTP/WebSocket → gRPC), `say` skill, recorder, deploy, observability
- **Consumes:** `TaskService`, `RobotService` (e-stop, teleop, state), all services' health and info

## 1. Mission

HUB makes the team productive and the robot operable. It provides the repo, CI, packaging, deploy, data, and monitoring. It also provides the console where a person gives tasks, watches the robot, and stops it.

## 2. Career value

You gain hands-on **robotics DevOps and MLOps**: multi-arch containers (x86 and Jetson ARM), CI/CD, fleet-style deployment and rollback, observability (Prometheus, Grafana, OpenTelemetry), data pipelines (MCAP, DVC), and a **real-time web operator UI** (React, WebSocket, video streaming). These skills fit **robotics platform engineer, DevOps/MLOps engineer, and full-stack engineer for robot fleets** roles. Your portfolio items are a reusable robot software platform template and an operator console.

## 3. Scope

**In scope:**
- Monorepo and package template.
- CI (lint, build, test, conformance, bench).
- Code generation for contracts.
- Docker images.
- Deploy and rollback.
- Data recording and catalog.
- Observability.
- Operator console: task input, plan view, live video, map, e-stop, teleop, approval, and the ATLAS place-labeling UI.
- Voice input and output.
- Project dashboard (tracking).

**Out of scope:** robot algorithms.

## 4. Design

- **Repo:** Follow [ENGINEERING_STANDARDS.md](../standards/ENGINEERING_STANDARDS.md). Use `tools/new_package.sh` to create packages with the standard `justfile`.
- **CI:**
  1. pre-commit
  2. `buf lint` and `buf breaking`
  3. build
  4. unit tests
  5. conformance suite (DOJO) against the mock and the real image
  6. short benchmark
  7. On a tag: multi-arch images pushed to the lab registry
- **Deploy:** `deploy/` uses Docker Compose for each machine (`robot`, `workstation`, `sim`). A release manifest `deploy/releases/<version>.yaml` pins the image version of every subsystem. `mf deploy <version>` deploys it, and `mf rollback` returns to the previous manifest.
- **Gateway:** FastAPI with WebSocket, converting gRPC streams to browser events. Video is sent as WebRTC or MJPEG.
- **Observability:** Prometheus scrapes each service's `/metrics`. Grafana has three dashboards: health, task traces by `task_id`, and the **scorecard** (latest result card of every subsystem).
- **Data:** The recorder starts automatically during robot sessions. MCAP files are uploaded to the NAS, and a catalog database (SQLite or Postgres) is indexed by session, site, task, and result.

## 5. Independent development setup

| Need | Substitute |
|------|-----------|
| Task and robot services | Scripted mocks (`TaskService` mock streams a fake plan; `RobotService` mock streams fake state + video from a recorded MCAP) |
| Deploy target | A spare Jetson or an ARM64 VM to rehearse PC2 deploys. Use the real PC2 only after rehearsal. |

## 6. Deliverables and validation

| ID | Deliverable | Verify command | Pass criteria |
|----|-------------|----------------|---------------|
| D1 | Repo skeleton, package template, CI, code generation | `just verify platform` (creates a test package from the template and runs CI locally) | New package passes CI with zero manual edits. Clean-checkout CI ≤ 20 min. |
| D2 | Images + release manifest + deploy/rollback | `just verify deploy --target arm64-staging` | Deploy ≤ 5 min. Rollback ≤ 2 min. Restart recovers all services. |
| D3 | Operator console v1 (task, plan, video, e-stop, teleop) | `just verify ui` (Playwright end-to-end tests against mocks) + e-stop timing test | All end-to-end tests pass. UI e-stop → `RobotService.EStop` ≤ 150 ms. |
| D4 | Observability + tracking dashboards | `just verify observability` | Every service visible. A task trace spans all services. Scorecard updates nightly. |
| D5 | Data recorder + catalog | `just verify data` | 100% of robot sessions recorded and indexed. Restore any session by ID. |
| D6 | Voice (speech-to-text with Whisper, text-to-speech) | `just verify voice` (recorded command set) | Word error rate ≤ 10% on the team test set |

## 7. Tracking (epics)

`HUB-E1` repo + CI · `E2` packaging + deploy · `E3` console · `E4` observability + scorecard · `E5` data · `E6` voice.

## 8. Milestones

| Week | Output |
|------|--------|
| 1 | D1 v0: repo, template, CI with `buf lint` |
| 3 | Code generation for all languages; mock `TaskService` |
| 5 | D3 v0: e-stop + teleop + video (with mocks) |
| 8 | D2 PASS (staging) |
| 12 | D3, D4 PASS |
| 14 | D5 PASS |
| 18 | D6; first release manifest for the robot |

## 9. Handover

Keep the platform runbooks (`docs/runbooks/deploy.md`, `release.md`, `incident.md`), infrastructure as code for the registry, NAS, MLflow, and Grafana (compose files), and the secrets in a password manager (never in git).

## 10. Risks

- **The Wi-Fi link to the robot is unreliable:** Use a wired link or a dedicated router. SPINE stops the robot when the heartbeat is lost.
- **Platform scope grows too much:** Use standard tools. Do not build custom infrastructure without an ADR.
