# MINDFRAME Integration and Deployment Plan

- **Status:** Draft v1
- **Owners:**
  - Tech lead: integration decisions.
  - HUB: release and deploy.
  - DOJO: test harness.
  - SPINE: robot sessions.
- **Related:** RFC-0000 §8–§12, [ENGINEERING_STANDARDS.md](standards/ENGINEERING_STANDARDS.md), [REPO_AND_DEPLOY.md](REPO_AND_DEPLOY.md)

## 1. Principle: integration is configuration, not new code

Every subsystem already implements a contract and passes the same conformance tests as its mock. So integration replaces a mock with the real service, **one at a time**, by changing the release manifest. When something breaks, the manifest diff shows exactly which subsystem changed.

```yaml
# deploy/releases/2025.04.0-rc1.yaml   (release manifest = the integrated platform)
release: 2025.04.0-rc1
contracts: 1.2.0
site: lab
profile: robot_safe
services:
  robot:        {impl: spine,  image: mindframe-spine:0.9.1,  host: pc2}
  world:        {impl: atlas,  image: mindframe-atlas:0.7.0,  host: pc2}
  navigation:   {impl: stride, image: mindframe-stride:0.8.2, host: pc2}
  manipulation: {impl: grip,   image: mindframe-grip:0.6.0,   host: pc2+workstation}
  perception:   {impl: iris,   image: mindframe-iris:0.9.0,   host: workstation}
  task:         {impl: cortex, image: mindframe-cortex:1.0.0, host: workstation}
  console:      {impl: hub,    image: mindframe-hub:1.1.0,    host: workstation}
# To swap a subsystem for its mock:   perception: {impl: mock, image: mindframe-iris:0.9.0, args: [--mock]}
```

## 2. Entry criteria (before a subsystem enters integration)

A subsystem can enter integration only when all of these are true:

1. Its image is tagged `vX.Y.Z` and is in the lab registry.
2. The DOJO conformance suite passes against the image (CI evidence).
3. Its own `just verify` result card is PASS for the milestone targets.
4. The backup owner has reproduced the result card.
5. Its health, info, and `/metrics` endpoints work.

## 3. Integration ladder

Climb one level at a time. A level is complete only when its exit test passes. The tech lead records the result in the integration log (`docs/integration/log.md`).

| Level | Configuration | Exit test | Target week |
|-------|---------------|-----------|-------------|
| **L0 All mocks** | CORTEX + HUB real; all others mock | `dojo bench task --profile mocks` → reference task 100% (mocks always succeed); fault-injection recovery ≥ 90% | 5 (G1) |
| **L1 Pairs in sim** | Pairwise real-to-real integration in DOJO, others mocked: STRIDE↔ATLAS, GRIP↔IRIS, CORTEX↔ATLAS, and STRIDE↔GRIP (dock → pick) | Each pair has a pair test (`tests/integration/pairs/`) and passes it | 13–14 |
| **L2 Full sim** | All real services, DOJO backend | `dojo bench task --n 50 --randomize` ≥ 80%. Failure attribution by subsystem published. | 16 (G3) |
| **L3 Robot, one skill at a time** | SPINE backend, `robot_safe`, gantry first. Order: SPINE → ATLAS loc → STRIDE nav/dock → IRIS → GRIP pick/place → CORTEX | Each subsystem's real-robot acceptance test (D-items in RFCs) | 17–20 (G4) |
| **L4 Robot full task** | All real services on the robot, `robot_safe` → `robot` | Reference task 10 runs ≥ 8 success; 30 varied instructions; zero safety incidents | 21–24 (G5) |

### 3.1 Fault isolation procedure

If a level fails:

1. Find the failing `task_id` in Grafana and follow the trace across the services.
2. Read `SkillResult.reason` to find which subsystem reported the failure.
3. Replay the MCAP of the failed run on that subsystem alone (`just replay <mcap>`).
4. Swap the suspect subsystem back to its mock. If the level passes, the problem is in that subsystem. If it still fails, the problem is in the contract or in the integration, so open an issue labeled `integration` and assign it to the tech lead.
5. Add the failure case to that subsystem's benchmark, so that it does not come back.

### 3.2 Integration rhythm

