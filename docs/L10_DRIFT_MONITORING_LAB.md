# L10: Inject and Interpret Synthetic Drift

- **Duration:** 70 to 85 minutes (no fixed cap; depth over speed)
- **Cloud required:** Optional instructor demonstration; students need no credentials
- **Docker required on the learner machine:** **No**
- **Outcome:** A synthetic drift finding, a clear map of the monitoring-and-retraining AWS
	resources, read-only CloudWatch / S3 / SageMaker evidence, and a guided console tour of the pages
	where drift detection appears — code and committed evidence first, with the console read as a live
	confirming view.

> **Docker boundary:** the required synthetic drift exercise is a Python command
> and runs without Docker. References to a monitoring image, ECR, container
> arguments, or SageMaker Processing describe optional remote AWS execution.
> Learners do not build or run that image locally. If AWS is unavailable, retain
> the local drift report and label the AWS inspection as skipped.

## How This Lab Teaches Monitoring (Read First)

This lab uses the same code-and-evidence-first approach as L9, and **also** tours the live console so
you can see the drift pages directly. Read each page correctly, because the console can mislead for
this stack:

- The SageMaker **Dashboard** counts only *active/running* resources, so a finished drift Processing
	job shows as zero there — open **Processing jobs**, not the Dashboard.
- The **Pipelines UI** needs a Studio domain (there are none). Use the CLI and the **Processing
	jobs** page instead.
- Drift work is proven by three durable sources that the console tour confirms: the **code**
	(`src/monitoring/`, `scripts/monitoring/`), the **Terraform** that schedules it
  (`infra/terraform/monitoring.tf`), and generated reports retained with execution
   context. Generated artifacts are not guaranteed to be in a fresh checkout.
   Keep account IDs, ARNs, and usernames hidden while touring.

### Monitoring & retraining resource map

| Resource | AWS service | Declared in | Role |
| --- | --- | --- | --- |
| Monitoring bucket | S3 | [`monitoring.tf`](../infra/terraform/monitoring.tf) | Captured inference inputs + drift reports. |
| Capture stream | Kinesis Firehose | [`monitoring.tf`](../infra/terraform/monitoring.tf) | Ships prediction-capture logs to S3 under `captured/`. |
| Drift + 5xx alarms, dashboard | CloudWatch | [`monitoring.tf`](../infra/terraform/monitoring.tf) | Alarm on `FeaturesDrifted`; operational dashboard. |
| Alerts topic | SNS | [`monitoring.tf`](../infra/terraform/monitoring.tf) | Notifies on drift / API 5xx / retraining. |
| SageMaker execution role | IAM | [`monitoring.tf`](../infra/terraform/monitoring.tf) | Assumed by the drift + retraining jobs. |
| Retraining schedule | EventBridge Scheduler | [`monitoring.tf`](../infra/terraform/monitoring.tf) | Weekly pipeline start, `DeployAfterQualityGate=false`. |
| **Drift Processing-job schedule** | EventBridge Scheduler + SageMaker Processing | [`monitoring.tf`](../infra/terraform/monitoring.tf) | Optional weekly drift job (gated on `monitoring_image_uri`). |
| Drift entrypoint | (runs on SageMaker) | [`scripts/monitoring/run_drift_job.py`](../scripts/monitoring/run_drift_job.py) | Computes PSI/KS, writes `drift_metrics.json`, pushes CloudWatch metrics. |
| Retraining pipeline | SageMaker Pipeline | [`retraining/pipeline.py`](../retraining/pipeline.py) | `Train → Evaluate → QualityGate → BuildAndDeploy`. |

## Monitoring Architecture Diagram

How a live prediction becomes drift evidence, an alarm, and — only with human
authorization — a released model.

