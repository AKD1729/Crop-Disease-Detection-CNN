# Crop Disease Detection Using CNN — Project Plan

**Domain:** Digital-data-ai
**Scope:** Two grain crops (Corn/Maize, Rice), single unified 10-class CNN model, built from scratch
**Orientation:** Scale and polish, coursework-driven (not field-robustness/cross-domain focused)

---

## Module 1 — Problem Definition

- **Task:** Single-label image classification
- **Classes (10 total):**
  - Corn: Cercospora/Gray Leaf Spot, Common Rust, Northern Leaf Blight, Healthy
  - Rice: Bacterial Leaf Blight, Brown Spot, Leaf Blast, Leaf Scald, Sheath Blight, Healthy
- **Model structure:** One unified model covering both crops (not two separate models)
- **Explicitly out of scope:** severe class-imbalance correction, cross-domain/field-robustness testing, severity estimation, multi-label classification

---

## Module 2 — Data Acquisition

| Crop | Source | Classes | Approx. Images | Condition |
|---|---|---|---|---|
| Corn (Maize) | Raw PlantVillage (GitHub: spMohanty/PlantVillage-Dataset) — not the pre-augmented Kaggle version, to avoid leakage | 4 | ~3,850 | Lab, plain background |
| Rice | Kaggle/AgML rice-disease-dataset | 6 | ~3,829 | Semi-field, more variable |
| **Combined** | | **10** | **~7,700** | |

**Note:** the pre-augmented "New Plant Diseases Dataset" on Kaggle was deliberately avoided as the corn source because its images are already augmented, risking near-duplicate leakage across splits.

---

## Module 3 — Preprocessing and EDA

1. **Deduplication** — run before anything else, especially checked on the rice set (more web/field-sourced, higher duplicate risk)
2. **Exploration** — class-count table across all 10 merged classes, visual sample inspection per class, image property checks (resolution, aspect ratio, format)
3. **Background handling** — left untouched; no segmentation, no center crop (kept simple, consistent with project scope)
4. **Split** — stratified, **70/15/15** (train/val/test), performed *before* augmentation
5. **Split caution** — checked for potential same-leaf/near-duplicate images across splits, especially in the rice source
6. **Imbalance handling** — no undersampling, no class dropping (imbalance is mild, ~2.4:1 at worst); addressed instead via class-weighted loss in Module 6

---

## Module 4 — Augmentation

Applied **only to the training split**, after the split is made.

| Technique | Setting |
|---|---|
| Horizontal flip | Yes |
| Vertical flip | No |
| Rotation | ±20–25° |
| Crop/zoom | 80–100% crop area scale (`RandomResizedCrop(scale=(0.8, 1.0))`, max ~15–20% edge crop / ~1.12× zoom) |
| Shear | Excluded |
| Photometric augmentation | Excluded entirely (geometric only, to avoid disturbing color-based disease signals) |
| Policy scope | Uniform — identical pipeline for both corn and rice |

---

## Module 5 — Model Development (Scratch CNN)

**Decision:** No pretrained/transfer learning — model built entirely from scratch, to learn CNN architecture and mechanics from first principles.

**Architecture:**
- Input: 224×224 RGB
- 4 convolutional blocks, filters doubling each block: 32 → 64 → 128 → 256
- Each block: Conv (3×3, stride 1) → BatchNorm → ReLU → MaxPool (2×2, stride 2)
- Spatial size progression: 224 → 112 → 56 → 28 → 14
- Global Average Pooling
- Dropout (0.5), applied once, before the final dense layer only
- Dense layer → 10 unnormalized logits (PyTorch `CrossEntropyLoss` expects logits during training)
- Softmax activation applied at inference time (converts raw logits to class probabilities)

**Reasoning basis:** block count chosen by balancing spatial-reduction limits (224×224 input) against pattern complexity of the task (disease texture/pattern recognition needs less depth than large-scale object classification) and overfitting risk relative to dataset size (~770 images/class average). Architecture is a starting point, expected to be adjusted based on real train/validation curve evidence rather than fixed in advance.

---

## Module 6 — Training and Tuning

**Optimizer: Adam (single optimizer for the entire training process)**

