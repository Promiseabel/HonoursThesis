# Pose Estimation in Motion: A Comparative Evaluation of Model Accuracy for Sprinting

CS4997 Honours Thesis — Faculty of Computer Science, University of New Brunswick
Author: Promise Abel-Adegbite · Supervisor: Scott Bateman

Full thesis: [`docs/Thesis.docx`](docs/Thesis.docx)

## Research question

**How accurate are MediaPipe, RTMPose, and YOLOv11 at estimating sprint biomechanics?**

Markerless human pose estimation (HPE) promises cheap, accessible sprint-form analysis from ordinary phone video — but sprinting is close to a worst case for these models: high-speed motion blur, self-occlusion, and a subject whose size in frame changes constantly. This project benchmarks three popular, architecturally distinct HPE models against manually annotated ground truth on real sprint footage, across sprint phases, subject sizes, and visual challenges.

**Headline result:** RTMPose wins decisively in every condition tested (67.3% mean correctness vs ~30% for the others), and it was the *only* model that produced any valid predictions when the sprinter was small in the frame — MediaPipe and YOLOv11 both failed completely at distance.

## Method

**Data.** Sprint footage of a single participant, recorded on an iPhone 16 Pro Max on a tripod. Three clips: acceleration phase, maximum-velocity close to camera, and maximum-velocity far from camera. 15 unique frames were hand-selected to cover four conditions — acceleration vs. max velocity (matched for subject size), and big vs. small bounding box (matched for biomechanical position).

**Ground truth.** All frames were manually annotated in Label Studio by one annotator: 14 keypoints per frame (left/right shoulder, elbow, wrist, hip, knee, ankle, foot tip), plus the sprinter's bounding box and head bounding box. Each keypoint carries visual-challenge tags (`blurry`, `occluded`, or both), which drive the condition analysis. This yields **210 keypoint comparisons and 150 derived joint-angle comparisons** (10 angles per frame: elbow, knee, shoulder, hip, ankle × both sides).

**Models.** Run frame-by-frame on static RGB frames with default settings, no preprocessing or temporal smoothing:

| Model | Architecture | Keypoint set | Hardware |
|---|---|---|---|
| RTMPose-x (MMPose, `body8-halpe26-384x288`) | Top-down, SimCC head; person boxes from MMDetection Faster R-CNN | Halpe-26 (includes toes) | Google Colab GPU |
| MediaPipe Pose (BlazePose) | Top-down detector–tracker | 33 landmarks (2D used) | 2019 MacBook Pro |
| YOLOv11 (`yolo11n-pose`) | One-stage detection + pose | COCO-17 (**no foot keypoints**) | 2019 MacBook Pro |

**Metrics.**
- *Keypoints:* Euclidean pixel error; MPJPE; PCKh@0.5 (correct if error ≤ 0.5 × a head-size reference, i.e. 0.6 × the annotated head-bbox diagonal).
- *Angles:* absolute error in degrees via 2D vector geometry; MPAE; PCA@12 (correct within 12°, a physiotherapist-level visual tolerance).
- *Custom scoring:* every prediction gets `score = w / (1 + normalized error)`, with 0 for missed detections. The unweighted score uses w = 1; the weighted score rewards accuracy under visual challenges, with weights derived from each challenge's measured impact on error (Blurry + Occluded = 2.0, Blurry = 1.75, Occluded = 1.5, Normal = 1.0).
- *Mean correctness:* each prediction is classified Correct / False / Don't Know (missing); correctness is averaged across condition combinations.

## Results

### 1. Standard pose-estimation metrics

| Model | PCKh@0.5 (%) | PCA@12 (%) | MPJPE (px) ↓ | MPAE (°) ↓ |
|---|---:|---:|---:|---:|
| MediaPipe | 30.95 | 25.33 | 80.46 | 27.00 |
| **RTMPose** | **78.57** | **52.00** | **58.41** | **20.24** |
| YOLOv11 | 35.24 | 18.67 | 146.45 | 37.20 |

### 2. Overall performance (custom scoring)

| Model | Mean weighted score | Mean unweighted score | Mean correctness (%) |
|---|---:|---:|---:|
| MediaPipe | 0.519 | 0.361 | 27.05 |
| **RTMPose** | **0.920** | **0.653** | **67.26** |
| YOLOv11 | 0.494 | 0.338 | 30.31 |

RTMPose was also the only model with **zero missed predictions** (0% "Don't Know"); MediaPipe and YOLOv11 each failed to produce a prediction for roughly a third of all targets.

### 3. Robustness across sprint conditions (mean correctness, %)

| Condition | MediaPipe | RTMPose | YOLOv11 |
|---|---:|---:|---:|
| Big bounding box (close) | 31.64 | **69.23** | 46.77 |
| Small bounding box (far) | 0.00 | **68.54** | 0.00 |
| Acceleration phase | 58.82 | **82.19** | 34.33 |
| Max-velocity phase | 18.32 | **51.11** | 36.84 |
| Normal (no visual challenge) | 26.34 | **91.00** | 32.74 |
| Occluded | 48.17 | **72.22** | 41.86 |
| Blurry | 17.54 | **56.09** | 21.03 |
| Blurry + occluded | 12.50 | **43.89** | 24.03 |

Key observations:

- **Distance kills two of the three models.** MediaPipe and YOLOv11 produced *no valid predictions* when the sprinter occupied < 5% of the frame. RTMPose was essentially unaffected by subject size.
- **Speed hurts everyone.** All models degraded from acceleration to max velocity (where blur dominates); MediaPipe fell hardest (58.8% → 18.3%).
- **Correct keypoints ≠ correct angles.** Every model was substantially worse at joint angles than at keypoints (e.g. RTMPose: 80.9% keypoint vs 55.3% angle correctness) — small localization errors compound in multi-joint angle calculations, worst in the lower limbs.
- **YOLOv11 structurally can't do sprint feet.** COCO-17 has no foot keypoints, so YOLOv11 scores zero on foot points and ankle angles — joints that matter most in sprint mechanics.

Per-joint breakdowns, plots, and the full C/F/DK tables are in [`results/`](results/); Tables 2–9 in the thesis walk through all of them.

## Repository structure

```
├── docs/Thesis.docx            # full thesis (method, all 9 result tables, discussion)
├── notebooks/                  # the analysis pipeline, in execution order
│   ├── 01_groundtruth_annotation.ipynb   # parse Label Studio export → keypoints/angles JSON
│   ├── 02_mediapipe_inference.ipynb      # MediaPipe Pose on selected frames
│   ├── 03_rtmpose_inference.ipynb        # RTMPose-x via MMPose + MMDetection (Colab; outputs stripped)
│   ├── 04_yolo_inference.ipynb           # YOLOv11-pose on selected frames
│   ├── 05_bounding_boxes.ipynb           # sprinter/head bbox sizes, PCKh normalizer, close/far split
│   ├── 06_organize_outputs.ipynb         # merge per-video outputs into per-phase comparison sets
│   ├── 07_compare_keypoints.ipynb        # keypoint errors, PCKh, C/F/DK, challenge impact
│   ├── 08_compare_angles.ipynb           # angle errors, PCA@12, C/F/DK
│   ├── 09_score_metric.ipynb             # weighted/unweighted scores, grouped results + plots
│   ├── 10_score_metric_correctness.ipynb # correctness (C/F/DK) aggregation → CFDK summary
│   └── 11_eda.ipynb                      # early exploratory pilot (MediaPipe vs YOLOv8)
├── data/
│   ├── predictions/            # per-model, per-clip keypoints.json + angles.json (incl. ground truth
│   │                           #   and Label Studio exports); BBoxV2/ holds bbox sizes + normalizers
│   └── comparison/             # merged per-phase comparison sets + error/CFDK intermediates
├── results/                    # final CSV tables and plots (scores, correctness, challenge impact)
└── requirements.txt
```

### Data formats

- `keypoints.json`: per frame, named joints → `[x, y]` in pixels; `-1` = no prediction. Ground-truth entries may carry tags (`blurry`, `occluded`, `partial_occlusion`).
- `angles.json`: per frame, named angles (elbow/knee/shoulder/hip/ankle × left/right) in degrees, folded to [0°, 180°].
- `results/data`, `results/S_data`, `results/P_data`: score and correctness tables grouped by phase, status, type, and joint — these back the tables above.

### Reproducing

The comparison and scoring stages (notebooks 05–10) run locally from the JSON in `data/` with just `numpy`/`pandas`/`matplotlib`. The inference stages (02–04) need the raw frames, which aren't distributed (see limitations); notebook 03 additionally needs the OpenMMLab stack and was run on Colab with a GPU. Paths inside notebooks are relative to the original working layout and may need adjusting — this is research code, kept as-run for transparency.

## Limitations (honest notes)

- **One participant, 15 frames.** Frames were deliberately selected for edge cases (blur, occlusion, size extremes), not randomly sampled — good for stress-testing, but a source of selection bias and limited statistical power. No inter-subject variation was tested.
- **Single annotator, no inter-rater check.** Ground truth was one person's manual Label Studio annotation; occluded joints were placed by best judgment. No formal annotation-uncertainty tolerance was defined.
- **No significance testing.** Reported differences are descriptive; no paired tests were run.
- **The weighted score is an ad-hoc heuristic.** Weights come from the measured error impact of each visual challenge, but the scheme has no peer-reviewed precedent. Reassuringly, its rankings agree with the standard metrics (PCKh, MPJPE).
- **Frame-by-frame only.** No temporal smoothing or sequence modeling — deliberately, for fair comparison, but it discards information that matters at sprint speeds.
- **2D only.** All comparisons are in the image plane; out-of-plane motion isn't captured.
- **Architectural asymmetry.** Two top-down models vs. one one-stage model; bottom-up approaches weren't evaluated. YOLOv11's missing foot keypoints are a dataset (COCO) constraint, not an inference failure — but they're counted against it because feet matter for sprinting.
- **Raw footage not included.** The videos show a single identifiable participant and are withheld; the annotation JSONs, model predictions, and all derived results are included so the analysis (notebooks 05–10) is fully reproducible.
- **Research code.** The notebooks are as-run thesis code: hardcoded paths, duplicated cells, and exploratory dead ends included. Notebook 03's outputs were stripped (it was 48 MB with rendered frames embedded).

## Citation

> Abel-Adegbite, P. (2025). *Pose Estimation in Motion: A Comparative Evaluation of Model Accuracy for Sprinting.* CS4997 Honours Thesis, Faculty of Computer Science, University of New Brunswick. Supervised by S. Bateman.
