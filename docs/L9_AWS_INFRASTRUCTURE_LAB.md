# L9: Trace the AWS Request Path

- **Duration:** 75 to 90 minutes (no fixed cap; depth over speed)
- **AWS credentials required:** Optional instructor demonstration; students need no credentials
- **Terraform required:** Recommended; a static inspection route is provided
- **Docker required on the learner machine:** **No**
- **Outcome:** A validated ALB-to-ECS route, a complete inventory of every AWS resource this project
	declares, read-only evidence from the deployed runtime and SageMaker pipeline, and a guided
	AWS Management Console tour — code and committed evidence first, with the console read as a live
	confirming view.

> **Docker boundary:** this is an infrastructure-tracing lab, not a local image
> build. `docker build` and `docker push` appear in diagrams and `buildspec.yml`
> because AWS CodeBuild performs them in the release path. Learners inspect that
> path through Terraform, source files, committed evidence, and optional
> read-only AWS views. Do not run Docker commands locally for L9. If Terraform
> is unavailable, use the lab's static inspection route.

## How This Lab Teaches Cloud (Read First)

This lab leads with code and committed evidence, and **also** tours the AWS Management Console so you
can see each resource in its live view. The order matters: code is the source of truth, and the
console confirms it. Reading the console *correctly* is part of the skill, because it mirrors
professional practice:

- **Consoles are ephemeral and account-specific.** A screen recording of a console is not
	reproducible, expires, and leaks account context. Infrastructure defined in code does not — so we
	trace every console page back to the `.tf` file that declares it, and keep account IDs, ARNs, and
	usernames off screen.
- **The console can look "empty" even when everything is deployed.** The SageMaker *Dashboard* only
	counts resources that are currently *active/running*. Completed processing jobs and finished
	pipeline executions are historical, so the dashboard shows zeros. This is expected — read the
	Processing jobs page, not the Dashboard, for completed work.
- **Some console pages need a SageMaker Studio domain.** The Pipelines UI lives inside Studio; with
	zero domains it is unreachable. You do **not** need to create a domain to prove the pipeline
	exists — the CLI and the Processing jobs page show the same facts.

The three durable evidence sources you will use throughout, which the console tour confirms:

1. **Terraform** (`infra/terraform/*.tf`) — infrastructure declarations; the SageMaker
	pipeline is defined separately in Python, and out-of-band resources are not managed here.
2. **Read-only AWS CLI queries** — live state rendered as text (reproducible on camera).
3. **Dated project evidence and locally retained results** — historical observations
	must be distinguished from fresh query output. The student repository does not
	include instructor presentation materials or generated verification artifacts.

> Framing to say aloud: *"Production ML is defined in code, executed by pipelines, and proven by
> committed evidence plus read-only queries. The console is a live confirming view we read against
> the code — not the thing we depend on."*

## System Architecture Diagram

The whole platform on one page. Solid arrows are the **live request path**; everything else is
control, delivery, storage, or operations.

![End-to-end AWS MLOps architecture for taxi fare prediction](aws%20arch.png)

### The four planes at a glance

![AWS serving, delivery, data, and ML-ops planes](aws%20table.png)

These existing illustrations are orientation views, not an inventory of current
AWS state. Use [PROJECT_ARCHITECTURE.md](PROJECT_ARCHITECTURE.md) and source code
for the implemented path. Any managed MLflow server shown is an out-of-band
resource, not the registry backend used by the default local, retraining, or
self-contained serving workflow.

> Say aloud: *"Only the serving plane answers a user. If you cannot name the plane a resource
> belongs to, you cannot reason about what breaks when it fails."*

### Request path vs. release path (do not confuse these)

```text
REQUEST  (milliseconds, per user call, synchronous)
  Client ─▶ ALB ─▶ ECS task ─▶ FastAPI route ─▶ model.predict() ─▶ JSON ─▶ back out

RELEASE  (minutes, per git push, asynchronous)
  git push main ─▶ CodeBuild webhook ─▶ pytest+ruff ─▶ docker build
                ─▶ docker push ECR ─▶ ecs update-service ─▶ rolling replace
                ─▶ ALB health check passes ─▶ old task drains
```

The **only** two touchpoints between them are **ECR** (the image the task pulls) and the **ECS
update API** (which asks for a new deployment). CodeBuild never sits in front of a prediction.