```text
  ┌──────────┐  POST /predict/fare
  │  Client  │─────────────┐
  └──────────┘             ▼
                 ┌────────────────────┐
                 │ ALB ─▶ ECS Fargate │   response returns immediately;
                 │   FastAPI + model  │   capture is asynchronous and
                 └─────────┬──────────┘   NEVER blocks the caller
                           │ structured capture line -> stdout
                           ▼
                 ┌────────────────────┐
                 │ CloudWatch Logs    │
                 │  /ecs/<api> stream │
                 └─────────┬──────────┘
                           │ subscription filter (IAM: logs_to_firehose)
                           ▼
                 ┌────────────────────┐
                 │ Kinesis Firehose   │  buffers, retries, batches
                 │  "capture" stream  │
                 └─────────┬──────────┘
                           ▼
             ┌─────────────────────────────┐
             │ S3  monitoring bucket       │
             │   captured/      ◀── inputs │  lifecycle-expired after
             │   drift-reports/ ◀── output │  capture_retention_days
             └───────┬─────────────▲───────┘
      reads current  │             │ writes drift_metrics.json
                     ▼             │
   ┌─────────────────────────────────────────┐
   │ SageMaker Processing job (drift)        │◀── EventBridge Scheduler
   │  scripts/monitoring/run_drift_job.py    │    "drift" (weekly, gated on
   │  1 load reference   4 write report      │     monitoring_image_uri)
   │  2 load current(S3) 5 push CloudWatch   │
   │  3 PSI + KS         6 exit code         │
   └────────────────────┬────────────────────┘
                        │ PutMetricData: FeaturesDrifted
                        ▼
   ┌─────────────────────────────┐    ┌──────────────────────────┐
   │ CloudWatch                  │───▶│ SNS "alerts" topic       │
   │  · drift alarm              │    │  └─ email subscription   │
   │  · API 5xx alarm            │    └──────────────────────────┘
   │  · monitoring dashboard     │
   └─────────────────────────────┘
                        │
                        │  ⚠ NO automatic arrow to retraining
                        ▼
              ┌───────────────────────┐
              │  HUMAN INVESTIGATION  │  freshness? schema? seasonality?
              │  (decision, not code) │  segment mix? real degradation?
              └───────────┬───────────┘
                          ▼
   ┌──────────────────────────────────────────────────────┐
   │ SageMaker Pipeline  taxi-forecasting-retrain         │◀── EventBridge
   │   TrainModels ─▶ EvaluateModels ─▶ QualityGate       │    "retrain"
   │                                        │             │    (weekly)
   │                     ┌──────────────────┴──────────┐  │
   │                     │ 5 conditions must ALL pass: │  │
   │                     │  1 demand RMSE ≤ 50         │  │
   │                     │  2 fare   RMSE ≤ 5          │  │
   │                     │  3 demand vs baseline ≤ 20% │  │
   │                     │  4 fare   vs baseline ≤ 20% │  │
   │                     │  5 DeployAfterQualityGate   │  │
   │                     │    requires "true"          │  │
   │                     │    schedule sends "false"   │  │
   │                     │    so deployment is BLOCKED │  │
   │                     └──────────────────┬──────────┘  │
   │                              BuildAndDeploy ✗ never  │
   │                              eligible on schedule    │
   └──────────────────────────────────────────────────────┘
```

### The four-stage escalation (each stage needs its own evidence)

```text
  STAGE 1          STAGE 2             STAGE 3            STAGE 4
  DETECT      ──▶  INVESTIGATE    ──▶  RETRAIN       ──▶  RELEASE
  ────────         ───────────         ───────────        ─────────
  PSI crossed      is it real?         train candidate    deploy it
  its threshold;   data bug?           evaluate vs        to ECS
  KS adds context  seasonality?        baseline
                   segment mix?
  ────────         ───────────         ───────────        ─────────
  evidence:        evidence:           evidence:          evidence:
  distribution     data quality +      RMSE vs gate       explicit human
  statistics       domain context      + vs baseline      authorization
  ────────         ───────────         ───────────        ─────────
  AUTOMATED        HUMAN               AUTOMATED          HUMAN
  (weekly job)     (this lab)          (pipeline)         (gate input)
```

> The single most important teaching point in this lab: **stage 1 does not imply stage 3.**
> A distribution alarm is evidence that inputs moved, not evidence that the model got worse.
> There is no alert-to-pipeline trigger in this implementation. Separately,
> `DeployAfterQualityGate=false` prevents a scheduled retraining run from releasing.

## Monitoring Component Catalog

