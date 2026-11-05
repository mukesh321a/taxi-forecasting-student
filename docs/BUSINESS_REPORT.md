# Business Impact Report — NYC Taxi Forecasting Platform

## Executive summary

The NYC Taxi Forecasting platform implements two models — an **hourly
demand forecaster** (pickups per taxi zone) and a **trip fare predictor** —
served through a single FastAPI application on AWS ECS Fargate. Together they
turn historical TLC trip data into forward-looking operational signals:

- **Where and when demand is forecast**, supporting driver-positioning decisions.
  Reduced wait times remain a proposed benefit, not a measured outcome here.
- **Expected trip fares**, supporting rider price transparency, ETA/fare quotes
  and revenue planning.

The platform implements AWS CodeBuild CI/CD, optional scheduled drift detection,
SHAP explainability, fairness reporting, and an interactive dashboard. Dated
project evidence records deployed SageMaker retraining and a weekly schedule
with deployment disabled. These are historical staging observations, not a fresh
verification of AWS state or production readiness. See
[PROJECT_ARCHITECTURE.md](PROJECT_ARCHITECTURE.md) for the implemented paths.

## Business value

| Lever | How the models help | Indicative impact |
|-------|--------------------|-------------------|
| Driver utilisation | Demand forecast steers supply toward high-demand zones/hours | Fewer idle miles, higher trips/hour |
| Passenger experience | Supply pre-positioned where demand is predicted | Shorter wait times |
| Revenue predictability | Fare model + demand forecast → revenue estimates per zone/hour | Better shift & pricing planning |
| Operational planning | 1–72h recursive demand horizon | Staffing and incentive targeting |

> Revenue and utilisation figures shown in the dashboard are **illustrative**,
> derived from live forecasts times an assumed average fare. They demonstrate
> the KPI framework rather than audited financials.

## Model performance

Historical example metrics reported for 6 September 2026, not live service
accuracy or results from your checkout. The original evaluated bundles are not
distributed. [Generate your own cards](model_cards/README.md) before making
claims about your trained models:

| Model | RMSE | MAE | MAPE | R² |
|-------|------|-----|------|----|
| Demand forecaster | 8.70 pickups | 2.54 | 40.2% | 0.971 |
| Fare predictor | $3.47 | $0.82 | 8.7% | 0.955 |

Both cards identify `LGBMRegressor`. Training compares XGBoost and LightGBM, but
these snapshot metrics describe the selected LightGBM models. The cards report
registry version `n/a`; they do not establish an immutable registry version or
that the currently served model is the same evaluated model.

## Technical deep-dive

**Architecture.** A self-contained container bakes the MLflow registry, models
and demand history, so the API needs no external database or object store on the
request path; startup reads its bundled SQLite registry and local artifacts. It
runs on **ECS Fargate behind an Application Load Balancer**, pulling the image
from **ECR**. Endpoints: `/predict/fare`, `/predict/demand`, `/forecast/demand`,
`/explain/fare`, `/explain/demand`, `/health`, `/metrics` (Prometheus).

**MLOps.**
- **CI/CD** — AWS CodeBuild lints and tests webhook builds; the deploy-branch
  path and manual starts without webhook events also build, push to ECR, and
  request an ECS rollout. Stability and smoke checks are separate.
- **Monitoring (Phase 5)** — prediction-capture logs are delivered via Firehose
  to S3 when monitoring is enabled. An optional weekly drift job compares common
  numeric input features with the training reference, publishes CloudWatch
  metrics, and supports SNS alerts. It does not join delayed actuals, measure
  subgroup error, or automatically start retraining. The separate weekly
  retraining schedule trains and evaluates with deployment disabled.
- **Responsible AI (Phase 6)** — model cards, subgroup fairness analysis and
  SHAP explanations (offline plots + live `/explain` endpoints).

**Showcase (Phase 7).** A Streamlit dashboard consumes the API to present
demand forecasts, fare predictions with live SHAP explanations, fairness
reports and business KPIs. Predictions call the API, while fairness and drift
panels read stored files with separate freshness limits. It is deployable to
AWS as a second Fargate service.

## Known limitations

- **Subgroup error.** Fare borough RMSE is 18.418 for `Unknown` (815 rows)
  versus 2.268 for Manhattan (245,640 rows), a ratio of 8.12. This is not an
  aggregated outside-Manhattan comparison. Airport/non-airport fare RMSE has a
  2.54 ratio. Demand RMSE varies with zone volume and target scale. These
  held-out findings require renewed subgroup evaluation with actual outcomes;
  feature-drift monitoring does not track their error ratios automatically.
- **Coverage and identity.** One-group comparisons cannot establish parity.
  Missing immutable model identity limits reproducibility of the snapshot.
- **Illustrative KPIs.** Revenue estimates use a flat average fare, not the
  fare model per trip.
- **Feature scope.** Demand uses calendar, zone, lag, and rolling features;
  fare also uses distance, duration, passenger count, and pickup/drop-off zones.
  Serving estimates duration when it is omitted. Weather, events, and traffic
  are not included.

## Future roadmap

1. **External signals** — weather, large events and traffic feeds to lift
   accuracy, especially outside Manhattan.
2. **Fairness remediation** — targeted data collection / reweighting for
   underperforming boroughs and airport zones.
3. **Precomputed forecasts** — schedule the batch forecaster to S3/SQLite so
   the dashboard renders city-wide multi-hour views instantly.
4. **Multi-modal expansion** — extend from taxis to for-hire-vehicle and
   micro-mobility demand.