## AWS Component Catalog (What Each Service Does and Why It Is Here)

For each component: what it is, why this project uses it, and what we would lose without it.

### Serving plane

| Component | What it is | Why it is used here | Without it |
| --- | --- | --- | --- |
| **ECR** (`main.tf`) | Private Docker registry | Stores the self-contained API image (model + MLflow registry baked in) so a task start needs no external download | Tasks would pull from a public registry — slower, unversioned, and an availability/supply-chain risk |
| **ECS Fargate** (`main.tf`) | Serverless container runtime | Runs FastAPI with no EC2 hosts to patch or scale; `desired_count` controls capacity | You would manage EC2 instances, AMIs, and autoscaling yourself |
| **ALB + target group + listener** (`main.tf`) | Layer-7 load balancer | Single public DNS entry point; routes only to targets passing `GET /health`; enables zero-downtime rolling deploys | Clients would hit a task IP directly — no health gating, no rolling deploy, no stable address |
| **Security groups** (`main.tf`) | Stateful virtual firewalls | ALB SG allows public :80; service SG accepts traffic **only** from the ALB SG | The container port would be reachable from the internet directly |
| **CloudWatch Logs** (`main.tf`) | Managed log store | Captures container stdout/stderr; also the *source* the capture subscription filter reads | No post-hoc debugging and no prediction capture for drift |
| **IAM task + execution roles** (`main.tf`) | Scoped identities | Execution role lets the ECS agent pull from ECR and write logs; task role is what app code assumes | Either nothing starts, or the container runs over-privileged |
| **ECR lifecycle policy** (`main.tf`) | Image retention rule | Expires untagged/old images so the registry does not grow without bound | Storage cost creeps and the registry fills with unusable layers |
| **Streamlit dashboard stack** (`dashboard.tf`, opt-in) | A *second* ECS+ALB+ECR stack | Serves the showcase dashboard; stateless, calls the API over its ALB DNS. Gated by `enable_dashboard` (default false) | No business-facing UI — and you would lose the proof that the serving pattern generalizes |

### Delivery plane

| Component | What it is | Why it is used here | Without it |
| --- | --- | --- | --- |
| **CodeBuild project + webhook** (`cicd.tf`) | Managed build service | On push to the deploy branch: lint, test, build image, push to ECR, request an ECS rollout | Releases become manual laptop builds — unreproducible and unauditable |
| **S3 cicd-artifacts** (`cicd.tf`) | Object storage | Supplies `mlruns/`, `mlflow.db`, and demand parquet that get baked into the image at build time | The build could not assemble a self-contained image |
| **CodeBuild IAM role + log group** (`cicd.tf`) | Identity + logs | Grants ECR, ECS, S3 artifact-read, and logging permissions; inspect action and resource scope in the policy | No traceability of who deployed what |
| **CodeBuild webhook + GitHub source credential** (`cicd.tf`) | Trigger + stored token | Starts a build on push to the deploy branch without polling; the credential is stored in AWS, never in the repo | Builds would be manual, or a token would end up in source control |

### Data plane

| Component | What it is | Why it is used here | Without it |
| --- | --- | --- | --- |
| **S3 datalake** (`datalake.tf`) | Object storage | Durable cloud home for trip/feature data, versioned and access-blocked | Data would live only on laptops |
| **Glue Data Catalog + Athena workgroup** (`datalake.tf`) | Schema registry + serverless SQL | Optional: query lake files with SQL, results to a dedicated bucket | You would need a cluster or a warehouse to ask a question |
| **S3 monitoring** (`monitoring.tf`) | Object storage | Two prefixes: `captured/` (inference inputs) and `drift-reports/` (job output); lifecycle-expired | No inputs to compare, and drift results would be ephemeral |
| **Kinesis Firehose + log subscription filter** (`monitoring.tf`) | Streaming delivery | Ships prediction-capture log lines from CloudWatch Logs into S3 `captured/` with buffering/retry | You would hand-roll a log exporter |

### ML-ops plane

