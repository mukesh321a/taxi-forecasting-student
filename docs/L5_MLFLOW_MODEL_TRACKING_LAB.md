# L5: Train and Compare Models with MLflow

- **Duration:** 45 to 60 minutes
- **Cloud required:** No
- **Outcome:** Tracked model runs and an evidence-based winner rationale.

## Task 1: Verify Training Inputs

From the repository root:

```powershell
Test-Path data\features\fare_train.parquet
Test-Path data\features\fare_val.parquet
Test-Path data\features\fare_test.parquet
Test-Path mlflow.db
```

Generate missing feature files with the L4 pipeline. `mlflow.db` is created when
training first logs to the local tracking store.

## Task 2: Predict the Comparison

Before training, record which candidate you expect to perform best and which
validation metric should decide the comparison.

## Task 3: Train a Small Fare Comparison

```powershell
.\.venv\Scripts\python.exe -m scripts.train_models --problem fare --n-trials 1 --no-register
```

Keep the run names, validation metrics, and selected candidate visible. The
`--no-register` option records runs without changing the model registry.

## Task 4: Inspect MLflow

Start the local UI in a separate terminal:

```powershell
.\.venv\Scripts\python.exe -m mlflow ui --backend-store-uri sqlite:///mlflow.db
```

Open `http://127.0.0.1:5000`. Compare at least two runs using their parameters,
validation metrics, and artifacts. Do not choose a winner from run duration or
the order in which runs appear.

## Task 5: Record the Result

Record the experiment and run identifiers, the metric values used for the
comparison, the preferred candidate, and one limitation. A one-trial teaching
run demonstrates tracking and comparison; it is not exhaustive tuning or
production approval.

## Completion Check

- The expected winner was recorded before training.
- At least two tracked runs were compared in MLflow.
- The winner rationale cites the same validation metric for each candidate.
- `--no-register` was distinguished from model registration.
- The limitation describes what a small local comparison cannot establish.

## Before the Next Lab

The finalized L5 recording intentionally leaves the registry unchanged. The
following is preparation for later labs, not an additional L5 task: before L6,
complete the fare registration prerequisite in [L6](L6_EXPLAINABILITY_LAB.md).
Before L7, register both demand and fare models using
[setup Phase 2](SETUP.md#3-training-and-registry). Skip repeat training only
when the required registered models and their artifacts already exist.