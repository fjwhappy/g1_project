# MINDFRAME Engineering Standards

- **Status:** Draft v1. Owner: tech lead. Every member must follow these standards.
- **Purpose:**
  1. Each member can work alone.
  2. Each person's work can be reproduced and checked.
  3. A new member can continue the work of a person who leaves.

Each standard below says **what** we use, **why** we use it, and **how** we check it automatically.

---

## 1. The five core rules

1. **The contract comes first.** Subsystems talk only through versioned contracts in `contracts/`. A subsystem does not read the code of another subsystem.
2. **Develop against mocks and datasets.** You do not wait for another member. A mock server, a recorded dataset, or the simulator stands in for every dependency.
3. **One command reproduces the result.** Every package has `just verify`. It rebuilds the package, runs the tests, runs the benchmark on a pinned dataset, and writes a result card.
4. **Every decision is in writing.** RFCs plan the work. ADRs record decisions. Weekly logs record progress. A new member reads these files, not chat history.
5. **Automation enforces the rules.** CI blocks a merge that breaks a contract, a test, a benchmark threshold, or the format rules.

---

## 2. Standard tools

| Area | Standard | Why | Enforced by |
|------|----------|-----|-------------|
| **Service contracts** | **Protocol Buffers (proto3) + gRPC** in `contracts/proto/` | Works with any language (Python, C++, TypeScript). Strict schema. Backward-compatibility rules. JSON mapping works directly with LLM tool calls. Widely used in industry. | `buf lint`, `buf breaking` in CI |
| **Sensor streams** (camera, LiDAR, TF, joint states) | **ROS 2 Humble** topics with standard `sensor_msgs`/`tf2` types, listed in `contracts/streams.yaml` | High-rate data. Drivers, Nav2, and RViz already use these types. | Stream contract test (checks topic, type, rate, frame) |
| **Data recording** | **MCAP** (rosbag2 storage) | One file format for ROS 2 and protobuf data. Indexed. Opens in Foxglove. | Recorder writes MCAP only |
| **Visualization** | **Foxglove Studio** layouts in `tools/foxglove/` | Same views for everyone. Works on recorded data. | Layouts stored in git |
| **Configuration** | YAML, validated by a **JSON Schema** per package (`config/schema/*.json`) | Invalid config fails at startup, not during a robot run. | `check-jsonschema` in CI |
| **Datasets and model weights** | **DVC** with remote storage on the lab NAS | Data has a version with the code. `dvc pull` gets exactly the data used in a result. | `dvc status` in `just verify` |
| **Experiments** | **MLflow** (self-hosted) for model training and tuning | Every benchmark number links to its code, data, and parameters. | Result card has the MLflow run ID |
| **Packaging** | **Docker** multi-arch images (amd64 + arm64), one per subsystem | Same environment on laptop, CI, workstation, and robot. | CI builds images on each tag |
| **Task runner** | **just** (`justfile` in each package) with standard recipes (§4) | Same commands in every package. A new member knows how to run anything. | CI calls the same recipes |
| **Python** | Python 3.10, **uv** for dependencies, **ruff** for lint and format, **mypy** strict for public APIs, **pytest** | Fast, modern, consistent. | pre-commit + CI |
| **C++** | C++17, CMake/colcon, **clang-format**, **clang-tidy**, **GoogleTest** | Same style in all real-time code. | pre-commit + CI |
| **Frontend** | TypeScript, React, ESLint, Prettier, Vitest | — | pre-commit + CI |
| **Logging** | Structured JSON logs (fields: `ts`, `level`, `subsystem`, `task_id`, `msg`) | Logs from all machines can be searched by `task_id`. | Log library in `mf_common` |
| **Metrics** | **Prometheus** metrics endpoint per service, **Grafana** dashboards | Health and performance can be seen live. | Each service exports `/metrics` |
| **Tracing** | **OpenTelemetry** trace ID = `task_id`, sent through gRPC metadata | One task can be followed across all subsystems. | gRPC interceptor in `mf_common` |
| **Docs** | Markdown in git. **MkDocs Material** site built by CI. | Documentation is versioned and reviewed like code. | markdownlint, link check |
| **Git** | Trunk-based development, short branches, **Conventional Commits**, squash merge, **SemVer** tags | Clean history. Changelogs can be generated. | commitlint, branch protection |
| **Ownership** | `CODEOWNERS`: each package has an owner and a backup | The right people review every change. | GitHub branch protection |
| **Work tracking** | **GitHub Issues + Projects** (or GitLab equivalent) | One place shows the work of all members. | Issue templates, PR must link an issue |