| Component | What it is | Why it is used here | Without it |
| --- | --- | --- | --- |
| **SageMaker Pipeline** (`retraining/pipeline.py`) | DAG orchestrator | Encodes Train → Evaluate → QualityGate → (conditional) BuildAndDeploy with dependencies and recorded decisions | Retraining becomes an unrepeatable script with no gate record |
| **SageMaker Processing jobs** | Ephemeral managed compute | Each step runs on temporary compute that disappears when finished — you pay only per run | You would keep a training box running to do occasional work |
| **EventBridge Scheduler x2** (`monitoring.tf`) | Managed cron | `retrain` (weekly, `DeployAfterQualityGate=false`) and `drift` (weekly, gated on a monitoring image) | Detection and retraining would depend on someone remembering |
| **CloudWatch alarms** (`monitoring.tf`) | Threshold monitors | `FeaturesDrifted` drift alarm and API 5xx alarm, both wired to SNS | Failures and drift would be noticed by users first |
| **CloudWatch dashboard** (`monitoring.tf`) | Metric view | One operational page combining service and drift metrics | Metrics scattered across consoles |
| **SNS topic + email subscription** (`monitoring.tf`) | Pub/sub notification | Fans alarm state changes out to humans | Alarms would fire silently |
| **SageMaker execution role** (`monitoring.tf`) | Scoped identity | Assumed by drift and retraining jobs to read S3, write reports, publish metrics | Jobs could not access their own inputs/outputs |
| **MLflow registry and artifacts** | Experiment tracking | Local SQLite plus `mlruns/`; retraining writes a candidate bundle to S3 and releases bake it into the API image | No tracked model/run lineage |
| **Resource Group** (`resource_group.tf`) | Tag-based grouping | Collects every project resource under one console view | You would hunt resources service by service |

### Cross-cutting: identity, network, and guardrails

These never appear on an architecture diagram, but they decide whether the
platform is safe. Show them explicitly.

| Component | What it is | Why it is used here | Without it |
| --- | --- | --- | --- |
| **Six IAM roles** (`main.tf`, `cicd.tf`, `monitoring.tf`) | Scoped assumable identities | ECS task-execution, CodeBuild, Firehose, Logs-to-Firehose, SageMaker execution, EventBridge Scheduler — each granted only what its job needs | One over-privileged role turns any single compromise into a full-account incident |
| **Four security groups** (`main.tf`, `dashboard.tf`) | Stateful virtual firewalls | Each ALB group faces the internet; each service group accepts traffic **only** from its ALB group | Containers would be directly reachable from the internet |
| **S3 Block Public Access** (`cicd.tf`, `datalake.tf`, `monitoring.tf`) | Bucket guardrail | Restricts public ACLs and policies on provisioned buckets; it does not prevent every form of data exposure | Public-access misconfiguration is less constrained |
| **S3 versioning** (`datalake.tf`) | Object history | Protects lake data from overwrite/delete mistakes | An accidental overwrite is unrecoverable |
| **S3 lifecycle rule** (`monitoring.tf`) | Retention policy | Expires captured inference data after `capture_retention_days` | Capture grows forever — unbounded cost and a growing privacy surface |
| **CloudWatch log subscription filter** (`monitoring.tf`) | Streaming tap on a log group | The seam joining serving to monitoring: forwards capture lines to Firehose without touching the request path | You would have to write to S3 from inside the prediction handler |

> Say aloud: *"The diagram shows what talks to what. The roles and security groups show what is
> **allowed** to. You need both to reason about blast radius."*

> Cost-shape teaching point: baseline costs include ALBs, running Fargate tasks,
> storage, and configured monitoring. **Per-run** costs include CodeBuild minutes
> and SageMaker Processing jobs. Any out-of-band MLflow server adds separate cost.
> There is deliberately **no** always-on SageMaker endpoint — serving stays on Fargate.

## AWS Resource Inventory (Reference Table)

Use this as a map of the implemented resources and their declarations. Counts
and current state depend on flags, supplied inputs, and the selected account.
An out-of-band resource cannot be verified from Terraform alone.

