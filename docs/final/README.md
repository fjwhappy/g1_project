# Final Reports and Presentations

At the end of the project, each of the 8 members writes one **individual report** and gives one **presentation** about their subsystem. The team also gives one shared live demo.

## 1. Deliverables per member

| # | Deliverable | Format | Length | Due |
|---|-------------|--------|--------|-----|
| 1 | Final report | Markdown or PDF, from [REPORT_TEMPLATE.md](REPORT_TEMPLATE.md) | 10–20 pages (excluding appendices) | Week 25, Friday |
| 2 | Presentation slides | PDF or PPTX, from [PRESENTATION_GUIDE.md](PRESENTATION_GUIDE.md) | 12–18 slides | Week 26, day before presentation |
| 3 | Demo video | MP4, 1–3 min, real robot preferred | — | Week 25, Friday |
| 4 | Benchmark results | Final **result cards** for every deliverable D1…Dn (Engineering Standards §5), reproduced by the backup owner | — | Week 24 (frozen) |
| 5 | Handover | Completed [handover checklist](../standards/ONBOARDING.md#handover-checklist-person-leaving) | — | Week 25, Friday |
| 6 | Portfolio item | The public artifact named in §2 of your RFC (dataset, benchmark, or tool), with a license and a README | — | Week 25, Friday |

Store all files here:

```text
docs/final/
  <project>/             # cortex, iris, atlas, stride, grip, spine, dojo, hub
    report.md (or report.pdf)
    slides.pdf
    demo.mp4             # or a link in report.md if the file is large (NAS)
    results/             # benchmark JSON, plots
  team/
    team_report.md       # system-level report (tech lead + all members)
```

## 2. Timeline

| Week | Activity |
|------|----------|
| 1–24 | Each member keeps a **weekly log** in `docs/logs/<project>/week-NN.md` using the [template](../templates/WEEKLY_LOG_TEMPLATE.md). The final report is built from these logs. |
| 12 | **Mid-term check:** 10-minute internal presentation per member. Same structure as the final, shorter. Practice run. |
| 24 | **Code and benchmark freeze.** Final benchmark run in DOJO and on the real robot. No new features after this. |
| 25 | Write the report and record the demo video. Mon–Wed: draft. Thu: peer review by the backup partner. Fri: submit. |
| 26 | Rehearsal (Mon–Tue), **Presentation Day** (Thu), retrospective (Fri). |

## 3. Peer review

Each member reviews the report of their backup partner (see the pairing table in [../rfc/README.md](../rfc/README.md)). The reviewer checks:

1. Are the results measured against the RFC deliverables (D1…Dn)? Does `just verify` reproduce every number in the report?
2. Can a new engineer build and run the subsystem from the handover notes alone?
3. Are failures and limits stated honestly?

## 4. Presentation Day agenda

Each slot: **15 min talk + 5 min Q&A**. The order follows the system layers, from the robot up to the brain.

| Time | Slot | Presenter |
|------|------|-----------|
| 09:00 | Opening: goal, architecture, team (RFC-0000) | Tech lead |
| 09:15 | SPINE — robot interface and safety | Member 6 |
| 09:35 | DOJO — simulation and evaluation | Member 7 |
| 09:55 | HUB — platform and operator app | Member 8 |
| 10:15 | Break | — |
| 10:30 | ATLAS — mapping and world model | Member 3 |
| 10:50 | IRIS — perception | Member 2 |
| 11:10 | STRIDE — navigation | Member 4 |
| 11:30 | GRIP — manipulation | Member 5 |
| 11:50 | CORTEX — task brain | Member 1 |
| 12:10 | **Live demo:** the reference cup task on the real G1, with an instruction from the audience | Whole team |
| 12:40 | Team results, lessons learned, next steps | Tech lead |
| 13:00 | End | — |

Safety: run the live demo with the DOJO hardware test checklist. Member 6 (SPINE) holds the e-stop for the whole demo.

## 5. Evaluation rubric

| Criterion | Weight | Good evidence |
|-----------|--------|---------------|
| Results vs. RFC acceptance criteria | 30% | Table of target vs. measured value, with the number of trials |
| Technical quality and design decisions | 20% | Clear reasons for choices; alternatives tried and why they were rejected |
| Integration and contribution to the full task | 20% | Subsystem works in the end-to-end demo; interface stable |
| Honest failure analysis | 15% | Failure cases with causes, frequencies, and proposed fixes |
| Communication (report clarity, presentation, Q&A) | 15% | Clear figures, on time, accurate answers |

A failed acceptance criterion does not mean a failed report. A clearly measured and explained failure is a valid result.

## 6. Team report

The tech lead coordinates `team/team_report.md`. Every member contributes one paragraph.

1. Final system architecture (what changed since RFC-0000).
2. End-to-end results: `bench_task` in simulation (50 runs) and on the real robot (10 runs).
3. Failure breakdown by subsystem across all end-to-end runs.
4. Integration problems and how the team solved them.
5. Recommended roadmap for the next phase.