- **Integration day:** every Wednesday from week 13, for 2 hours, and all members attend. HUB builds a release candidate (RC) manifest from the latest tagged images. DOJO runs the benchmark. The team fixes the top problems.
- **Robot sessions:** booked by issue. SPINE runs them with the session runbook. Maximum 3 sessions per week.
- **Release train:** one RC every week, and one stable release at each gate.

## 4. Deployment architecture

```text
                    Lab network (router)                           Lab services (NAS / server)
  ┌─────────────────────────────────────────────┐     ┌───────────────────────────────────────┐
  │ GPU workstation (x86 + NVIDIA)              │     │ Docker registry  (images)             │
  │  cortex · iris · grip-grasp · hub-console   │◄───►│ DVC remote       (datasets, weights)  │
  │  vLLM (local LLM) · prometheus · grafana    │     │ MLflow           (experiments)        │
  └──────────────────┬──────────────────────────┘     │ MCAP archive + catalog                │
                     │ wired Ethernet                 └───────────────────────────────────────┘
  ┌──────────────────▼──────────────────────────┐
  │ G1 PC2 (Jetson, arm64)                      │      Operator laptops → http://workstation:8080
  │  spine · atlas · stride · grip-exec · rec   │
  │  (systemd → docker compose, starts in DAMP) │
  └──────────────────┬──────────────────────────┘
                     │ Unitree internal network
           G1 motion control computer (Unitree, not modified)
```

## 5. Release and deploy procedure

### 5.1 Create a release (HUB)

1. Every subsystem owner tags a version that meets the §2 entry criteria.
2. HUB generates `deploy/releases/<release>.yaml` from those tags with `mf release create`.
3. CI runs the **release gate**:
   - Conformance tests for every image.
   - `dojo bench task --n 50`.
   - A check that every image uses the same contracts version.
4. The tech lead approves the release PR. The tag `release-<version>` is created.

### 5.2 Deploy to the robot

Before you deploy, do the safety checklist (`docs/runbooks/robot_session.md`). Check the gantry or a clear area, test the remote e-stop, check that the battery is above 50%, and have two people present.

1. Deploy to the workstation:

   ```bash
   mf deploy <release> --target workstation
   ```

2. Deploy to PC2. The robot starts in DAMP mode.

   ```bash
   mf deploy <release> --target robot
   ```

3. Check that every service is healthy and on the correct version:

   ```bash
   mf status
   ```

   Every service must show `OK`, and the versions must match the manifest.

4. Run the robot smoke test:

   ```bash
   mf smoke
   ```

   This checks the streams and localization. It also does a stand-up and sit-down, and checks the e-stop.

5. Open the operator console and send the task.

### 5.3 Rollback

```bash
mf rollback --target robot        # previous manifest; the robot goes to DAMP before restart
mf rollback --target workstation
```

### 5.4 New site (for example, a customer home)

1. SPINE records a walk through the site.
2. ATLAS builds the map: `just map <bag>`.
3. The team labels the rooms and places in the HUB console. This makes `config/sites/<site>/`.
4. Run `just verify site <site>`.
5. Run `mf deploy <release> --site <site>`.

## 6. Production readiness checklist (Release 1.0, G5)

- [ ] All 8 subsystems PASS their final result cards, reproduced by the backup owner.
- [ ] L4 exit test passed and recorded (video + MCAP + result card).
- [ ] Safety: e-stop latency measured and documented ≤ 100 ms. Watchdog and fall tests pass. Zero open safety issues.
- [ ] Deploy and rollback rehearsed on the robot.
- [ ] Runbooks complete: robot session, deploy, new site, incident, battery and charging.
- [ ] All handover checklists complete ([ONBOARDING.md](standards/ONBOARDING.md)).
- [ ] Release notes and known limitations published.

## 7. Responsibilities during integration

| Role | Responsibility |
|------|----------------|
| Tech lead (CORTEX) | Runs integration day. Owns the integration log and the go/no-go decision for each level. |
| HUB | Release manifests, deploy, observability, data capture of every run |
| DOJO | Benchmarks, failure attribution, pair tests |
| SPINE | Robot sessions, safety officer during all real-robot tests |
| Every owner | Fixes issues in their subsystem within the next release train. Keeps the mock in sync. |