| Component | What it is | Why it is used here | Without it |
| --- | --- | --- | --- |
| **CloudWatch Logs + subscription filter** (`monitoring.tf`) | Log store with a streaming tap | The API already writes structured capture lines to stdout; the filter taps that stream without changing app code | You would add an S3 client to the request path and slow down predictions |
| **Kinesis Firehose** (`monitoring.tf`) | Managed streaming delivery | Buffers, batches, retries, and writes to S3 — a delivery problem we do not want to hand-roll | Lost capture data whenever S3 or the network hiccups |
| **S3 monitoring bucket** (`monitoring.tf`) | Object storage, two prefixes | `captured/` is the drift job's *current* window; `drift-reports/` is its durable output; a lifecycle rule expires old capture | Nothing to compare against, and reports would vanish |
| **SageMaker Processing job** (`run_drift_job.py`) | Ephemeral managed compute | Uses the same `compute_drift` function and report schema as the local generator, on managed compute, paying only per run | You would keep a box running to do a weekly job |
| **EventBridge Scheduler "drift"** (`monitoring.tf`) | Managed cron | Runs detection weekly; gated on `monitoring_image_uri` so it exists only when you choose to pay | Detection depends on someone remembering |
| **CloudWatch custom metric** `FeaturesDrifted` | Numeric time series | The job publishes a count; a metric (not a log line) is what an alarm can watch | You could not alarm on drift at all |
| **CloudWatch drift alarm** (`monitoring.tf`) | Threshold monitor | Converts the metric into a state change humans can be notified about | Drift results would sit unread in S3 |
| **CloudWatch API 5xx alarm** (`monitoring.tf`) | Threshold monitor | Watches *service* failure, which is a different question from *model* drift | You would conflate "API is broken" with "inputs moved" |
| **CloudWatch dashboard** (`monitoring.tf`) | Metric view | Puts service health and drift on one operational page | Correlating an incident means opening several consoles |
| **SNS topic + email** (`monitoring.tf`) | Pub/sub notification | Fans alarm state out to a human owner | Alarms would fire silently |
| **SageMaker execution role** (`monitoring.tf`) | Shared job identity | Supports drift and retraining; inspect the actual S3, metric, and release-related permissions rather than assuming drift-only access | Jobs could not reach their inputs or outputs |
| **EventBridge Scheduler "retrain"** (`monitoring.tf`) | Managed cron | Starts the pipeline weekly with `DeployAfterQualityGate=false` | Retraining would be ad hoc, and the no-deploy guarantee would be a convention rather than config |
| **SageMaker Pipeline** (`retraining/pipeline.py`) | DAG orchestrator | Records the train/evaluate/gate decision chain as a durable artifact | A gate decision would be a line in someone's terminal history |

> Cost note: the drift path is almost entirely **per-run and per-byte** — a weekly Processing job,
> Firehose throughput, and stored objects that a lifecycle rule expires. There is no always-on
> monitoring server.

## Task 1: Predict the Alert

Record which problem and features you expect the synthetic drift mode to flag.

## Task 2: Generate Drift Evidence

State your Task 1 expectation first, then generate the synthetic reports:

Prerequisite: complete the feature pipeline so both training splits exist. The
generator samples those splits; it does not invent an independent dataset and
does not require registered models. Record command, seed (default `42`), mode,
input paths, and execution time alongside the output.

```powershell
.\.venv\Scripts\python.exe -m scripts.monitoring.demo_drift_report --problem all --mode drift
```

`--mode drift` deliberately perturbs the current sample to provoke PSI drift;
inspect the actual flags rather than assuming a result. `--problem all` produces
both a demand and a fare report. Record the generated file paths
under `artifacts/drift/`, and say the boundary aloud: the **pipeline is real, the drift is
manufactured** for teaching — these are synthetic inputs, not captured production traffic.

## Task 3: Inspect Both Reports

Open the demand and fare drift JSON files. For each problem, identify:

- the **problem** and **n_current** fields,
- the per-feature **PSI/KS** statistic or score,
- the configured **threshold** (PSI `0.2`, KS alpha `0.05`),
- the per-feature **alert state**, and
- the **generation context**, retained separately from the JSON.

The JSON does not embed mode, seed, generation timestamp, or a reference-period
description. Use the retained command/time and inspect the source training split
to establish those facts. A file modification time is not the reference period.
The current sample is a sampled training distribution with optional deliberate
shifts, not captured production requests.