| Resource | AWS service | Declared in | Purpose |
| --- | --- | --- | --- |
| API image repository | ECR | [`main.tf`](../infra/terraform/main.tf) | Stores the self-contained API image the ECS task pulls. |
| API service + tasks | ECS Fargate | [`main.tf`](../infra/terraform/main.tf) | Runs the FastAPI container; no servers to manage. |
| Load balancer + target group + listener | ALB (ELBv2) | [`main.tf`](../infra/terraform/main.tf) | Public entry point; routes to healthy tasks via `/health`. |
| Task/execution IAM roles | IAM | [`main.tf`](../infra/terraform/main.tf) | Least-privilege roles the task and ECS agent assume. |
| API log group | CloudWatch Logs | [`main.tf`](../infra/terraform/main.tf) | Container stdout/stderr; ALB access logging is not configured here. |
| CI/CD project + webhook | CodeBuild | [`cicd.tf`](../infra/terraform/cicd.tf) | Test → build → push to ECR → roll ECS on main. |
| Baked-artifact bucket | S3 | [`cicd.tf`](../infra/terraform/cicd.tf) | Holds `mlruns/`, `mlflow.db`, demand parquet for builds. |
| Data lake bucket (+ optional Athena/Glue) | S3 / Athena / Glue | [`datalake.tf`](../infra/terraform/datalake.tf) | Cloud data lake; optional SQL query layer. |
| Monitoring bucket | S3 | [`monitoring.tf`](../infra/terraform/monitoring.tf) | Captured inference data + drift reports. |
| Alerts topic | SNS | [`monitoring.tf`](../infra/terraform/monitoring.tf) | Drift / API 5xx / retraining notifications. |
| Capture delivery stream | Kinesis Firehose | [`monitoring.tf`](../infra/terraform/monitoring.tf) | Ships prediction-capture logs to S3. |
| Drift + API alarms, dashboard | CloudWatch | [`monitoring.tf`](../infra/terraform/monitoring.tf) | Alarms and an operational dashboard. |
| SageMaker execution role | IAM | [`monitoring.tf`](../infra/terraform/monitoring.tf) | Assumed by drift + retraining jobs. |
| Retraining schedule | EventBridge Scheduler | [`monitoring.tf`](../infra/terraform/monitoring.tf) | Weekly, `DeployAfterQualityGate=false`. |
| Drift Processing-job schedule | EventBridge Scheduler + SageMaker Processing | [`monitoring.tf`](../infra/terraform/monitoring.tf) | Optional weekly drift job (gated on `monitoring_image_uri`). |
| Dashboard ECR/ALB/ECS stack | ECR / ALB / ECS | [`dashboard.tf`](../infra/terraform/dashboard.tf) | Stateless Streamlit showcase (optional). |
| Retraining pipeline | SageMaker Pipeline | [`retraining/pipeline.py`](../retraining/pipeline.py) | `Train → Evaluate → QualityGate → (conditional) BuildAndDeploy`. |
| Optional managed experiment tracking | SageMaker MLflow tracking server | Out-of-band, not declared here | Account-specific resource; do not assume it exists or backs this workflow. |

### Read the inventory as four planes

The table above is easier to reason about when grouped into four *planes*. Each plane answers a
different question, and every resource belongs to exactly one:

1. **Serving plane — "how does a prediction get answered?"**
	ECR (image store) -> ECS Fargate task (runtime) -> ALB listener + target group (routing) ->
	CloudWatch Logs (observability). This is the only plane on the live request hot path. The
	optional Streamlit dashboard stack in [`dashboard.tf`](../infra/terraform/dashboard.tf) is a
	second, independent serving stack of the same shape.
2. **Delivery plane — "how does new code reach the serving plane?"**
	CodeBuild project + webhook and the baked-artifact S3 bucket in
	[`cicd.tf`](../infra/terraform/cicd.tf). It tests, builds, pushes to ECR, and requests an ECS
	rollout. It touches the serving plane only through ECR and the ECS update API.
3. **Data plane — "where does the data live?"**
	The data-lake bucket (and optional Athena/Glue query layer) in
	[`datalake.tf`](../infra/terraform/datalake.tf), plus the monitoring bucket and Firehose capture
	stream in [`monitoring.tf`](../infra/terraform/monitoring.tf). Storage and query, not compute.
4. **ML-ops plane — "how are models trained, gated, and watched?"**
	The SageMaker retraining Pipeline ([`retraining/pipeline.py`](../retraining/pipeline.py)), the
	SageMaker execution role, the retraining and drift EventBridge schedules, the CloudWatch alarms +
	dashboard, the SNS alerts topic, and the MLflow registry/artifact bundles. This is where the
	"console looks empty" confusion lives, because most of its work is *historical jobs*, not
	always-on resources.

> Say aloud: *"Four planes — serve, deliver, store, operate. Every AWS resource here is one of
> those four, and only the serving plane is on the live request path."*

