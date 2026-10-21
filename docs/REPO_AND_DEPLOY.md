# MINDFRAME Repository Structure and Deployment

- **Status:** Draft design, updated for RFC-0000 v2. The interface contracts exist in `contracts/`. The package code and deploy tooling do not exist yet. HUB (Member 8) builds them in weeks 1–8 (see RFC-0008).
- **See also:** [INTEGRATION_AND_DEPLOYMENT.md](INTEGRATION_AND_DEPLOYMENT.md) for release manifests, the integration ladder, and the full deploy procedure. That file takes precedence where the two documents differ.
- **Related:** RFC-0000 §10 (layout), RFC-0006 (SPINE), RFC-0008 (HUB).

---

## 1. Repository structure

One monorepo. It is also one ROS 2 colcon workspace. Each member owns one package under `src/`.

```text
g1_project/
├── README.md                     # Quick start: clone, build, run sim
├── .github/workflows/            # CI: lint, build, unit tests, sim benchmarks   (HUB)
├── .devcontainer/                # VS Code dev container = same image as CI      (HUB)
├── .pre-commit-config.yaml       # ruff, clang-format, markdownlint              (HUB)
├── repos.yaml                    # vcstool list of third-party repos (pinned versions)
├── CODEOWNERS                    # owner + backup per package
│
├── contracts/                    # THE interfaces: protobuf/gRPC + skills/reasons/streams YAML
│   ├── proto/mindframe/<domain>/v1/*.proto
│   ├── skills.yaml  reasons.yaml  streams.yaml
│   └── gen/                      # generated stubs (python, cpp, ts); do not edit
│
├── docs/
│   ├── rfc/                      # RFC-0000 … RFC-0008
│   ├── logs/<project>/           # weekly logs (week-NN.md)
│   ├── final/                    # reports and slides
│   ├── hardware/                 # G1 network, PC2 setup, sensor calibration    (SPINE)
│   ├── standards/                # engineering standards, onboarding
│   ├── templates/                # ADR, weekly log
│   ├── runbooks/                 # robot session, deploy, new site, incident
│   ├── integration/log.md        # integration ladder results
│   └── REPO_AND_DEPLOY.md        # this file
│
├── src/                          # ROS 2 packages (colcon builds everything here)
│   ├── mf_common/                # shared libs: gRPC server base, logging, tracing, metrics (HUB)
│   ├── mf_bringup/               # top-level launch files + profiles (HUB)
│   ├── mf_description/           # G1 URDF/meshes, TF tree, sensor mounts (SPINE)
│   ├── mf_cortex/                # Member 1  CORTEX  (Python)
│   ├── mf_iris/                  # Member 2  IRIS    (Python, GPU)
│   ├── mf_atlas/                 # Member 3  ATLAS   (C++ + Python)
│   ├── mf_stride/                # Member 4  STRIDE  (Nav2 config + C++ plugins)
│   ├── mf_grip/                  # Member 5  GRIP    (Python + C++)
│   ├── mf_spine/                 # Member 6  SPINE   (C++, unitree_sdk2)
│   ├── mf_dojo/                  # Member 7  DOJO    (sim bridge, scenes, benchmarks, mocks)
│   └── mf_hub/                   # Member 8  HUB     (operator gateway, recorder, monitor)
│
├── apps/
│   └── operator_ui/              # React web app for operators                   (HUB)
│
├── config/                       # runtime config, one folder per site
│   ├── common/                   # limits, topic names, model choices
│   ├── sites/lab/                # map, semantic places, calibration for the lab
│   └── sites/home_demo/
│
├── models/                       # model manifest only (names, versions, checksums)
│   └── manifest.yaml             # weights are on the NAS, not in git
│
├── deploy/                       # everything needed to run on machines          (HUB)
│   ├── docker/
│   │   ├── base.Dockerfile       # ROS 2 Humble + CycloneDDS + unitree_sdk2
│   │   ├── robot.Dockerfile      # aarch64 (Jetson PC2): spine, stride, atlas-loc, grip-exec
│   │   └── workstation.Dockerfile# x86_64 + CUDA: cortex, iris, grip-grasp, LLM server
│   ├── compose/
│   │   ├── robot.yaml            # services on PC2
│   │   ├── workstation.yaml      # services on GPU workstation
│   │   └── sim.yaml              # everything on one PC + simulator
│   ├── dds/cyclonedds.xml        # network interface + peers for DDS
│   ├── systemd/mindframe.service # starts robot compose on PC2 boot
│   ├── releases/<version>.yaml   # release manifest: image version per subsystem
│   ├── mf                        # deploy CLI: deploy / status / smoke / rollback / logs / record
│   └── env/                      # .env per machine (IP, ROS_DOMAIN_ID, profile)
│
├── tools/                        # scripts: record bag, calibrate, map building
└── third_party/                  # imported by vcstool from repos.yaml (not edited)
    ├── unitree_sdk2/
    ├── unitree_ros2/
    ├── unitree_mujoco/
    └── unitree_ros/              # G1 URDF source
```

