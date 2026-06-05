    ## Project Structure

This repository is organized to support the full lifecycle of an early sepsis prediction system: data ingestion, exploratory analysis, temporal feature engineering, supervised model training, calibration, explainability, evaluation, testing, and Streamlit deployment. The structure keeps experimental work in notebooks, reusable logic in `src/`, trained artifacts in `models/`, and outputs such as plots and metrics in `reports/`, which is a common and maintainable layout for Python ML projects.

```bash
earlysepsis/
│
├── README.md
├── requirements.txt
├── .gitignore
├── .env.example
│
├── data/
│   ├── raw/
│   │   ├── training/
│   │   ├── validation/
│   │   └── sample_patient/
│   ├── interim/
│   └── processed/
│
├── notebooks/
│   ├── 01_data_understanding.ipynb
│   ├── 02_eda_missingness.ipynb
│   ├── 03_feature_engineering.ipynb
│   ├── 04_baseline_model.ipynb
│   ├── 05_boosting_model.ipynb
│   ├── 06_calibration_shap.ipynb
│   └── 07_error_analysis.ipynb
│
├── src/
│   └── earlysepsis/
│       ├── __init__.py
│       ├── data/
│       ├── features/
│       ├── models/
│       ├── evaluation/
│       ├── explainability/
│       ├── app/
│       ├── pipelines/
│       ├── utils/
│       └── config/
│
├── models/
│   ├── baseline/
│   ├── xgboost/
│   ├── calibrated/
│   └── artifacts/
│
├── reports/
│   ├── figures/
│   ├── tables/
│   └── metrics/
│
├── tests/
│   ├── test_data_loader.py
│   ├── test_features.py
│   ├── test_training.py
│   ├── test_inference.py
│   └── test_app_smoke.py
│
├── scripts/
│   ├── run_training.py
│   ├── run_inference.py
│   ├── generate_sample_input.py
│   └── export_metrics.py
│
└── deployment/
    ├── Dockerfile
    └── streamlit_config.toml
```

## Root files

- `README.md` — Main project documentation, problem motivation, setup steps, usage instructions, and results summary. A README is the standard entry point for understanding and running a repository.
- `requirements.txt` — Lists Python dependencies needed to reproduce the project environment, including data processing, modeling, explainability, and Streamlit packages.
- `.gitignore` — Prevents unnecessary or sensitive files such as virtual environments, cache folders, raw model binaries, and local secrets from being committed to Git.
- `.env.example` — Template for environment variables such as data paths, app settings, or optional deployment secrets; users copy this to `.env` and fill in local values.

## `data/`

This folder stores all dataset files at different stages of the ML pipeline so raw data remains untouched and processed data can be reproduced cleanly, which is a standard practice in data science project organization.

- `data/raw/` — Original downloaded dataset files exactly as obtained from the source.
- `data/raw/training/` — Training patient files used for model development.
- `data/raw/validation/` — Validation split used for threshold tuning, calibration, and model comparison.
- `data/raw/sample_patient/` — Small sample patient records used for debugging, demos, and app input testing.
- `data/interim/` — Intermediate outputs created during preprocessing, such as merged hourly tables, cleaned records, or partially engineered features.
- `data/processed/` — Final model-ready datasets after imputation strategy, missingness indicators, rolling-window features, deltas, and label alignment have been applied.

## `notebooks/`

This folder is for experimentation, exploratory data analysis, and research workflow. Notebooks are useful for iterative analysis, but production logic should eventually move into reusable Python modules inside `src/`.

- `01_data_understanding.ipynb` — Initial inspection of dataset format, patient-level files, columns, label distribution, and clinical variable overview.
- `02_eda_missingness.ipynb` — Analysis of missing values, feature sparsity, clinically meaningful absence of measurements, and sepsis vs non-sepsis distributions.
- `03_feature_engineering.ipynb` — Prototyping of temporal features such as rolling means, rolling standard deviations, trend slopes, deltas, and time-since-last-measurement features.
- `04_baseline_model.ipynb` — Interpretable baseline modeling, typically logistic regression with class weighting and basic evaluation.
- `05_boosting_model.ipynb` — Training and tuning of stronger models such as Random Forest, XGBoost, or LightGBM on engineered features.
- `06_calibration_shap.ipynb` — Probability calibration, SHAP analysis, feature importance interpretation, and local explanation testing.
- `07_error_analysis.ipynb` — Investigation of false positives, false negatives, lead time behavior, subgroup performance, and failure modes.