## Task 1: Predict the Request Path

Before opening Terraform, commit to a prediction on paper. Arrange these components in the expected
request order and write one sentence on what each does:

1. **Client** issues an HTTP request to the load balancer's public DNS name.
2. **Application Load Balancer (ALB)** accepts the connection and forwards it to a *healthy* target.
3. **ECS Fargate task** is the serverless container the ALB routes to (no EC2 host to manage).
4. **FastAPI application** inside the container matches the route and validates the request body.
5. **Loaded model** returns the prediction, which flows back out the same path.

The current Terraform listener is HTTP on port 80. HTTPS needs additional TLS
certificate/listener configuration; do not treat this demo route as encrypted.

Then predict two *control-plane* facts you will verify later: (a) how a new image reaches that task
(CI/CD), and (b) how the ALB decides a task is "healthy" (the `/health` check). Writing the guess
first turns the later evidence into a confirmation or a correction of a concrete claim, not a passive
read.

## Task 2: Validate Terraform Without AWS Credentials

Static validation proves the configuration is internally consistent — providers resolve, variables
type-check, references exist — **without** touching an AWS account. State the expected result aloud
("validation succeeds; nothing is created") before running each command:

```powershell
terraform -chdir=infra/terraform fmt -check -recursive   # formatting is canonical
terraform -chdir=infra/terraform init -backend=false     # load providers, skip remote state
terraform -chdir=infra/terraform validate                # syntax + references are valid
```

- `fmt -check` fails loudly if formatting drifted — a cheap, deterministic correctness signal.
- `init -backend=false` deliberately skips the remote (S3/DynamoDB) backend, so **no credentials are
	needed** and no state is read or written.
- `validate` confirms the resource graph is buildable; it does **not** confirm anything is deployed.

Expected result: `Success! The configuration is valid.` If Terraform is unavailable, inspect
[`main.tf`](../infra/terraform/main.tf) and [`README.md`](../infra/terraform/README.md) and record
that static route instead. Name the boundary out loud: **validation is not deployment.**

## Task 3: Trace the Runtime Resources

Open [`main.tf`](../infra/terraform/main.tf), locate each runtime resource, and state its single
responsibility:

- **ECR** — the private registry holding the immutable API image; the task pulls it by tag/digest.
- **ECS Fargate service + task definition** — the serverless runtime. Note the task's `image` and the
	container `port` it exposes.
- **ALB listener + target group** — the listener accepts public traffic; the target group tracks which
	tasks are healthy and receives the forwarded request.
- **`/health` health check** — the target group's HTTP probe. Inspect the JSON
	model-readiness fields separately: this endpoint can return HTTP 200 with a
	degraded body, so a healthy target alone does not prove model readiness.
- **CloudWatch Logs** — the task's stdout/stderr; ALB access logging is not
	enabled by this Terraform configuration.

Then draw the path and separate the **data plane** (the live request travelling
client -> ALB -> task -> model -> back) from the **control plane** (ECR, IAM roles, log groups) that
exists to support that request but is not itself on the hot path.

## Task 4: Connect CI/CD

Open [`buildspec.yml`](../buildspec.yml) and trace how a successful `main`-branch build (1) runs the
tests, (2) builds the image, (3) pushes it to ECR, and (4) requests an ECS service update. The
buildspec does **not** wait for service stability; the later read-only ECS and smoke evidence verifies
rollout separately.

The instructor can show recent build outcomes read-only (keep account IDs and secrets off screen):

```powershell
$Region = "us-east-1"
$Ids = (aws codebuild list-builds-for-project --project-name taxi-forecasting-staging-ci `
	--region $Region --sort-order DESCENDING --query "ids[0:3]" --output text) -split "\s+"
aws codebuild batch-get-builds --ids $Ids --region $Region `
	--query "builds[].{Status:buildStatus,Phase:currentPhase,Start:startTime}"
```

Example output (sanitized):

```json
[
  { "Status": "FAILED",    "Phase": "COMPLETED", "Start": "2026-09-03T22:02:25Z" },
  { "Status": "SUCCEEDED", "Phase": "COMPLETED", "Start": "2026-09-03T21:00:39Z" },
  { "Status": "SUCCEEDED", "Phase": "COMPLETED", "Start": "2026-09-03T16:20:03Z" }
]
```

