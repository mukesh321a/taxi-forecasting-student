# L4: Build Features and Demand Data

- **Duration:** 35 to 45 minutes
- **Cloud required:** No
- **Outcome:** A verified feature file and pipeline summary with an evidence-based finding.

## Expected Result

Before running the pipeline, record what you expect it to do:

```text
The pipeline will reuse the configured January 2024 source files, clean the
trips, create hourly demand, generate demand and fare features, split both
tables by time, and write a summary.
```

## Task 1: Verify Inputs and Configuration

1. Confirm the repository root and project interpreter.
2. Confirm both January 2024 files exist under `data/raw`.
3. Inspect the configured period, directories, cleaning bounds, split fractions,
   lags, and rolling windows in `config/config.yaml`.

If raw inputs are missing on a fresh checkout, acquire and build them once with
the following command (requires internet), then use Task 2 for the cached rerun:

```powershell
.\.venv\Scripts\python.exe -m scripts.run_pipeline --year 2024 --month 1
```

## Task 2: Run the Pipeline

From the repository root:

```powershell
.\.venv\Scripts\python.exe -m scripts.run_pipeline --skip-download
```

Record whether the command completed and identify the last successful stage.

## Task 3: Inspect Persisted Evidence

Inspect:

- `data/processed/pipeline_summary.json`
- `data/processed/trips_clean.parquet`
- `data/processed/demand_hourly.parquet`
- `data/features/demand_features.parquet`
- `data/features/fare_features.parquet`
- The six demand and fare train, validation, and test files

Record the cleaning counts, demand dimensions, feature shapes, and chronological
date ranges shown by the files.

## Task 4: Run Consistency Checks

Confirm that:

- Clean rows match the pipeline summary.
- Fare-feature rows match clean-trip rows.
- Summed hourly pickup counts match clean-trip rows.
- Demand zone-hour keys are unique.
- Demand and fare feature rows match the pipeline summary.

## Task 5: Explain the Result

### Evidence or output paths

```text

```

### Main finding

```text

```

### Decision or next action

```text

```

### Limitation

```text

```

## Completion Check

- The expected result was recorded before execution.
- The pipeline command and final result are visible.
- At least one generated feature file and the pipeline summary were inspected.
- The consistency checks passed or any failure was explained.
- The finding cites observed evidence rather than inference alone.
- The next action follows from the evidence.
- The limitation stays within what a local January 2024 run can establish.