If the team prefers GitLab or Jira, keep the same structure: an issue for each deliverable, milestones for each phase, and labels for each project.

---

## 3. Contract standards

### 3.1 Protobuf rules

- Location: `contracts/proto/mindframe/<domain>/v1/*.proto`, with package `mindframe.<domain>.v1`.
- Use `buf` with the `STANDARD` lint rules. CI runs `buf breaking` against `main`.
- **Never** change or reuse a field number. To remove a field, mark it `reserved`.
- Make only additive changes in `v1`. A breaking change needs a new `v2` package and an ADR. The `v1` service then stays alive for at least one milestone.
- Every message and field has a comment that states the unit and the frame, for example `// meters, in frame "map"`.
- Units are SI, angles are in radians, and time uses `google.protobuf.Timestamp`.
- Every response of a long-running skill ends with `mindframe.common.v1.SkillResult`.

### 3.2 Long-running skill pattern

Every skill (navigate, pick, place, find) uses the same RPC shape:

```proto
rpc NavigateTo(NavigateToRequest) returns (stream NavigateToEvent);
// The stream sends zero or more `progress` events, then exactly one `result` event, then it closes.
rpc Cancel(mindframe.common.v1.CancelRequest) returns (mindframe.common.v1.CancelResponse);
```

- The request contains a `RequestHeader` with `task_id`, `request_id`, and `deadline`.
- Cancellation: the client calls `Cancel(request_id)` or closes the stream. The server must stop motion within 200 ms.

### 3.3 Stream contracts (`contracts/streams.yaml`)

Each topic entry lists its name, message type, minimum rate, frame, QoS, and owner. The stream contract test subscribes for 10 s and checks each item. DOJO and SPINE must both pass the same test. This guarantees that the simulator and the real robot are interchangeable.

### 3.4 Generated code

`just gen` (in the repo root) generates Python, C++, and TypeScript stubs into `contracts/gen/`. Do not edit generated code. CI checks that the generated code is up to date.

---

## 4. Standard package layout and commands

Every subsystem package uses the same layout. Use `tools/new_package.sh <name>` to create a new package from the template.

```text
src/mf_<name>/
├── README.md            # what, how to run, how to verify (≤ 1 page)
├── justfile             # standard recipes below
├── pyproject.toml | CMakeLists.txt + package.xml
├── Dockerfile
├── config/
│   ├── default.yaml
│   └── schema.json
├── <name>/ or src/      # code
├── mocks/               # mock of THIS service for other members (owner keeps it working)
├── test/
│   ├── unit/
│   ├── contract/        # tests this service against contracts/
│   └── bench/           # benchmark definitions
├── data.dvc             # pinned evaluation dataset
├── results/             # result cards (committed), raw outputs (DVC)
└── docs/
    ├── design.md        # current design (updated from the RFC)
    └── adr/             # ADR-NNN-*.md
```

| Recipe | What it does | Must pass in CI |
|--------|--------------|-----------------|
| `just setup` | Install dependencies, `dvc pull` the pinned data | — |
| `just build` | Build the code and the Docker image | yes |
| `just test` | Unit tests + contract tests | yes |
| `just mock` | Start the mock server of this service on its standard port | yes (smoke) |
| `just run` | Start the real service (sim or robot profile) | — |
| `just bench` | Run the benchmark, write `results/<date>-<version>.json` | short version, yes |
| `just verify` | `setup` + `build` + `test` + `bench` + compare with thresholds → **PASS/FAIL** | yes |
| `just demo` | Run the recorded demo scenario (for presentations) | — |

**Definition of reproducible:** a new member on a clean machine runs `git clone`, then `just setup`, then `just verify`. The command must print PASS with the same metrics as the latest result card, within the stated tolerance.

### 4.1 Standard ports (gRPC)

