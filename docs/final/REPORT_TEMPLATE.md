# <PROJECT NAME> — Final Report

- **Author:** <name>, Member <#>
- **Subsystem:** <e.g. IRIS — Perception>
- **RFC:** <link to RFC-000X>
- **Reviewer (backup partner):** <name>
- **Date:** <YYYY-MM-DD>
- **Code version:** <git tag / commit>

---

## Abstract

In 5–8 sentences, cover:
- What you built.
- Why the full task needs it.
- The main method.
- The main measured results.
- The most important limitation.

## 1. Introduction

- The role of the subsystem in MINDFRAME. Include one diagram that shows its inputs and outputs.
- The scope as defined in the RFC.
- Any scope changes since the RFC and the reasons for them.

## 2. Requirements

Copy the deliverables table (D1…Dn) from your RFC. These are the targets that Section 5 measures. Every result in Section 5 must link to a result card and its `just verify` command.

| ID | Requirement (from RFC) | Target |
|----|------------------------|--------|
| R1 | | |
| R2 | | |

## 3. Design

- Final architecture of the subsystem: components, data flow, and where each part runs (PC2 or the workstation).
- Interfaces as delivered: actions, topics, and services. Note any differences from the RFC.
- The key design decisions, in this format:

| Decision | Options considered | Choice | Reason |
|----------|--------------------|--------|--------|
| | | | |

## 4. Implementation

- Main packages, nodes, and libraries, with their versions.
- Models used: name, size, source, and fine-tuning if any.
- Configuration and parameters that matter, and how you tuned them.
- Important engineering problems you solved, for example latency, calibration, and timing.

## 5. Evaluation

### 5.1 Method
- The test setup: DOJO scene or real environment, number of trials, and how you measured the ground truth.
- The benchmark command used, so that another person can reproduce the result.

### 5.2 Results vs. requirements

| ID | Target | Sim result (n) | Real robot result (n) | Pass/Fail |
|----|--------|----------------|-----------------------|-----------|
| R1 | | | | |

### 5.3 Contribution to the full task
- The `bench_task` failures caused by this subsystem, given as count and percentage.
- The interfaces with other subsystems: what worked and what did not.

### 5.4 Failure analysis

| Failure mode | Frequency | Root cause | Fix / proposed fix |
|--------------|-----------|------------|--------------------|
| | | | |

Include 2–4 figures, for example plots, images of failure cases, and maps.

## 6. Sim-to-real

- Differences between the simulation and real robot results, and the reasons for them.

## 7. Safety

- Safety measures in or around this subsystem.
- All incidents and near-misses during testing, and the actions taken.

## 8. Lessons learned

- 3–5 points about the technical work and the teamwork.
- What you would do differently.

## 9. Future work

- A prioritized list of the next steps for this subsystem, each with an estimated effort.

## 10. Handover

- Where the code is, and the commands to build and run it.
- Known bugs and limits.
- Data and model locations on the NAS.

## References

- Papers, repositories, and documentation used. For example: the Unitree G1 developer guide, https://support.unitree.com/home/en/G1_developer

## Appendices

- A. Full benchmark output
- B. Parameter files
- C. Weekly log summary
