# Finalized Recordings and Current Documentation

All R and L presentations and narration are finalized. These documents support
the recordings; they do not replace the spoken material or change recorded
tasks. Setup prerequisites and current-project corrections are identified
separately so learners can follow the same activity on a fresh checkout.

## Fresh Checkout Boundaries

Videos are supplied separately. This checkout includes the learner labs,
reviewed diagrams, and an output-free notebook at the paths used in the recordings.
Follow [setup](SETUP.md) before executing an activity. Notebook packages are
included in core requirements; the dashboard and managed monitoring have separate
requirements.

Generate datasets, registry entries, model cards, and reports yourself. Recorded
results are historical examples, not required acceptance values. Your notebook
starts without saved outputs. Use the recorded lab command for each activity;
setup provides prerequisites, not a replacement for that command.

AWS demonstrations do not grant access to the recorded account. Use an authorized
account and your own Terraform state, or the documented local fallback. The L11
publishing helper writes to S3; L12 downloads can replace its isolated local cache.
Do not run either merely to check syntax. Resource names shown in labs are examples;
substitute your own. Never share credentials or state files.

The source-wide lint failures seen in R8 are not deliberately reproduced here.
Judge your current lint/test output independently. The optional notebook rebuild
command regenerates the supplied notebook; it is not needed to open it and can
overwrite your work.

The AWS diagrams are retained to match the recordings. Their HTTPS labels do not
configure TLS: this Terraform stack uses HTTP port 80. Read the L9 corrections;
the weekly retraining route disables deployment and does not require managed
MLflow. The business report contains historical example results, not shipped
model evidence.

## Reading Map

R recordings explain concepts and demonstrations. L recordings guide the live
activity. The seven engineering phases are not the same numbering system as
the twelve R/L lesson families. Setup is ordered by runtime dependencies.

| Family | Recorded focus | Learner documentation |
| --- | --- | --- |
| R1 / L1 | Two prediction questions; finished-product preview; ML in plain English; course route; environment and pytest | [Project overview](../README.md), [architecture](PROJECT_ARCHITECTURE.md), [setup](SETUP.md) |
| R2 / L2 | Features/targets, regression, chronological splits, metrics, overfitting; five-trip arithmetic | [Fare metrics](L2_FARE_METRICS_LAB.md), [data scope](DATA_STATEMENT.md) |
| R3 / L3 | Repository and environment; official TLC source, row meanings, notebook exploration | [Setup](SETUP.md), [source and notebook activity](L3_DATA_EXPLORATION_LAB.md) |
| R4 / L4 | Cleaning, aggregation, features, time splits, persisted evidence | [Pipeline activity](L4_PIPELINE_EVIDENCE_LAB.md), [lineage diagram](data-lineage-route.png) |
| R5 / L5 | Candidate training, model families, MLflow runs, validation-only selection; no-registration comparison | [MLflow activity](L5_MLFLOW_MODEL_TRACKING_LAB.md) |
| R6 / L6 | Tuning, SHAP, model cards; global fare explanation with limits | [Explainability activity](L6_EXPLAINABILITY_LAB.md), [generate the fare card](model_cards/README.md) |
| R7 / L7 | API requests, recursive batch, Docker packaging; valid/invalid requests and local fallback | [API examples](API_EXAMPLES.md), [API/Docker activity](L7_API_DOCKER_LAB.md), [setup](SETUP.md) |
| R8 / L8 | Controlled lint failure and repair; local CI; read-only CodeBuild evidence | [CI activity](L8_CI_QUALITY_GATE_LAB.md) |
| R9 / L9 | Request versus release path; Terraform; SageMaker Processing and gated pipeline | [AWS activity](L9_AWS_INFRASTRUCTURE_LAB.md), [infrastructure runbook](../infra/terraform/README.md) |
| R10 / L10 | Synthetic feature drift; PSI/KS; capture, alarms, schedule, bounded response | [Drift activity](L10_DRIFT_MONITORING_LAB.md) |
| R11 / L11 | Operational subgroup errors, versioned S3 evidence, provenance gaps, accountable mitigation | [Fairness activity](L11_FAIRNESS_REVIEW_LAB.md), [data statement](DATA_STATEMENT.md), [generate model cards](model_cards/README.md) |
| R12 / L12 | Dashboard source tracing, mixed freshness, stakeholder action, lifecycle close | [Dashboard activity](L12_DASHBOARD_INSIGHT_LAB.md), [business report](BUSINESS_REPORT.md), [cost guide](COST_OPTIMIZATION.md) |

## Recorded Activity and Current-Project Clarification