Confirm the flagged feature matches — or corrects — the prediction you wrote in Task 1.

In this implementation, **PSI alone determines the `drifted` flag**: a feature is flagged when PSI
is at least `0.2`. The KS statistic, p-value, and `ks_alpha` are reported as supporting context and do
not independently trigger the flag.

## Task 4: Decide the Response

For one flagged feature, record:

```text
Evidence:
Operational risk:
Immediate investigation:
Retraining decision and reason:
Limitation:
```

A drift alert is a reason to investigate. It is not automatic proof of model
performance degradation and does not by itself justify promotion of a new model.

Now classify each statement as **signal**, **evidence**, **gate**, or
**authorization**:

| Statement | Classification |
| --- | --- |
| PSI reaches its configured threshold; KS supplies supporting context |  |
| Delayed actuals show candidate error on a held-out split |  |
| Candidate RMSE and baseline-relative checks pass |  |
| `DeployAfterQualityGate=true` is supplied for an approved run |  |

The intended chain is `detect -> investigate -> decide whether to retrain ->
evaluate candidate -> authorize release`. These are separate decisions. In this
repository, drift detection does not automatically start the SageMaker Pipeline.
An optional weekly drift-detection **SageMaker Processing job** can be scheduled
(set `monitoring_image_uri`; see the `drift_schedule_name` Terraform output); it
computes drift and publishes metrics, but still does not start retraining.

## Task 5: The Drift Detection Job on SageMaker

Task 2 ran drift **locally** through `scripts/monitoring/demo_drift_report.py`. In the cloud,
[`scripts/monitoring/run_drift_job.py`](../scripts/monitoring/run_drift_job.py) runs as a **SageMaker
Processing job**. These are different entrypoints that share `src.monitoring.compute_drift` and the
same report schema. Trace what the cloud job does, top to bottom, from its module docstring:

1. Load the **reference** distribution — the training-split features per problem.
2. Load the **current** captured inputs — Firehose-delivered prediction-capture records from
	`s3://<monitoring_bucket>/captured/`.
3. Compute per-feature **PSI + KS** drift via `src.monitoring.compute_drift`.
4. Write `drift_metrics.json` (and optional Evidently HTML) to the output dir; SageMaker uploads it
	to `s3://<monitoring_bucket>/drift-reports/`.
5. Push summary metrics to **CloudWatch** (`TaxiForecasting/Drift` namespace), which feeds the
	`FeaturesDrifted` alarm.
6. Exit non-zero when drift is detected. This happens after metrics are published, so SageMaker marks
    the job `Failed` even when detection completed as designed. Use the report, metrics, and logs to
    distinguish detected drift from an execution or infrastructure error; the alarm reacts to the
    published metric, not directly to the process exit code.

### 5a. How it is scheduled (Terraform)

The weekly job is declared as an EventBridge Scheduler target that calls the SageMaker
`createProcessingJob` API. Read the `aws_scheduler_schedule "drift"` block in
[`monitoring.tf`](../infra/terraform/monitoring.tf) and note:

- It is **gated**: created only when `monitoring_image_uri` is set (local `drift_enabled`). Absent an
	image, the schedule does not exist — an intentional, cost-safe default.
- It **reuses** the existing `sagemaker` execution role and the scheduler role's
	`sagemaker:CreateProcessingJob` + `iam:PassRole` permissions — no new IAM.
- Its `ContainerArguments` pass `--problem`, `--current-s3 …/captured/`, and the CloudWatch
	`--namespace`, exactly mirroring the local run.

### 5b. Produce a job on demand (optional, instructor)

To make a fresh Processing job visible in the console before a walkthrough, an authorized instructor
can submit one with the CLI wrapper (mirrors the training wrapper
[`scripts/aws/sagemaker_train.py`](../scripts/aws/sagemaker_train.py)):

```powershell
python -m scripts.aws.submit_drift_job `
	--role-arn (terraform -chdir=infra/terraform output -raw sagemaker_role_arn) `
  --image-uri "<ecr>/taxi-monitoring:latest" `
	--problem fare `
	--current-s3 "s3://<monitoring_bucket>/captured/" `
	--output-s3-uri "s3://<monitoring_bucket>/drift-reports/"
