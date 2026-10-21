# RFC-0001: CORTEX — Language-Driven Task Planning and Execution

- **Status:** Draft v2
- **Owner:** Member 1 (AI/LLM engineer, also tech lead)
- **Backup:** Member 8 (HUB)
- **Package:** `src/mf_cortex` (Python)
- **Implements:** `mindframe.task.v1.TaskService`
- **Consumes:** all skill services, `WorldModelService`, `contracts/skills.yaml`

## 1. Mission

CORTEX turns a natural-language instruction into a validated skill plan. It executes the plan, watches each step, and recovers from failures.

## 2. Career value

You build a complete **LLM agent with tool use for embodied AI**. You get practice in prompt and structured-output engineering, LLM evaluation, planning with validation, and behavior trees. You also learn to serve local models (vLLM) and to fine-tune them (LoRA). These skills apply directly to **AI agent engineer, embodied AI / robot foundation model engineer, and LLM applied scientist** roles. Your public portfolio items are an open instruction benchmark and a published planner evaluation.

## 3. Scope

**In scope:**
- Ground the instruction to world IDs.
- Plan with an LLM.
- Validate the plan.
- Execute the plan.
- Recover from failures.
- Ask the operator to clarify.
- Evaluate the planner.

**Out of scope:** perception, motion, and mapping.

## 4. Design

```text
instruction ─► Grounder ─► Planner (LLM) ─► Validator ─► Executor (behavior tree) ─► result
                 ▲             ▲               │                   │
       WorldModel.GetWorldSummary   skills.yaml   errors → replan (≤3)   failure → Recovery table
```

- **Grounding:** Put `GetWorldSummary` (rooms, places, aliases, objects) into the prompt. The LLM must only output IDs from it. An unknown name returns `clarification_needed`.
- **Planner:**
  - Convert `skills.yaml` to LLM tool or function definitions automatically.
  - The output must be a `Plan` (JSON mapping of the protobuf).
  - Use a pluggable `LLMBackend`: a cloud API, or a local model (for example Qwen2.5-14B/32B-Instruct on vLLM).
  - A non-LLM `RulePlanner` (template matching) is the baseline and the fallback.
- **Validator (deterministic, no LLM):**
  - Check the args with the JSON Schema.
  - Check that every ID exists.
  - Simulate preconditions and effects symbolically (`hand_empty`, `holding`, `docked`).
  - Allow at most 30 steps.
- **Executor:** Use py_trees. Each step is a gRPC streaming call with the skill timeout. Bind variables (`$cup`). Cancel on e-stop or `CancelTask`.
- **Recovery:** A table keyed by `reasons.yaml`:

| Reason | Recovery |
|--------|----------|
| `object_not_found` | `look_at_place` → retry `find_object` → ask operator |
| `path_blocked`, `stuck` | Retry once → try next approach pose → ask operator |
| `grasp_failed`, `object_dropped` | `find_object` again → retry `pick` (max 2) |
| `FAILED_FATAL`, `estop` | Stop, report, wait for operator |

## 5. Independent development setup

| Need | Substitute |
|------|-----------|
| Skills | Scripted mocks from every package (`just mock`), plus CORTEX's own **fault-injection mock** that returns configured failures |
| World | Fixture `config/sites/dojo_home/world.yaml` |
| Robot | None. All work uses text and mocks until integration. |

## 6. Deliverables and validation

| ID | Deliverable | Verify command | Pass criteria |
|----|-------------|----------------|---------------|
| D1 | **Instruction benchmark** `cortex-bench-v1`: 300 instructions (fetch, place, multi-step, ambiguous, impossible, and adversarial) with gold plans, in Chinese and English | `just bench-data-check` | Schema-valid. Two people label 10% of it, with agreement ≥ 0.9. |
| D2 | Planner + validator (`PlanTask` RPC) | `just verify plan` | Valid plan rate ≥ 95%. Plan-match with gold ≥ 85%. Clarification is asked in ≥ 90% of ambiguous cases. No unknown IDs (100%). p95 latency ≤ 5 s. |
| D3 | Executor + recovery (`ExecuteTask` RPC) | `just verify exec` (against fault-injection mocks, 200 scripted scenarios) | Correct recovery ≥ 90%. Cancellation within 300 ms. |
| D4 | Local-model option | `just verify plan --backend local` | Local model within 5 points of cloud on D2, offline. |
| D5 | Planner evaluation report (MLflow runs, comparison of backends and prompts) | `docs/evaluations/` | Reproducible from MLflow run IDs |

## 7. Tracking (epics)

`CORTEX-E1` benchmark dataset · `E2` planner + validator · `E3` executor · `E4` recovery · `E5` local model · `E6` integration support.

## 8. Milestones

| Week | Output |
|------|--------|
| 3 | `skills.yaml` and `TaskService` contract frozen (with the tech lead) |
| 4 | D1 v0 (100 instructions); `RulePlanner` baseline result card |
| 5 | G1: reference plan executes against all mocks in CI |
| 8 | D2 PASS |
| 12 | D3 PASS; D1 complete (300) |
| 16 | D4; sim integration |
| 24 | D5; final report |

## 9. Handover

Keep `docs/design.md`, prompt versions in git (`prompts/`, with version tags), and all evaluation runs in MLflow. A new owner runs `just verify plan` and gets the same numbers for the same model version.

## 10. Risks

- **LLM hallucination:** Use structured output, the validator, and grounding to IDs.
- **API drift or cost:** Pin model versions, cache, and keep a local backend.