| Recording anchor | How to follow it today |
| --- | --- |
| R1 introduces a trip's fare and potential business benefits | The implemented target is `fare_amount`, not the complete `total_amount` bill. Pricing, fraud, and wait-time benefits are use cases, not implemented decision policies or measured impact. |
| R1-2 previews a six-hour demand forecast and a 17-mile JFK-to-Times-Square fare | Use the recorded showcase example in [API examples](API_EXAMPLES.md). Other examples there use different valid inputs and should not produce the same number. |
| R2 separates train, validation, and final test roles | R5/L5 and current code log test metrics for every fitted candidate, but choose using validation only. Do not use test results for tuning or change a winner after reading them. |
| L3/L4 rerun cached raw inputs | Download first when the raw files are absent. Keep the recorded cached rerun separate from initial setup. Notebook calculations explain stages; the pipeline writes official artifacts. |
| R5/L5 train fare with `--n-trials 1 --no-register` | Keep that command unchanged for the comparison. Registration for L6/L7 is separate preparation. Existing runs or database files do not prove registered models exist. |
| R5-3 starts `mlflow server`; L5 uses `mlflow ui` | These are distinct recorded launch commands for inspecting the local store. Follow the matching activity and the same SQLite backend; do not start both on port 5000. Neither command trains or registers models. |
| L6 loads the configured `latest` reference and shows version `n/a` | Record the reference and retain the unresolved numeric-version gap. Do not treat `latest` as an immutable version or the global beeswarm as a named-trip explanation. |
| R7/L7 package already-trained models | Prepare the portable registry before building the current Dockerfile. This additional setup does not change the recorded API activity. Docker remains optional; L8-L12 do not require learner Docker. |
| R7-2 runs a three-hour recursive batch forecast | Use `--horizon 3` to match that demo; setup's `24` is the broader example. SQLite rows are appended, and the writer's `latest` label is not resolved numeric model lineage. |
| R8/L8 distinguish local checks from hosted release | Terraform checks run locally, not in the current buildspec. CodeBuild success alone does not prove ECS stability. Manual builds without webhook events deploy by default, so use local CI for test-only work. |
| R8 repairs the scratch error but retains a red wider lint gate | The three remaining Ruff findings and passing pytest result belong to the recorded snapshot. Report today's gate independently; passing tests cannot erase a required lint failure. |
| R9/L9 show HTTP ALB/ECS and separate control paths | HTTP port 80 is the implemented route. Account-specific managed MLflow servers are out-of-band; default retraining and serving use registry/artifact bundles. Read JSON readiness separately from target health. |
| R9-2/R10-2 show managed jobs and release gates | Inspect help offline. Creating/upserting the SDK pipeline requires configured AWS/S3 inputs. A succeeded condition step does not mean every release condition was true; the weekly route disables deployment. |
| R10/L10 generate drift and inspect stored JSON | Preserve the command, seed, execution time, and input context separately; mode and timestamp are not embedded in the report. PSI drives flags; KS is supporting context. Drift is not accuracy loss or an automatic retraining trigger. |
| R11/L11 read operational slices and incomplete provenance | The 8.12 fare borough ratio is `Unknown` versus Manhattan, not aggregated outside-Manhattan error. Single-group ratios are coverage gaps. The current report/card does not establish immutable model identity. |
| R12/L12 use an AWS route with a labeled local fallback | Keep the API URL set by AWS preparation; set localhost only for the fallback. API response time, artifact generation, S3 metadata, and download time are different evidence clocks. |

## Snapshot Rules

Recorded counts, metric values, AWS status, resource names, timestamps, and URLs
belong to the captured run. They are not acceptance constants for every rerun.
For example, L10's captured synthetic run flags two of twenty demand features
(`lag_2`, `lag_3`) and two of fourteen fare features (`PULocationID`,
`DOLocationID`), using 1,500 current rows per problem. Reconcile your own JSON
before adopting those numbers; do not present synthetic results as production drift.

Keep the primary and fallback routes labeled as recorded. L11's primary route
publishes evidence to S3; L12's primary route uses the staging API. A learner
without AWS access can use the documented local fallback but must not claim
the cloud step was demonstrated. No presentation files are required in the
student export to follow these instructions.

## Comparison Scope

The documentation comparison used the available recorded activity references, lab
guides, and presentation text. It did not replay the final
audio/video or transcribe raster-only screenshots. Some earlier decks have no
embedded note text and some concept scripts contain prompts rather than full
spoken narration; those lessons were compared using the available slide content
and companion scripts. Exact spoken-word agreement cannot be established from
those files alone. Finalized materials remain unchanged.