## `src/earlysepsis/`

This is the main Python package containing reusable source code. Keeping core logic here instead of inside notebooks or the Streamlit file improves maintainability, testing, and deployment readiness. 

### `src/earlysepsis/__init__.py`
Marks `earlysepsis` as a Python package so modules can be imported cleanly across the project.

### `src/earlysepsis/data/`
Handles dataset reading, schema checks, and patient-level splits.

- `loader.py` — Reads raw patient files, parses hourly records, and returns consistent pandas DataFrames.
- `schema.py` — Defines expected columns, data types, feature groups, and label conventions.
- `validation.py` — Validates raw data quality, missing columns, invalid value ranges, duplicate rows, and malformed records.
- `split.py` — Creates train/validation/test splits at the patient level to avoid leakage across time rows.

### `src/earlysepsis/features/`
Implements temporal and clinically motivated feature engineering.

- `build_features.py` — Main feature assembly module that combines raw variables, rolling statistics, trend features, missingness features, and SOFA-related features into one model-ready table.
- `rolling.py` — Computes rolling mean, std, min, max, and other window-based summaries over the last few hours.
- `trends.py` — Computes rates of change, deltas, slopes, and direction-of-change features over time.
- `missingness.py` — Adds missingness indicators, forward-fill logic, decay features, and time-since-last-observed measurements.
- `sofa.py` — Builds SOFA or SOFA-like organ dysfunction components from available vitals and lab values.

### `src/earlysepsis/models/`
Contains supervised learning model code.

- `train_logreg.py` — Trains the logistic regression baseline for interpretability and comparison.
- `train_rf.py` — Trains a Random Forest model for nonlinear baseline performance and feature importance exploration.
- `train_xgb.py` — Trains the main boosted tree model such as XGBoost or LightGBM for strong predictive performance.
- `calibrate.py` — Applies probability calibration methods such as Platt scaling or isotonic regression.
- `predict.py` — Loads trained pipelines and generates risk probabilities for new patient inputs.
- `thresholding.py` — Selects decision thresholds based on recall, specificity, AUPRC, or clinical operating targets.

### `src/earlysepsis/evaluation/`
Evaluates model quality using clinically meaningful metrics.

- `metrics.py` — Computes AUROC, AUPRC, sensitivity, specificity, precision, recall, F1, and confusion matrices.
- `plots.py` — Generates ROC curves, PR curves, feature importance plots, missingness charts, and model comparison visuals.
- `calibration.py` — Evaluates and visualizes calibration quality using calibration curves and related metrics.
- `leadtime.py` — Measures how many hours before sepsis onset the model is able to raise an alert.

### `src/earlysepsis/explainability/`
Makes the model clinically interpretable.

- `shap_utils.py` — Computes SHAP values for tree-based models and prepares global and patient-level explanations.
- `clinical_text.py` — Converts raw explanation outputs into clinician-friendly language such as “rising lactate and falling MAP contributed to increased risk.”

### `src/earlysepsis/app/`
Contains the Streamlit application and UI-related logic. Streamlit best practice is to keep the app layer lightweight and move non-UI functionality into reusable modules.

- `streamlit_app.py` — Main Streamlit entry point that loads the trained model, accepts user inputs, runs inference, and displays outputs.
- `ui_components.py` — Reusable UI elements such as metric cards, warning banners, and result panels.
- `patient_form.py` — Form or upload interface for entering a patient’s recent 6-hour vitals and lab history.
- `visualizations.py` — App-side plotting utilities for SHAP bars, risk curves, and feature summaries.

### `src/earlysepsis/pipelines/`
Orchestrates end-to-end workflows.

- `training_pipeline.py` — Full training pipeline from loading processed data to fitting models and saving artifacts.
- `inference_pipeline.py` — End-to-end prediction flow used by the app for new patient data.
- `preprocess_pipeline.py` — Shared preprocessing sequence used consistently in both training and inference.

### `src/earlysepsis/utils/`
Stores general helper utilities used across modules.

- `config.py` — Loads YAML or environment-based configuration.
- `logger.py` — Central logging setup for training runs, app events, and debugging.
- `paths.py` — Centralized project path definitions so file locations are not hardcoded everywhere.
- `helpers.py` — Miscellaneous shared helper functions.

### `src/earlysepsis/config/`
Stores configuration files that make experiments easier to reproduce without changing code.