### Rules

1. Each package has a `README.md`, `package.xml`, tests in `test/`, and one launch file `launch/<name>.launch.py`.
2. A package depends on other packages only through `contracts/` (generated stubs) and `mf_common`. Do not import Python or C++ code from another member's package.
3. Commit config and maps to git. Do not commit model weights, bags, or datasets. These go to the NAS, and git keeps only a reference to them in `models/manifest.yaml`.
4. Pin all third-party versions in `repos.yaml` (git tag or commit).
5. Branches: `main` must stay working. Use feature branches and pull requests. Each PR needs one review, and changes to a package need a review from the package owner (CODEOWNERS).
6. Every package follows the standard layout and `justfile` recipes in [ENGINEERING_STANDARDS.md](standards/ENGINEERING_STANDARDS.md) §4.

---

## 2. Deployment targets

The system runs on three kinds of machines. The launch profile decides which nodes start.

| Machine | Arch / OS | Runs | Profile |
|---------|-----------|------|---------|
| **G1 PC2** (Jetson Orin, `192.168.123.164`) | aarch64, Unitree image | SPINE, STRIDE, ATLAS localization, GRIP execution, sensor drivers, recorder | `robot` / `robot_safe` |
| **GPU workstation** (example `192.168.123.222`) | x86_64, Ubuntu 22.04, NVIDIA GPU | CORTEX + LLM server, IRIS, GRIP grasp generation, operator UI gateway | `robot` |
| **Developer PC** | x86_64 | Everything + DOJO simulator | `sim` |

Rule: if the workstation or network fails, PC2 must still stop the robot safely. Safety code runs only on PC2.

### Why Docker

The PC2 operating system is set by Unitree. Its Ubuntu, JetPack, and ROS versions can differ from ours. Run MINDFRAME in containers so that it does not depend on the host OS. **Do not upgrade or modify the PC2 operating system or the Unitree services.** Check the PC2 OS version and the Unitree SDK version in week 1, and record them in `docs/hardware/`.

---

## 3. Network setup (one time)

```text
            ┌─────────── G1 robot internal switch (192.168.123.0/24) ───────────┐
            │  .161 motion control (Unitree, do not touch)                       │
            │  .164 PC2 (Jetson, our robot containers)                           │
            │  LiDAR, camera                                                     │
            └──────────────────────┬─────────────────────────────────────────────┘
                                   │ Ethernet cable (or Wi-Fi bridge)
                     GPU workstation .222  ──── lab router ──── team laptops (operator UI)
```

1. Connect the workstation to the robot Ethernet port. Give it a static IP in `192.168.123.0/24` that no device uses (for example `.222`).
2. In `deploy/dds/cyclonedds.xml`, set the network interface name for each machine.
3. Use the same `ROS_DOMAIN_ID` on all machines (for example `42`). Write it in `deploy/env/*.env`.
4. Make sure that each machine can see the other: `ros2 topic list` must show the `/mf/robot/*` and `/mf/sensors/*` topics on the workstation, and `mf status` must reach every gRPC service.
5. Laptops connect to the operator UI through the lab router (HTTP only). They do not join DDS.

