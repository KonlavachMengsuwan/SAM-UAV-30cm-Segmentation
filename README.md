# UAV 30 cm Segmentation with SAM (`.dlpk`) in ArcGIS Pro
<img width="1566" height="1022" alt="image" src="https://github.com/user-attachments/assets/e209f225-1570-4cc7-9116-e1656fc936d8" />

**Goal:** Apply a **pretrained SAM model** (`SAM.dlpk`) to 30 cm UAV RGB imagery to segment small objects (notably **cars** and **trees**) and study how inference parameters influence **recall**, **precision**, and **runtime**.

---

## TL;DR (Recommended Starting Setup)

- **Padding:** `256`
- **Batch Size:** `4`
- **box_nms_thresh:** `0.7`  *(raise to 0.85–0.9 to catch more adjacent cars)*
- **Points Per Batch:** `56`  *(increase to 128–256 to catch more small objects)*
- **Stability Score Threshold:** `0.9`
- **Minimum Mask Region Area:** `0` *(set to 10–30 if you want to suppress tiny speckle)*

**Observation:** Overall segmentation is good; **some cars and trees are missed**. Increasing **Points Per Batch** and moderately raising **box_nms_thresh** is the least risky way to improve recall while keeping **Stability = 0.9** for quality.

---

## 1) Introduction

This repository documents a practical evaluation of **Segment Anything (SAM)** packaged as an ArcGIS **Deep Learning Package (`.dlpk`)** on **30 cm UAV RGB** imagery. The objective is to understand how key inference parameters affect the segmentation of **small, compact objects** (e.g., **cars**) and **textured organic objects** (e.g., **trees**), and to provide **actionable guidance** for practitioners who want better recall without sacrificing output quality.

**What is a pretrained model?**  
A pretrained model is an AI model that has **already learned general patterns** from large datasets (e.g., cars, roads, trees in aerial images). You **don’t retrain** it; you **apply** it to your own data. This saves time, data, and usually gives good accuracy out of the box.

---

## 2) Methods

### Data
- **Imagery:** UAV RGB, **30 cm** spatial resolution *(resampled)*.
- **AOI:** Urban/suburban scenes with parked / moving **cars** and **tree canopies**.
- **Preprocessing:** None required beyond standard ArcGIS raster handling. (Optional: build pyramids & statistics for smoother visualization.)

### Software & Model
- **ArcGIS Pro** with Deep Learning support.
- **Model:** `SAM.dlpk` (pretrained).
- **Tool:** *Geoprocessing → Detect Objects Using Deep Learning* (or raster function equivalent).

### Baseline Inference Parameters
- **Padding:** `256`
- **Batch Size:** `4`
- **box_nms_thresh:** `0.7`
- **Points Per Batch:** `56`
- **Stability Score Threshold:** `0.9`
- **Minimum Mask Region Area:** `0`
- **Non Maximum Suppression (NMS):** ON
- **Use pixel space:** OFF (defaults are fine)

> Rationale: These settings produce clean masks with strong boundaries (`Stability = 0.9`) while keeping runtime and memory manageable.

### Parameter Sweeps (One-Factor-at-a-Time)
To compare impacts clearly, we vary **one parameter at a time** while holding others at the baseline above:

- **Points Per Batch:** `56`, `128`, `256`, `512`
- **Stability Score Threshold:** `0.5`, `0.8`, `0.9`
- **box_nms_thresh:** `0.3`, `0.5`, `0.7`, `0.9`

> NOTE: SAM is proposal-driven. **Points Per Batch** controls proposal density; **Stability** controls post-proposal filtering; **box_nms_thresh** controls overlap suppression.

---

## 3) Results

### 3.1 Qualitative Impact by Parameter

#### A) Points Per Batch (proposal density)
| Points/Batch | Cars (Recall) | Trees (Recall) | Precision | Duplicates/Overlaps | Runtime/Memory |
|---:|:---:|:---:|:---:|:---:|:---:|
| **56**  | ◔ (good) | ◔ (good) | **◎ (high)** | ◎ (low) | ◎ (low) |
| **128** | ◕ (better) | ◕ (better) | ◎ (high) | ◔ (slightly ↑) | ◔ (moderate) |
| **256** | **◎ (high)** | **◎ (high)** | ◕ (slightly ↓) | ◕ (↑ duplicates) | ◕ (high) |
| **512** | ◎/◕ (diminishing returns) | ◎/◕ | ◔ (drops) | **◔ (many duplicates)** | **◔ (very high)** |

**Takeaway:** Moving from **56 → 128** yields a **meaningful recall boost** for small objects (cars) and finer tree edges with minor cost. **256** can help more, but watch for overlaps/duplicates and longer runtime. **512** often has **diminishing returns**.

---

#### B) Stability Score Threshold (mask quality filter)
| Stability | Cars | Trees | Precision / Clean edges | Noisy blobs | Recommendation |
|---:|:---:|:---:|:---:|:---:|:---|
| **0.5** | ◕ (higher recall) | ◕ | ◔ (noticeable drop) | **◔ (many)** | Too permissive on UAV textures |
| **0.8** | ◕ | ◕ | ◕ (balanced) | ◔ | Usable, but still noisier than 0.9 |
| **0.9** | **◔/◕ (may miss some)** | **◔/◕** | **◎ (best shape quality)** | **◎ (lowest)** | **Preferred** for clean outputs |

**Takeaway:** For UAV RGB (shadows, texture), **0.9** best preserves quality. If **specific misses** are critical, loosen to **0.8** sparingly, or increase proposals instead (Points/Batch).