A mix of `SUCCEEDED` and `FAILED` is normal history — the teaching point is that each build is an
auditable, timestamped record of the release path, not that every build passed. Do not run deployment
commands or display credentials, account identifiers, or live Terraform outputs.

## Video Timestam-  from 15:30 to 16:45
## Task 5: Inspect the Live Runtime and Retraining Path

The instructor runs this read-only extension with the authorized staging profile. Keep credentials,
account IDs, ARNs, and private state off screen.

### 5a. The two SageMaker execution paths

SageMaker is used in **two distinct ways** in this project. Students must not conflate them:

1. **Standalone managed training** — [`scripts/aws/sagemaker_train.py`](../scripts/aws/sagemaker_train.py)
	submits [`scripts/train_models.py`](../scripts/train_models.py) as a **single SageMaker Processing
	job** (a `ScriptProcessor`). This is the "run the same training on cloud compute instead of a
	laptop" path. It is a one-off job that starts, runs, and stops.
2. **Orchestrated gated retraining** — [`retraining/pipeline.py`](../retraining/pipeline.py) defines a
	**SageMaker Pipeline** whose steps are themselves Processing jobs, wired through a condition
	(quality gate). This is the "automated, audited, gated" path.

> Key mental model: **A Processing job is temporary compute for one script. A Pipeline is an
> orchestration graph of steps, each of which runs as a Processing job.** Everything trains through
> *Processing jobs*, never through SageMaker "Training jobs" (which require an `Estimator`, unused
> here).

Trace the SageMaker contract in `retraining/pipeline.py`:

```text
EventBridge Scheduler
  -> SageMaker Pipeline
	  -> TrainModels: train demand and fare candidates on temporary managed compute
	  -> EvaluateModels: score candidates and production baselines on the same test split
	  -> QualityGate: check absolute RMSE, baseline-relative quality, and release authorization
		  -> BuildAndDeploy only when every condition passes
```

Use these terms consistently:

- **Candidate:** the newly trained model being considered, not yet the released model.
- **Baseline:** the seed production-registry model used for a like-for-like comparison.
- **Absolute gate:** demand RMSE must be at most `50`; fare RMSE must be at most `5`.
- **Relative gate:** no compared candidate metric may degrade beyond the configured `20%` tolerance.
- **Authorization gate:** `DeployAfterQualityGate` must be `true` before `BuildAndDeploy` is eligible.

The weekly schedule supplies `DeployAfterQualityGate=false`. It automates training and evaluation,
not deployment. Drift detection remains a separate workflow and does not automatically start this
pipeline.

### 5b. Runtime + pipeline evidence

```powershell
$Region = "us-east-1"
$Cluster = terraform -chdir=infra/terraform output -raw ecs_cluster_name
$Service = terraform -chdir=infra/terraform output -raw ecs_service_name
$ApiUrl = terraform -chdir=infra/terraform output -raw api_url

Invoke-RestMethod "$ApiUrl/health" | ConvertTo-Json -Depth 4
aws ecs describe-services --region $Region --cluster $Cluster --services $Service `
	--query "services[0].{Status:status,Desired:desiredCount,Running:runningCount,Pending:pendingCount}"

aws sagemaker describe-pipeline --region $Region `
	--pipeline-name taxi-forecasting-retrain `
	--query "{Name:PipelineName,Status:PipelineStatus,Updated:LastModifiedTime}"
$ExecutionArn = aws sagemaker list-pipeline-executions --region $Region `
	--pipeline-name taxi-forecasting-retrain --max-results 1 `
	--query "PipelineExecutionSummaries[0].PipelineExecutionArn" --output text
aws sagemaker list-pipeline-execution-steps --region $Region `
	--pipeline-execution-arn $ExecutionArn `
	--query "PipelineExecutionSteps[].{Step:StepName,Status:StepStatus}"
aws scheduler list-schedules --region $Region `
	--name-prefix taxi-forecasting-staging --query "Schedules[].{Name:Name,State:State}"
aws scheduler get-schedule --region $Region `
	--name taxi-forecasting-staging-retrain `
	--query "{State:State,Expression:ScheduleExpression,TargetInput:Target.Input}"
```

Example output (sanitized):