- `model_config.yaml` — Hyperparameters, model names, class-weight settings, and calibration options.
- `feature_config.yaml` — Feature selection options, rolling window sizes, missingness settings, and inclusion of SOFA-related variables.
- `app_config.yaml` — Streamlit-specific settings such as feature display names, alert thresholds, and UI behavior.

## `models/`

This folder stores serialized model outputs and related artifacts produced during training.

- `models/baseline/` — Saved logistic regression and simple baseline model files.
- `models/xgboost/` — Saved boosted tree models and tuning outputs.
- `models/calibrated/` — Calibrated versions of trained models used for reliable probability outputs.
- `models/artifacts/` — Supporting objects such as scalers, encoders, selected feature lists, preprocessing pipelines, and metadata.

## `reports/`

Stores project outputs that are meant to be inspected by humans rather than consumed directly by code.

- `reports/figures/` — Plots for EDA, missingness analysis, ROC/PR curves, calibration curves, SHAP summaries, and dashboard screenshots.
- `reports/tables/` — Comparison tables for models, features, thresholds, and subgroup results.
- `reports/metrics/` — Saved metric summaries in CSV or JSON format for experiment tracking.

## `tests/`

This folder contains automated tests, which are important for ensuring the preprocessing logic, feature generation, and app inference do not silently break during refactoring. Dedicated test directories are standard in Python projects.

- `test_data_loader.py` — Tests file reading, schema consistency, and patient loading behavior.
- `test_features.py` — Tests rolling statistics, trend generation, missingness features, and output shapes.
- `test_training.py` — Tests that training functions run correctly and save expected artifacts.
- `test_inference.py` — Tests inference on sample patient inputs and validates probability outputs.
- `test_app_smoke.py` — Basic smoke test to ensure the Streamlit app launches and core imports work.

## `scripts/`

This folder contains convenient command-line scripts for repeated actions. Keeping these tasks in scripts avoids depending on notebooks for operational workflows. 
- `run_training.py` — Starts the end-to-end training pipeline from processed data to saved models.
- `run_inference.py` — Runs prediction on a new patient file or batch of files.
- `generate_sample_input.py` — Creates a valid demo patient input file for testing the app.
- `export_metrics.py` — Exports final metrics, threshold tables, or experiment summaries to `reports/`.

## `deployment/`

This folder contains files required to package and deploy the application.

- `Dockerfile` — Defines the containerized environment for reproducible deployment of the model and Streamlit app, a common step for production-ready ML dashboards. 
- `streamlit_config.toml` — Streamlit server configuration such as theme, port, headless mode, and UI settings. 

## Workflow summary

The intended project flow is:

1. Download and inspect raw ICU data in `data/raw/`.
2. Explore and prototype ideas in `notebooks/`.
3. Move stable logic into `src/earlysepsis/`.
4. Train and calibrate supervised models through `pipelines/` and `scripts/`.
5. Save artifacts in `models/`.
6. Store figures and result summaries in `reports/`.
7. Validate reliability with `tests/`.
8. Serve predictions and explanations through the Streamlit app in `src/earlysepsis/app/`. 

***

## Shorter version

If you want a shorter README subsection, use this:

- `data/` — Raw, interim, and processed ICU sepsis datasets.
- `notebooks/` — EDA, missingness analysis, feature engineering, modeling, SHAP, and error analysis experiments.
- `src/earlysepsis/data/` — Data loading, schema checks, and train/validation splitting.
- `src/earlysepsis/features/` — Rolling windows, deltas, missingness features, and SOFA-related engineering.
- `src/earlysepsis/models/` — Logistic regression, Random Forest, XGBoost, calibration, prediction, and threshold selection.
- `src/earlysepsis/evaluation/` — Metrics, plots, calibration analysis, and lead-time evaluation.
- `src/earlysepsis/explainability/` — SHAP computation and clinician-friendly explanation text.
- `src/earlysepsis/app/` — Streamlit dashboard and UI logic.
- `src/earlysepsis/pipelines/` — Shared training, preprocessing, and inference workflows.
- `src/earlysepsis/utils/` — Config, logging, paths, and helper utilities.
- `models/` — Saved model binaries and preprocessing artifacts.
- `reports/` — Output figures, tables, and performance summaries.
- `tests/` — Unit and smoke tests.
- `scripts/` — CLI runners for training, inference, sample input generation, and metric export.
- `deployment/` — Docker and Streamlit deployment configuration.