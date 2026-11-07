# Taxi Forecasting

End-to-end machine learning system for two New York City taxi prediction tasks, built on NYC TLC trip records:

| Task | Target | Granularity |
| --- | --- | --- |
| **Demand forecasting** | Number of pickups | One pickup zone (`PULocationID`) × one hour |
| **Fare prediction** | `fare_amount` (metered fare only, not the full passenger bill) | One trip |

The repository covers the full lifecycle: data ingestion and feature engineering, XGBoost/LightGBM training with Optuna tuning, MLflow experiment tracking and model registry, a FastAPI inference service, recursive batch forecasting, SHAP explanations, subgroup (fairness) evaluation, drift detection, a Streamlit dashboard, and optional AWS deployment (ECR, ECS, CodeBuild, Terraform).

> **No data or trained models are committed.** Datasets, MLflow runs, model bundles and evidence reports are generated locally. The API cannot return predictions until you have trained and registered both models and saved the demand history.

---

## Contents

- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Quick Start](#quick-start)
- [End-to-End Workflow](#end-to-end-workflow)
- [Configuration](#configuration)
- [Reference Results](#reference-results)
- [Testing and Code Quality](#testing-and-code-quality)
- [Docker Images](#docker-images)
- [AWS Deployment (Optional)](#aws-deployment-optional)
- [Repository Layout](#repository-layout)
- [Documentation](#documentation)
- [Limitations and Responsible Use](#limitations-and-responsible-use)

---

## Architecture

```text
 NYC TLC trip data
        │
        ▼
 src/data ──► src/features ──► time-based train / val / test split
                                        │
                                        ▼
                     src/models: Optuna tuning → XGBoost & LightGBM
                                        │
                                        ▼
                     MLflow tracking (mlflow.db + mlruns/) and model registry
                       • taxi-demand-forecaster
                       • taxi-fare-predictor
                                        │
              ┌─────────────────────────┼─────────────────────────┐
              ▼                         ▼                         ▼
     FastAPI service           Batch forecasts             Responsible AI & monitoring
     (src/serving)             → predictions.db            SHAP, subgroup metrics, drift
     /metrics (Prometheus)       demand_forecasts          (src/responsible, src/monitoring)
              │
              ▼
     Streamlit dashboard (dashboard/) — calls the API over HTTP and reads local reports
```

See [System architecture](docs/ARCHITECTURE.md) for component-level detail.

## Tech Stack

| Area | Libraries / services |
| --- | --- |
| Data | pandas, NumPy, PyArrow (Parquet), `holidays` |
| Modelling | scikit-learn, XGBoost, LightGBM, Optuna |
| Tracking & explainability | MLflow (SQLite backend), SHAP, Matplotlib |
| Serving | FastAPI, Uvicorn, httpx, Prometheus FastAPI Instrumentator |
| Dashboard | Streamlit, Plotly |
| Monitoring & retraining | Evidently, boto3, SageMaker SDK (separate requirements file) |
| Quality | pytest, pytest-cov, Ruff |
| Cloud (optional) | Docker, Amazon ECR, ECS, CodeBuild, S3, Terraform |

All Python dependencies are pinned. Python **3.11** is required.

## Quick Start

> Using the preinstalled Windows course image? Follow the [AWS account and image-access guide](docs/STUDENT_AWS_IMAGE_ACCESS.md) instead of creating a new environment.

From the repository root in Windows PowerShell:

```powershell
py -3.11 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m pytest
```

On macOS/Linux:

```bash
python3.11 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pytest
```

The test suite runs on synthetic data and needs no downloads, trained models or AWS access.

Optional dependency sets:

```bash
pip install -r requirements-dashboard.txt     # Streamlit dashboard only
pip install -r requirements-monitoring.txt    # Drift jobs, SageMaker retraining (with requirements.txt)
```

## End-to-End Workflow

The full command sequence is documented in [Local setup and commands](docs/SETUP.md). At a high level:

1. **Download** a month of TLC trip data (default: January 2024, set by `TLC_TRIP_YEAR` / `TLC_TRIP_MONTH`).
2. **Clean and aggregate** trips into hourly zone demand (`data/processed/demand_hourly.parquet`) and trip-level fare features.
3. **Engineer features and split by time** so validation and test periods follow the training period.
4. **Tune and train** XGBoost and LightGBM for each task; every run is logged to MLflow.
5. **Register** the best model per task in the MLflow registry (`taxi-demand-forecaster`, `taxi-fare-predictor`).
6. **Serve** both models through the FastAPI service — see [API request examples](docs/API_EXAMPLES.md).
7. **Batch-forecast** demand recursively for all zones; results are written to `predictions.db` (`demand_forecasts` table: zone, hour, predicted pickups, generation timestamp, model version).
8. **Explain and evaluate** with SHAP, subgroup metrics and drift reports.
9. **Explore** results in the Streamlit dashboard. Set `API_BASE_URL` in the shell before starting it — `.env` alone does not set it for the dashboard.

> The default forecast horizon continues from the **latest stored history**, not the current wall-clock time.

## Configuration

Copy `.env.example` to `.env` for local settings. Never commit real credentials; `.env` is gitignored.

| Variable | Default | Purpose |
| --- | --- | --- |
| `MLFLOW_TRACKING_URI` | `sqlite:///mlflow.db` | MLflow backend store |
| `MLFLOW_EXPERIMENT_DEMAND` | `taxi-demand-forecasting` | Demand experiment name |
| `MLFLOW_EXPERIMENT_FARE` | `taxi-fare-prediction` | Fare experiment name |
| `MLFLOW_ARTIFACT_ROOT` | *(empty)* | Optional artifact location override |
| `TLC_TRIP_YEAR`, `TLC_TRIP_MONTH` | `2024`, `1` | Source data period |
| `DATA_RAW_DIR`, `DATA_PROCESSED_DIR`, `DATA_FEATURES_DIR` | `data/raw`, `data/processed`, `data/features` | Local data directories |
| `AWS_REGION` | `us-east-1` | Region for optional cloud tooling |
| `DATALAKE_BUCKET`, `DATALAKE_PREFIX`, `MONITORING_BUCKET`, `CAPTURE_PREFIX`, `SAGEMAKER_ROLE_ARN`, `ALERT_EMAIL` | *(empty)* | Cloud destinations — set only when deploying |

Infrastructure schedules are configured through Terraform variables, not `.env`.

## Reference Results

Metrics from one local training run on January 2024 data, as logged in MLflow. They are a reference point only — your results will differ with the data period, split and tuning budget.

**Demand (pickups per zone-hour)**

| Model | Test RMSE | Test MAE | Test R² | Test MAPE |
| --- | --- | --- | --- | --- |
| **LightGBM** (registered) | 8.70 | 2.54 | 0.971 | 40.2% |
| XGBoost | 9.09 | 2.64 | 0.968 | 41.8% |

**Fare (`fare_amount`, USD)**

| Model | Test RMSE | Test MAE | Test R² | Test MAPE |
| --- | --- | --- | --- | --- |
| **LightGBM** (registered) | 3.47 | 0.82 | 0.955 | 8.7% |
| XGBoost | 3.77 | 0.84 | 0.947 | 7.3% |

Demand MAPE is high despite a strong R² because many zones have very low hourly counts, where small absolute errors become large percentage errors. Prefer MAE/RMSE for demand, and review per-zone errors before relying on forecasts for low-volume zones.

## Testing and Code Quality

```bash
ruff check .                                              # lint (pycodestyle, pyflakes, isort, pyupgrade, bugbear)
pytest --cov=src --cov-report=term-missing                # tests with coverage
```

- Tests live in `tests/` and are offline unit and integration checks.
- Coverage is measured over `src/`; the build **fails below 75%** (`pyproject.toml`).
- Ruff targets Python 3.11 with a 100-character line length; `mlruns`, `data`, `artifacts` and `notebooks` are excluded.

## Docker Images

Two images are built from `docker/`:

| Image | Dockerfile | Contents |
| --- | --- | --- |
| **API** | `docker/Dockerfile` | Self-contained: application code plus baked-in `mlruns/`, a container-portable `mlflow.container.db`, and `data/processed/demand_hourly.parquet` |
| **Dashboard** | `docker/Dockerfile.dashboard` | Streamlit app plus static model cards, fairness and drift reports, and the taxi zone lookup — no models or MLflow |

Before building the API image locally, generate the portable MLflow database so registry paths resolve inside the container:

```bash
python scripts/make_mlflow_portable.py --src mlflow.db --dest mlflow.container.db --new-base "file:///app/mlruns"
docker build -f docker/Dockerfile -t taxi-forecasting-api .
```

`.dockerignore` keeps the build context lean: raw data, features, notebooks, tests, the local `mlflow.db` and `predictions.db` are excluded.

## AWS Deployment (Optional)

The cloud templates are **not connected** to any account, remote repository or existing deployment. Applying them with your own credentials creates billable resources — review [Cloud cost boundaries](docs/COST_OPTIMIZATION.md) first.

**API pipeline — `buildspec.yml`** (CodeBuild project defined in `infra/terraform/cicd.tf`)

- Every webhook-triggered build installs dependencies, runs Ruff, and runs pytest with coverage (`coverage.xml` is published as a build artifact).
- Builds triggered by a push to `DEPLOY_BRANCH`, and manually started builds, also:
  1. Pull the gitignored `mlruns/`, `mlflow.db` and demand-history Parquet from `s3://$ARTIFACTS_BUCKET/`
  2. Generate the container-portable MLflow database
  3. Build and push the image to ECR as `$IMAGE_TAG` (and as `latest` when the tag differs)
  4. Force a new deployment of the ECS service
- Required environment variables: `AWS_REGION`, `ECR_REPOSITORY_URL`, `ECS_CLUSTER`, `ECS_SERVICE`, `ARTIFACTS_BUCKET`, `IMAGE_TAG`, `DEPLOY_BRANCH`.
- Seed the S3 artifacts once with `scripts/aws/upload_artifacts.ps1`.

**Dashboard pipeline — `buildspec-dashboard.yml`**

- Run on demand against the same CodeBuild project:
  ```bash
  aws codebuild start-build --project-name <project> --buildspec-override buildspec-dashboard.yml \
    --environment-variables-override name=DASH_ECR_URL,value=<url> name=DASH_ECS_SERVICE,value=<service> name=DASH_IMAGE_TAG,value=<tag>
  ```
- Pulls static reports from `s3://$ARTIFACTS_BUCKET/dashboard/`, builds the image, runs the `scripts/aws/verify_dashboard.py` smoke test inside the container, and only then pushes and rolls the ECS service.
- Seed the S3 artifacts with `scripts/aws/upload_dashboard_artifacts.ps1`.

Step-by-step guides: [Deploy the API to AWS with CodeBuild](docs/STUDENT_AWS_DEPLOYMENT_WALKTHROUGH.md) and [Optional AWS infrastructure](infra/terraform/README.md).

## Repository Layout

| Path | Responsibility |
| --- | --- |
| `src/data/`, `src/features/` | Download, clean, aggregate, engineer features, split by time |
| `src/models/` | Tune, train, evaluate, explain and register models |
| `src/serving/`, `clients/python/` | API contracts, inference, batch forecasting, HTTP client |
| `src/monitoring/`, `src/responsible/` | Drift, performance comparisons, SHAP, subgroup metrics |
| `scripts/` | Command-line entry points (including `scripts/aws/` deployment helpers) |
| `dashboard/` | Streamlit views that call the API and read local reports |
| `tests/` | Offline unit and integration tests |
| `notebooks/` | Data exploration (committed without saved outputs) |
| `docs/` | Setup, architecture, API, learner activities and reports |
| `docker/` | API and dashboard Dockerfiles |
| `infra/terraform/` | Optional AWS infrastructure, including CI/CD |
| `retraining/` | Optional cloud retraining pipeline |
| `buildspec.yml`, `buildspec-dashboard.yml` | AWS CodeBuild specifications |
| `requirements*.txt` | Pinned dependencies for core, dashboard and monitoring |

## Documentation

**Getting started**
- [Local setup and commands](docs/SETUP.md)
- [API request examples](docs/API_EXAMPLES.md)
- [Request and launch the Windows course image](docs/STUDENT_AWS_IMAGE_ACCESS.md)

**Design and reference**
- [System architecture](docs/ARCHITECTURE.md)
- [Recorded architecture walkthrough](docs/PROJECT_ARCHITECTURE.md)
- [Data scope and model limitations](docs/DATA_STATEMENT.md)
- [Historical business report](docs/BUSINESS_REPORT.md)

**Cloud**
- [Deploy the API to AWS with CodeBuild](docs/STUDENT_AWS_DEPLOYMENT_WALKTHROUGH.md)
- [Optional AWS infrastructure](infra/terraform/README.md)
- [Cloud cost boundaries](docs/COST_OPTIMIZATION.md)

**Course material**
- [Recording companion and learner activity map](docs/RECORDING_COMPANION.md) — links all L2–L12 labs; L1 uses setup and tests. Videos are distributed separately. Lab files, reviewed diagrams and the exploration notebook keep the paths shown in the recordings. AWS activities require your own authorised configuration or the documented local fallback.

## Limitations and Responsible Use

- **Historical evaluation does not establish current production quality.** Metrics reflect the training data period only.
- **Forecasts follow stored history**, not the current clock; refresh the demand history before generating new forecasts.
- **Fare predictions cover `fare_amount` only** — tolls, taxes, surcharges and tips are excluded.
- **Review model errors, duration estimation, drift and subgroup disparities** before using predictions for operational or financial decisions. See [Data scope and model limitations](docs/DATA_STATEMENT.md).
- Docker, Terraform and AWS are optional; the project runs fully locally. No GitHub Actions workflows are included. The only automated release path is the CodeBuild webhook, and it exists only if you apply the Terraform in `infra/terraform/`.