```json
// describe-services -> the API service
{ "Status": "ACTIVE", "Desired": 1, "Running": 1, "Pending": 0 }

// list-pipeline-executions (most recent first)
[ { "Status": "Failed" },      // bounded baseline-gate experiments
  { "Status": "Failed" },
  { "Status": "Succeeded" } ]  // the authorized end-to-end validation run
```

A healthy run shows the pipeline steps in reverse-chronological order, for example:

```text
QualityGate     Succeeded
EvaluateModels  Succeeded
TrainModels     Succeeded
```

`BuildAndDeploy` is legitimately **absent** when `DeployAfterQualityGate=false`.
`QualityGate Succeeded` means the condition step executed successfully, not that
all release conditions were true. Inspect the condition outcome and evaluation
report separately; deployment authorization is false on this route.

### 5c. Enumerate every SageMaker resource (and read the empties correctly)

This is the heart of "the console looks empty." Run the read-only enumeration and interpret each
result. Empty lists are **expected** and are themselves teaching points.

```powershell
$Region = "us-east-1"
aws sagemaker list-pipelines --region $Region --query "PipelineSummaries[].PipelineName"
aws sagemaker list-processing-jobs --region $Region --sort-by CreationTime --sort-order Descending `
	--max-results 12 --query "ProcessingJobSummaries[].{Name:ProcessingJobName,Status:ProcessingJobStatus}"
aws sagemaker list-training-jobs --region $Region --max-results 5 --query "TrainingJobSummaries[].TrainingJobName"
aws sagemaker list-endpoints --region $Region --query "Endpoints[].EndpointName"
aws sagemaker list-model-package-groups --region $Region --query "ModelPackageGroupSummaryList[].ModelPackageGroupName"
aws sagemaker list-models --region $Region --query "Models[].ModelName"
```

Example output (sanitized) — the shape you should expect on staging:

```json
// list-pipelines
[ { "Name": "taxi-forecasting-retrain" } ]

// list-processing-jobs (recent; names encode their pipeline execution + step)
[ { "Name": "taxi-baseline-eval-fixed-...",              "Status": "Completed" },
  { "Name": "pipelines-uygdvehsr80c-EvaluateModels-...", "Status": "Completed" },
  { "Name": "pipelines-uygdvehsr80c-TrainModels-...",    "Status": "Completed" } ]