- Learning rate: 1e-3 (standard starting point for training from scratch with Adam), with reduce-on-plateau decay when validation loss stalls
- Betas: default (0.9, 0.999) — left untuned
- *(Earlier two-phase Adam→SGD+Nesterov approach was considered and dropped in favor of this simpler, single-optimizer setup; trade-off noted: Adam is faster/more forgiving to tune, though a well-tuned SGD run can sometimes generalize marginally better — acceptable given this project's scale-and-polish orientation)*

**Other settings:**
- Batch size: 32 (chosen relative to ~5,400-image training set, giving ~169 updates/epoch)
- Epochs: up to ~50–100, governed by early stopping (patience ~8–10 on validation loss)
- Loss function: Cross-entropy with class weights (addresses the mild corn-class imbalance)
- Reproducibility: fixed random seed, logged configuration per run

---

## Module 7 — Evaluation and Analysis

**Metrics:**
- Overall accuracy
- Per-class precision, recall, F1
- **Macro-F1** as the primary summary metric

**Confusion matrix — built with 4 explainability techniques:**
1. Row-normalized (percentages, not raw counts)
2. Crop-grouped class ordering (all 4 corn classes block, then all 6 rice classes block)
3. Heatmap color styling
4. Annotated explanations for notable off-diagonal confusions

**Explicitly deferred:** Grad-CAM (planned for a later update, not this initial build)

**Explicitly out of scope:** cross-domain testing, calibration analysis, perturbation/robustness testing

---

## Module 8 — Deployment

- **Format:** Web app via **Streamlit** (chosen for ease of build without deep web-dev experience)
- **Output (above threshold):** Top-1 prediction + confidence score only (no top-3)
- **Confidence threshold:** 60%
- **Below threshold:** Show only an uncertainty fallback message — no prediction displayed
- **Hard requirement:** inference-time preprocessing (resize, normalization) must exactly match training-time preprocessing from Module 3

---

## Module 9 — Documentation and Presentation

**Report structure:**
1. Problem statement (task, scope, success criteria)
2. Data (sources, citations, class counts, imbalance note)
3. Methodology (preprocessing, augmentation, architecture, training strategy — with reasoning, not just final values)
4. Results (accuracy, per-class F1, macro-F1, explainable confusion matrix)
5. Limitations (stated plainly — see below)
6. Future work (Grad-CAM, possible wheat addition, possible TFLite/offline export)
7. Demo (Streamlit screenshots — both a confident prediction and a below-threshold fallback example)

**Supporting visuals:** architecture diagram, pipeline diagram, training curves, confusion matrix, demo screenshots

**Reproducibility package:** README, environment/dependency list, fixed seed noted, dataset source links

---

## Known Limitations (for honest disclosure in the report)

1. Lab (corn) vs. semi-field (rice) condition mismatch — possible shortcut learning via background differences
2. No cross-domain/real-world generalization testing performed
3. Narrow disease coverage per crop — diseases/conditions outside the 10 classes will be misclassified into a known class
4. Datasets likely emphasize clear, developed symptoms — early-stage disease detection is likely overstated
5. Possible residual leakage risk if corn source handling isn't carefully verified
6. Geographic/cultivar narrowness of source datasets
7. No severity information — only disease presence/absence per class
8. Dataset licensing/citation requirements to be honored in documentation

---

## Status

All 9 modules fully discussed and locked. Ready to move from planning into execution, starting with Module 3 (data download, deduplication, exploration).

---

## Revision Log

- Module 6 training strategy simplified from a two-phase Adam→SGD+Nesterov approach to **Adam-only** (LR 1e-3, reduce-on-plateau decay). All other modules unaffected by this change.
- Module 4 augmentation scale clarified: `RandomResizedCrop(scale=(0.8, 1.0))` represents crop area ratio (0.8–1.0) achieving ~1.12× zoom and ~15–20% edge crop without out-of-bounds padding.
- Module 5 architecture clarified: Dense layer returns 10 raw logits for numerical stability with PyTorch `CrossEntropyLoss`; Softmax is applied externally in the inference pipeline.
- Module 8 deployment strengthened: Out-of-Distribution (OOD) chromatic foliar filter and prediction entropy thresholding added to prevent non-leaf false positives.
