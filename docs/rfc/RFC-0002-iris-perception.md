# RFC-0002: IRIS — Open-Vocabulary 3D Object Perception

- **Status:** Draft v2
- **Owner:** Member 2 (Computer vision engineer)
- **Backup:** Member 5 (GRIP)
- **Package:** `src/mf_iris` (Python, GPU)
- **Implements:** `mindframe.perception.v1.PerceptionService`
- **Consumes:** camera streams (`streams.yaml`), `WorldModelService.UpsertObjects` (optional)

## 1. Mission

IRIS finds objects from a text query and returns their 3D boxes in the `map` frame. It also gives GRIP object point clouds and free surface areas.

## 2. Career value

You build a production **open-vocabulary 3D perception** pipeline: detection, segmentation, depth lifting, and tracking. You create a **labeled RGB-D dataset**, optimize models with TensorRT on Jetson, and evaluate them carefully. These skills fit **computer vision engineer, 3D perception engineer (robotics or autonomous driving), and ML engineer for edge devices** roles. Your portfolio items are the dataset, the benchmark, and an optimized model with latency and accuracy trade-off curves.

## 3. Scope

**In scope:**
- RGB-D processing.
- Open-vocabulary detection.
- Segmentation.
- 3D boxes and point clouds.
- Tracking.
- Support plane and free-area detection.
- Camera calibration verification.

**Out of scope:** mapping, and grasp generation (GRIP).

## 4. Design

```text
RGB + aligned depth ─► detector (open-vocab) ─► segmenter ─► depth lift ─► tracker ─► ObjectInstance[]
                                                         └─► plane RANSAC ─► free-area polygon
```

| Stage | Baseline | Upgrade |
|-------|----------|---------|
| Detection | YOLO-World (fast) | Grounding DINO 1.5 / OWLv2 |
| Segmentation | EfficientViT-SAM | SAM 2 |
| 3D | Mask × depth → filtered cloud → oriented box (PCA) | Multi-view fusion |
| Tracking | 3D IoU + label association | — |
| Edge | ONNX → TensorRT FP16 on Jetson | INT8 |

## 5. Independent development setup

| Need | Substitute |
|------|-----------|
| Camera | **Recorded MCAP dataset** (`iris-eval-household`, DVC). The first recording comes from SPINE in week 2. You can also record with a handheld RealSense D435i, the same model as the robot. |
| Ground truth | You label 3D boxes on recorded frames (CVAT + a 3D labeling script). For sim frames, DOJO ground truth is free. |
| Map frame | Use the `map→camera` TF stored in the recording |

A handheld RealSense lets you start in **week 1**, without the robot.

## 6. Deliverables and validation

| ID | Deliverable | Verify command | Pass criteria |
|----|-------------|----------------|---------------|
| D1 | **Dataset** `iris-eval-household` v1: ≥ 1,500 RGB-D frames, ≥ 20 object classes, ≥ 10 cup types, 3D box labels, test split | `just data-check` | Schema-valid labels. 5% spot-checked by the backup owner. |
| D2 | Detection + 3D localization (`FindObjects`) | `just verify` (replay on test split) | Cup recall ≥ 90% at 0.5–2 m. 3D center error p95 ≤ 2 cm at ≤ 1 m. False positives ≤ 0.1 per frame. |
| D3 | Object cloud + surface (`GetObjectCloud`, `GetSurface`) | `just verify surface` | Plane height error ≤ 1 cm. Free-area IoU ≥ 0.8. |
| D4 | Jetson deployment option | `just verify --device jetson` | `FindObjects` p95 ≤ 1 s on workstation and ≤ 2 s on Jetson, within 5 points of the D2 accuracy |
| D5 | Hard-case study: transparent and reflective cups, low light | Report + result cards | Failure rates measured and published |

## 7. Tracking (epics)

`IRIS-E1` data capture + labeling · `E2` detection · `E3` 3D lift + tracking · `E4` surface · `E5` edge optimization · `E6` hard cases.

## 8. Milestones

| Week | Output |
|------|--------|
| 2 | Mock `PerceptionService` (returns fixture objects) |
| 4 | D1 v0 (300 frames) + baseline result card |
| 8 | D2 PASS on D1 v1 |
| 11 | D3 PASS |
| 14 | D4 |
| 18 | D5; live sim + robot integration |

## 9. Handover

Keep the label guide in `docs/labeling.md`, the DVC-pinned datasets and weights, and the MLflow training and evaluation runs. Each result card names its dataset revision.

## 10. Risks

- **Transparent objects give bad depth:** Measure and report it (D5). Use opaque cups for the v1 demo.
- **Domain shift between the lab and the home:** Record data at the demo site early.
