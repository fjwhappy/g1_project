# Onboarding and Handover

The goal is that a new member can run their package and reproduce its latest result in **2 days**, and can make a merged change in **1 week**.

## Onboarding checklist (new member)

### Day 1 — understand

- [ ] Read `docs/rfc/README.md` (team, names, phases).
- [ ] Read RFC-0000 (architecture and contracts) and your package RFC.
- [ ] Read `docs/standards/ENGINEERING_STANDARDS.md`.
- [ ] Read the `.proto` file of your service in `contracts/proto/`.
- [ ] Read the ADRs in `src/mf_<name>/docs/adr/` and the last 4 weekly logs of your package.
- [ ] Get access: Git, NAS (DVC remote), MLflow, Grafana, lab registry, team chat.

### Day 2 — reproduce

- [ ] Follow the instructions in `README.md` in the repo root to set up the dev container.
- [ ] `cd src/mf_<name> && just setup && just verify` → PASS. The metrics must match the latest result card.
- [ ] `just demo` and watch the recorded demo in Foxglove.
- [ ] Start the full system with mocks: `just -f deploy/justfile sim-mocks`.

### Week 1 — contribute

- [ ] Pick an issue labeled `good-first-issue` in your project.
- [ ] Make a PR that passes CI and gets a review from your backup partner.
- [ ] Write your first weekly log.
- [ ] Complete one supervised real-robot session (`docs/runbooks/robot_session.md`).

## Handover checklist (person leaving)

Complete this checklist at least **1 week** before the last day. The backup owner signs off on it.

- [ ] All work is merged, or pushed to a branch that is linked from an issue. Nothing is only on a laptop.
- [ ] All data and models are in DVC and pushed to the NAS. All experiments are in MLflow.
- [ ] `just verify` passes on `main`, and the latest result card is committed.
- [ ] `docs/design.md` describes the current design, not only the original RFC.
- [ ] Every open question has an issue with context.
- [ ] Every important decision since the last ADR is written as an ADR.
- [ ] A recorded 30-minute walkthrough video is on the NAS: code tour, how to run, known problems.
- [ ] Access to secrets and accounts is transferred. Personal tokens are removed.
- [ ] The backup owner has run `just verify` and `just demo` alone.
- [ ] `CODEOWNERS` is updated.