Confirm the IP addresses and the network interface names on your robot. Use the G1 developer guide (https://support.unitree.com/home/en/G1_developer/quick_start).

---

## 4. Build pipeline

```text
git push / PR ──► CI: lint ─► colcon build ─► unit tests ─► DOJO short benchmark
                                                                   │
git tag vX.Y.Z ──► CI: docker buildx (amd64 + arm64) ─► push to lab registry
                         mindframe-<subsystem>:vX.Y.Z   (one image per subsystem)
release manifest ──► deploy/releases/<release>.yaml pins all 8 image versions
```

- Only tagged versions go to the real robot.
- Every image contains its git commit and the `models/manifest.yaml` version.
- The lab registry is a local Docker registry on the NAS, so the robot can pull images without internet access.

---

## 5. How to deploy

### 5.1 Simulation (any developer)

```bash
git clone <repo-url> g1_project && cd g1_project
vcs import third_party < repos.yaml
docker compose -f deploy/compose/sim.yaml up        # starts sim + all nodes
# open http://localhost:8080 for the operator UI
```

Without Docker (native development):

```bash
source /opt/ros/humble/setup.bash
colcon build --symlink-install
source install/setup.bash
ros2 launch mf_bringup mindframe.launch.py profile:=sim
# or one subsystem alone against mocks:
cd src/mf_iris && just mock-deps && just run
```

### 5.2 Real robot

**Before you deploy:** Do the DOJO hardware test checklist (RFC-0007 §3.4). Put the robot on the gantry or harness. Test the remote e-stop. Check that the battery charge is above 50%.

1. Choose the release. Its manifest must have passed the release gate in CI.

   ```bash
   RELEASE=2025.04.0
   ```

2. Deploy to the GPU workstation.

   ```bash
   ./deploy/mf deploy $RELEASE --target workstation
   ```

3. Deploy to PC2. The CLI connects by SSH, pulls the images listed in the manifest from the lab registry, and installs the systemd service. It then restarts MINDFRAME in DAMP mode.

   ```bash
   ./deploy/mf deploy $RELEASE --target robot --profile robot_safe
   ```

4. Check health and run the smoke test. All services must show `OK` with the manifest versions, and the robot mode must show `DAMP`.

   ```bash
   ./deploy/mf status
   ./deploy/mf smoke
   ```

5. Open the operator UI on a laptop: `http://192.168.123.222:8080`.
6. Test the software e-stop in the UI before you send any task.
7. In the UI, put the robot in `STAND_UP`, then `READY`. Then send the task.

What `mf deploy --target robot` does internally:

```bash
ssh unitree@192.168.123.164 "
  cd /opt/mindframe && docker compose -f releases/$RELEASE.yaml pull &&
  ln -sfn releases/$RELEASE.yaml CURRENT &&
  sudo systemctl restart mindframe"      # compose up, starts in DAMP mode
```

### 5.3 Rollback

The robot keeps the previous image. Make sure that the robot is in DAMP mode or on the gantry, then run:

```bash
./deploy/mf rollback --target robot        # switches to the previous manifest and restarts
./deploy/mf rollback --target workstation
```

### 5.4 Other commands

```bash
./deploy/mf logs [service]           # follow logs
./deploy/mf record start|stop        # MCAP recording on PC2, then sync to NAS
./deploy/mf stop                     # stop MINDFRAME (robot goes to DAMP)
```

---

## 6. Launch profiles

| Profile | Where | Limits | Use |
|---------|-------|--------|-----|
| `mocks` | Developer PC / CI | — | All services mocked except the ones under test |
| `replay` | Developer PC | — | Perception or mapping on recorded MCAP |
| `sim` | Developer PC | Normal | Daily development and CI |
| `robot_safe` | Real robot | Speed is 50% of normal. Arm workspace is reduced. Plans need operator approval. | First tests, new features, demos with guests |
| `robot` | Real robot | Normal (RFC limits) | Benchmark runs after a feature passes in `robot_safe` |

The profile only selects config files from `config/`. The code is the same for all profiles.

---

## 7. Who does what

| Task | Owner |
|------|-------|
| Repo skeleton, CI, Docker images, release manifests, `mf` CLI | HUB (Member 8) |
| `contracts/` | Tech lead (Member 1). Affected owners approve changes. |
| PC2 setup, network, DDS config, `mf_description` | SPINE (Member 6) |
| `sim.yaml`, simulator images, CI benchmarks | DOJO (Member 7) |
| Site config (maps, semantic places) | ATLAS (Member 3) |
| Model manifest entries | Package owner (CORTEX, IRIS, GRIP) |