// the four "empty" lists are EXPECTED, not errors:
list-training-jobs         -> []
list-endpoints             -> []
list-model-package-groups  -> []
list-models                -> []
```

Interpret the results with this table:

| Console section | Expected here | Why |
| --- | --- | --- |
| **Pipelines** | `taxi-forecasting-retrain` (1) | The gated retraining pipeline. |
| **Processing jobs** | Several (`pipelines-…-TrainModels…`, `…-EvaluateModels…`, standalone eval) | Every training/eval step runs as a Processing job. |
| **Training jobs** | **Empty** | Project uses `ScriptProcessor` (Processing), not an `Estimator`. |
| **Endpoints** | **Empty** | Serving stays on ECS Fargate; no always-on SageMaker endpoint (cost decision). |
| **Model registry** (package groups) | **Empty** | Models are registered in **MLflow**, not the SageMaker registry. |
| **Models** | **Empty** | No `Model` objects created because there is no endpoint deployment. |
| **MLflow tracking servers** | Account-specific; not required | The implemented default uses SQLite/MLflow artifact bundles, not a managed tracking server. |

> Teaching point: an *empty* Endpoints / Training jobs / Registry page is a **design signature**, not
> a gap. It tells you this platform serves on ECS, trains via Processing, and tracks with MLflow.

### 5d. Tour the console, plane by plane

Open the console (account ID, username, and ARNs hidden) and read one page per plane as a live
confirming view, tracing each back to its `.tf` file:

**SageMaker (ML-ops plane), reading each page as *what to expect*:**

- **Dashboard** counts only active/running resources → shows zeros for completed work. Do not judge
	the deployment from here.
- **Pipelines UI** requires a Studio domain (there are zero) → unreachable, and that is fine.
- **Data preparation → Processing jobs** is the one page that shows the real work without a domain.
	Job names encode their pipeline execution (`pipelines-<executionId>-<StepName>-<hash>`).
- **Inference → MLflow tracking servers**, if available, lists account-specific
	managed servers. Their existence does not establish that this project logs there.

**The other three planes:**

- **Serving:** **ECS** service (desired/running `1/1`) and task definition; **EC2 → Load Balancers**
	(the ALB) and **Target Groups** (health); **ECR** (the pushed image tag); **CloudWatch → Log
	groups** (the service stream).
- **Delivery:** **CodeBuild** (project + latest succeeded build); **S3** (baked-artifact bucket).
- **Data:** **S3** (data-lake and monitoring buckets); **Kinesis → Firehose** (capture stream).
- **ML-ops (rest):** **CloudWatch → Alarms** (both `OK`) and the monitoring **Dashboard**; **SNS**
	(alerts topic); **EventBridge → Scheduler** (weekly retraining and drift schedules).

**Cross-cutting and opt-in (complete the sweep):**

- **Resource Groups → project group** — the tag-based page listing every project resource; use it as
	the "did we miss anything?" checkpoint.
- **IAM → Roles** — the six roles; open one to show its inline policy scope (no ARNs on screen).
- **EC2 → Security Groups** — ALB group public on :80; service group sourced from the ALB group.
- **CloudWatch → Log groups → subscription filters** — the tap feeding Firehose.
- **Guardrails** — ECR lifecycle policy; S3 Block Public Access on all four buckets; data-lake
	versioning; monitoring-bucket capture expiry.
- **CodeBuild → webhook** and the stored GitHub source credential (existence only, never the token).
- **Opt-in stacks if enabled:** Streamlit dashboard ECR/ALB/target group/ECS service/log group
	(`enable_dashboard`), and Athena workgroup + Glue database + results bucket (`enable_athena`). If
	disabled, show the variable default rather than hunting for an absent resource.

Show only names, statuses, counts, and health. Expected staging evidence is a healthy API, ECS
desired/running `1/1`, an active retraining pipeline, successful train/evaluate/quality-gate steps,
and the weekly schedule configured with deployment disabled. Do not start a pipeline execution,
update the ECS service, or take any mutating action during this lab.

## Task 6: Record Evidence and Limits

Record the Terraform files inspected, validation result, request path, and one
limitation. Static validation proves configuration syntax and internal
references; it does not prove resource creation, service health, permissions,
or production readiness. The optional AWS queries provide current runtime evidence, but only for
the queried region, resources, and time.

## Task 7: Cite Reproducible Evidence

Use sources included in the student repository and distinguish their evidence types:

- **Implemented contract:** [the pipeline](../retraining/pipeline.py) and
	[monitoring infrastructure](../infra/terraform/monitoring.tf) show the gates
	and schedule conditions. Code proves implementation, not execution.
- **Historical summary:** [the recording companion](RECORDING_COMPANION.md) explains dated staging
	observations, including the September 2026 retraining verification. It is a
	summary, not a fresh AWS query or a substitute for raw execution evidence.
- **Current observation:** retain sanitized output and query time from Task 5,
	when authorized AWS access is available. Otherwise record static inspection
	and mark current runtime verification as skipped.

For each source, state what it supports and what it does not prove. Instructor
recording materials and generated verification files are not prerequisites for
this exercise and are not bundled with the student export.

## Completion Check

- The request path was predicted before inspection.
- The AWS resource inventory table was reviewed and at least five resources traced to their `.tf` file.
- Terraform validation ran, or the static fallback was documented.
- ALB, ECS, ECR, health checks, and logs have distinct roles.
- The CI-to-ECR-to-ECS release path is explained.
- The two SageMaker execution paths (standalone Processing job vs. orchestrated Pipeline) are distinguished.
- The enumeration was run (or read) and each **empty** section (Training jobs, Endpoints, Registry) explained as a design signature.
- The reasons the console can look "empty" (dashboard = active only; Pipelines UI needs a Studio domain) are stated.
- The console was toured plane by plane (serving, delivery, data, ML-ops) with each page traced to its `.tf` file and private identifiers hidden.
- The cross-cutting components were shown: Resource Groups, the six IAM roles, the security groups, the log subscription filter, and the S3/ECR guardrails.
- The opt-in stacks (Streamlit dashboard, Athena/Glue) were shown, or their disabled defaults were stated.
- Live API, ECS, SageMaker, and schedule evidence was inspected, or the optional cloud step was
	explicitly skipped.
- Source code and a dated project summary were cited with their limits; current
	runtime evidence was retained or explicitly marked as skipped.
- No secrets, account identifiers, or live private outputs are included.