```

It then appears under **SageMaker → Data preparation → Processing jobs** (the one console page that
needs no Studio domain). Do not run this during a graded student lab; it incurs cost.

### 5c. Where the drift job shows up (and where it does not)

| Console section | Drift job appears? | Note |
| --- | --- | --- |
| **Processing jobs** | Yes, after it runs | Name begins `taxi-forecasting-drift-…`. |
| **Dashboard** | No | Counts active only; a finished job is historical. |
| **Pipelines** | No | The drift job is a standalone Processing job, not a Pipeline step. |
| **CloudWatch → Alarms** | Indirectly | Publishes `FeaturesDrifted`, which drives the drift alarm. |
| **S3 `drift-reports/`** | Yes | `drift_metrics.json` output object. |

## Task 6: Inspect Live Monitoring Evidence

The instructor runs this read-only extension with the authorized staging profile. Do not publish
metrics, change alarm state, start retraining, or expose account IDs and ARNs.

```powershell
$Region = "us-east-1"
$Bucket = terraform -chdir=infra/terraform output -raw monitoring_bucket

aws cloudwatch describe-alarms --region $Region `
	--alarm-name-prefix taxi-forecasting-staging `
	--query "MetricAlarms[].{Alarm:AlarmName,State:StateValue,Updated:StateUpdatedTimestamp}"
aws s3api list-objects-v2 --region $Region --bucket $Bucket --prefix captured/ `
	--query "reverse(sort_by(Contents,&LastModified))[:5].{Modified:LastModified,Size:Size,Key:Key}"
aws scheduler list-schedules --region $Region `
	--name-prefix taxi-forecasting-staging --query "Schedules[].{Name:Name,State:State}"
aws scheduler get-schedule --region $Region `
	--name taxi-forecasting-staging-retrain `
	--query "{State:State,Expression:ScheduleExpression,TargetInput:Target.Input}"
```

Also check for the drift job's own outputs and any drift Processing jobs:

```powershell
aws s3api list-objects-v2 --region $Region --bucket $Bucket --prefix drift-reports/ `
	--query "reverse(sort_by(Contents,&LastModified))[:5].{Modified:LastModified,Size:Size,Key:Key}"
aws sagemaker list-processing-jobs --region $Region --sort-by CreationTime --sort-order Descending `
	--max-results 5 --query "ProcessingJobSummaries[?starts_with(ProcessingJobName,'taxi-forecasting-drift')].{Name:ProcessingJobName,Status:ProcessingJobStatus}"
```

Example output (sanitized) with `monitoring_image_uri` **unset** (the cost-safe default):

```json
// scheduler list-schedules -> only the retraining schedule exists; no drift schedule
[ { "Name": "taxi-forecasting-staging-retrain", "State": "ENABLED" } ]

// drift-specific Processing jobs -> none, because no drift image is configured
[]
```

If `monitoring_image_uri` is unset, the drift schedule and any `taxi-forecasting-drift-*` Processing
jobs are legitimately **absent** — record that as the current boundary, not a failure.

Compare the live alarm and capture evidence with the synthetic JSON reports. CloudWatch state is a
current operational signal; an S3 object proves delivery to that key; neither establishes model
accuracy loss. The weekly schedule confirms that training and evaluation are automated. Its
`DeployAfterQualityGate=false` input confirms that scheduled runs cannot enter the pipeline's
`BuildAndDeploy` branch.

## Completion Check

- The expected alert was recorded before generation.
- Both demand and fare reports were inspected.
- A finding cites a feature score and its threshold.
- Investigation is separated from retraining and deployment decisions.
- Signal, performance evidence, model-quality gates, and release authorization were distinguished.
- The monitoring/retraining resource map was reviewed and the drift Processing job traced from code to schedule.
- The difference between the local drift run and the SageMaker Processing-job run is explained.
- The reasons the drift job may be **absent** (no `monitoring_image_uri`) or **invisible on the dashboard** (active-only) are stated.
- Synthetic evidence is not described as live production monitoring.
- Live CloudWatch/S3/SageMaker evidence was inspected, or the optional cloud step was explicitly skipped.
- The console drift pages (alarms, dashboard, S3, Processing jobs, schedule) were toured with private identifiers hidden, or the optional console tour was explicitly skipped.