---

#### C) box_nms_thresh (overlap suppression)
| NMS IoU (box_nms_thresh) | Effect on Adjacent Cars | Duplicates | Precision | Recommendation |
|---:|:---:|:---:|:---:|:---|
| **0.3** | May **suppress** close neighbors | ◎ (few) | ◎ | Risk of **dropping** one of two nearby cars |
| **0.5** | Balanced suppression | ◎ | ◎ | Safe mid-ground |
| **0.7** | Keeps neighbors better | ◔ (some) | ◕ | **Good default** (current) |
| **0.9** | **Max recall** for close cars | ◕/**◔** (more/duplicates) | ◔ | Use when parking lots matter; be ready to dedupe |

**Takeaway:** If cars are close, raising to **0.85–0.9** often recovers misses. Post-processing can remove duplicates afterwards.

---

### 3.2 Recommended Profiles

- **Balanced Quality (current style):**  
  `Points/Batch = 128`, `Stability = 0.9`, `box_nms_thresh = 0.7–0.8`  
  *Cleaner masks, modest recall boost over 56.*

- **Car-Recall Focus (parking lots / dense streets):**  
  `Points/Batch = 256`, `Stability = 0.9`, `box_nms_thresh = 0.9`  
  *Finds more adjacent cars. Expect more overlaps → dedupe post-processing.*

- **Tree-Edge Emphasis:**  
  `Points/Batch = 256`, `Stability = 0.9`, `box_nms_thresh = 0.7`  
  *Refines canopy boundaries; watch runtime.*

> **Baseline used in this study:**  
> `Padding = 256`, `Batch Size = 4`, `box_nms_thresh = 0.7`, `Points/Batch = 56`, `Stability = 0.9`, `Min Region Area = 0`, NMS = ON.

---

## 4) Discussion

### Why cars and trees can be missed
- **Proposal sparsity:** With too few sampled points, small/occluded cars never get proposed.
- **Over-suppression:** Low `box_nms_thresh` may drop one car when two are adjacent.
- **Strict stability:** `0.9` can prune borderline masks (dark cars, heavy shadows, partial occlusion). We keep it for quality and instead raise proposals/NMS threshold.
- **Resolution & contrast:** 30 cm is strong, but fine edges, shadows, and similar textures (dark car vs asphalt) still challenge segmentation.

### Practical ways to recover more objects (without messy outputs)
1. **Increase proposals first:** `Points/Batch 56 → 128 → 256`  
2. **Then relax overlap suppression:** `box_nms_thresh 0.7 → 0.85–0.9`  
3. **Keep high stability (`0.9`)** for shape quality and fewer artifacts.  
4. **Post-process** to clean duplicates/noise (see below).

---

## 5) Post-Processing Recipes (ArcGIS Pro)

- **Polygonize masks** → *Add Geometry Attributes* (Area, Perimeter, Compactness).  
- **Car filter (typical ranges @ 30 cm):**
  - Area: **60–200 px²** (~6 × 10 to 10 × 20 px footprints ≈ 3–12 m²)
  - Compactness/Rectangularity: **≥ 0.6**
- **Dedupe overlaps:**  
  - *Eliminate* or *Dissolve* by “max area” or “max stability score” attribute if available.
- **Tree cleanup:**  
  - Optional *Smooth Polygon*, remove micro-holes (*Eliminate Holes*), and filter tiny fragments by area threshold (e.g., **≥ 100 px²**) for coherent crowns.

---

## 6) How to Reproduce

1. Open **Detect Objects Using Deep Learning** in ArcGIS Pro.  
2. Set:
   - Input Raster: your 30 cm UAV RGB.
   - Model: `SAM.dlpk`.
   - Parameters: start from the **Baseline** above.
3. Change **only one parameter** per run when evaluating (e.g., Points/Batch 56 → 128 → 256 → 512).  
4. Export masks, polygonize, and measure object counts to compare runs.

> Tip: Save each run’s outputs into separate feature classes (e.g., `cars_pts56_nms07_stab09.gdb`).

---

## 7) Suggestions for Further Improvement

- **More hyperparameter tuning**
  - **Points/Batch:** 128–256 sweet spot; test 192 if UI allows.
  - **box_nms_thresh:** Try **0.85–0.9** for dense cars; pair with deduping.
  - **Minimum Region Area:** 10–30 px² to suppress speckle without deleting cars.
  - **Tile size / stride:** If exposed, larger tiles (e.g., 1024) can help global context.

- **Higher spatial resolution**
  - If available, run at **finer GSD** (≤20 cm). Small objects stand out better.

- **Radiometric/contrast enhancement**
  - Gentle **contrast stretch** or **shadow mitigation** can help the model separate dark cars from asphalt.

- **Class-specific post-processing**
  - Cars: area/rectangularity/elongation filters; optional *Minimum Bounding Rectangle* to estimate orientation/length.
  - Trees: area threshold + smoothing; join with NDVI (if available) to validate vegetation.

- **Multi-scale inference**
  - Run SAM at two scales (e.g., 30 cm and a downsampled 60 cm) and **union** results to capture both tiny and context-aware segments.

- **Model choice / ensemble**
  - If the goal is **only cars** or **only trees**, consider running a second, **class-specific** model and **fusing** results with SAM outputs.

- **Quality metrics**
  - If you have ground truth, compute **Precision/Recall/F1** per class and per parameter set to select the best operating point.

---