| Service | Port | | Service | Port |
|---------|------|-|---------|------|
| RobotService (SPINE / DOJO) | 50051 | | ManipulationService (GRIP) | 50055 |
| WorldModelService (ATLAS) | 50052 | | TaskService (CORTEX) | 50056 |
| PerceptionService (IRIS) | 50053 | | Operator gateway (HUB) | 8080 (HTTP) |
| NavigationService (STRIDE) | 50054 | | Prometheus metrics | 9100 + n |

---

## 5. Result card (the validated output)

Every benchmark run writes a result card: a JSON file plus a short Markdown summary. The project dashboard reads these cards.

```json
{
  "subsystem": "iris",
  "version": "v0.3.0",
  "git_commit": "a1b2c3d",
  "dataset": {"name": "iris-eval-household", "dvc_rev": "4f5e..."},
  "environment": {"profile": "replay", "gpu": "RTX 4090"},
  "mlflow_run": "http://mlflow.lab/#/runs/...",
  "metrics": {
    "cup_recall@0.5-2m": {"value": 0.93, "target": 0.90, "pass": true, "n": 412},
    "position_error_cm_p95": {"value": 1.8, "target": 2.0, "pass": true, "n": 380},
    "latency_ms_p95": {"value": 640, "target": 1000, "pass": true, "n": 412}
  },
  "verdict": "PASS"
}
```

---

## 6. Documentation standards

| Document | When | Template |
|----------|------|----------|
| RFC | Before large work, or before an interface change | `docs/rfc/` |
| ADR (Architecture Decision Record) | For every important technical choice | [templates/ADR_TEMPLATE.md](../templates/ADR_TEMPLATE.md) |
| Weekly log | Every Friday | [templates/WEEKLY_LOG_TEMPLATE.md](../templates/WEEKLY_LOG_TEMPLATE.md) |
| Package README | Always current | §4 |
| Runbook | For each robot operation (start, map, deploy, recover) | `docs/runbooks/` |
| Result card | Each benchmark run | §5 |

Write in clear technical English (see the writing rules in the project instructions). Use short sentences, active voice, and one term for one concept. Define each term in `docs/GLOSSARY.md`.

---

## 7. Tracking standards

### 7.1 Structure

```text
Milestone (phase gate, e.g. "M2 Skills in sim")
 └── Epic issue per member deliverable  (e.g. "IRIS D2: 3D object localization")
      └── Task issues (≤ 3 days each)   ← linked by PRs
```

- **Labels:** `proj:cortex` … `proj:hub`, `type:feature|bug|research|docs|chore`, `prio:P0-P3`, `blocked`.
- **Board columns:** Backlog → Ready → In progress → In review → Done.
- **Rule:** every PR closes or references an issue. Every deliverable epic has its acceptance criteria copied from the RFC.

### 7.2 What makes progress measurable

| Signal | Source | Seen on |
|--------|--------|---------|
| Deliverables done / planned | Epics closed per milestone | Project board (roadmap view) |
| Benchmark trend | Result cards from nightly CI | Grafana "scorecard" dashboard |
| Code health | CI pass rate, coverage, open bugs | CI + board |
| Narrative | Weekly logs | `docs/logs/<project>/` |

### 7.3 Meetings (lightweight)

- **Async stand-up:** a daily 3-line post in the team chat (yesterday, today, blockers).
- **Weekly demo (30 min, Friday):** each member shows one result card or video for 3 minutes.
- **Contract review (on demand):** for any PR to `contracts/`. The tech lead and the affected owners must approve it.
- **Milestone review (every 4–6 weeks):** go or no-go for each gate, based on result cards.

---

## 8. Handover and onboarding standards

A member who leaves must complete the [handover checklist](ONBOARDING.md#handover-checklist-person-leaving). A new member follows the [onboarding guide](ONBOARDING.md). The **bus-factor rule**: the backup owner must be able to run `just verify` and `just demo` for the package at every milestone review.

---

## 9. Safety standards (non-negotiable)

1. Only SPINE sends motion commands to the real robot (RFC-0006).
2. Every robot session uses the hardware test checklist (`docs/runbooks/robot_session.md`) and records an MCAP file.
3. Test new motion code first in the simulator, then on the robot on a gantry with the `robot_safe` profile.
4. Report every incident or near-miss in an issue labeled `safety` within 24 hours.
