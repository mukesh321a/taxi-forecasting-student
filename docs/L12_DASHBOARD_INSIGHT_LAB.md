# L12: Present a Dashboard Insight

- **Duration:** 45 to 60 minutes
- **Cloud required:** Yes for the primary route; a labeled local fallback is available.
- **Docker required on the learner machine:** **No**
- **Outcome:** A working dashboard panel connected to evidence and stakeholder action.

> **Docker boundary:** do not start Docker Compose for this lab. The primary
> route points Streamlit at an API already running in AWS. The local fallback
> starts the API with Uvicorn and the dashboard with Streamlit in two terminals.
> References to ECS tasks describe the remote AWS runtime. A "SHAP image" is a
> PNG plot artifact, not a Docker image.

## What the Dashboard Combines

The dashboard is a presentation layer over sources with different owners and
freshness rules. Do not describe the entire dashboard as simply "live."

| Panel type | Immediate source | What "current" means |
| --- | --- | --- |
| Health | `GET /health` | The selected API route responded at the recorded request time. |
| Fare, demand and explanations | API response | The request used the model currently loaded by that API process. |
| Responsible AI | Fairness JSON, SHAP PNG and model card | The files belong to a named analysis run or one staged S3 snapshot. |
| Drift | Newest configured drift JSON | The report file time and represented data period are stated. |
| Monitoring KPIs | API forecast plus configured assumptions | The forecast time and business assumption are both disclosed. |

An API health response proves readiness at one moment, not model accuracy,
capacity, freshness of every artifact, or a service-level objective. A rendered
panel proves presentation, not that its source was recently regenerated.

## Task 1: Select and Verify the API Route

Install `requirements-dashboard.txt` in the project environment as described in
[SETUP.md](SETUP.md). For the local fallback, complete its data and registration
steps for both models. Generate or stage any fairness/drift artifacts needed
for the chosen panel; the dashboard does not create them automatically.

For the primary AWS route, prepare the dashboard against the staging ECS API
behind its Application Load Balancer:

```powershell
.\scripts\aws\prepare_dashboard_staging.ps1
```

The script reads Terraform's `api_url`, requests `/health`, requires both models
to report ready, and sets `API_BASE_URL` for the current PowerShell process.
Retain the URL, health fields, and request time as AWS evidence. Do not expose
credentials or account identifiers.

In the AWS Console, confirm the same request path:

1. Open **ECS** -> **Clusters** -> `taxi-forecasting-staging` and show the API
	service's desired tasks, running tasks, and current deployment.
2. Open the staging API load balancer and show its DNS name and HTTP listener.
3. Open its target group and show registered-target health.
4. Compare the ALB DNS name with `API_BASE_URL`.

Keep the account menu, task environment values, and unrelated networking
details out of view. A running ECS task is not sufficient health evidence by
itself; retain both target health and the application `/health` response.

To also use the latest L11 fairness evidence and available drift reports from
S3, run:

```powershell
.\scripts\aws\prepare_dashboard_staging.ps1 -DownloadS3Artifacts
```

This downloads the two fairness reports, two SHAP summaries, and two model
cards into `artifacts/aws-showcase/responsible/`, then sets `RESPONSIBLE_DIR`
and `MODEL_CARDS_DIR` to that same snapshot. It also sets `DRIFT_DIR` for
available drift reports. The dashboard still reads local files, but their
recorded source is S3. The script retains API check time and S3
`head-object` details under `artifacts/verification/l12-s3-sources-*.json`;
download time alone is not generation time. An empty drift prefix is allowed,
but must be reported as **no staged drift evidence** rather than a current
no-drift result.

If AWS is unavailable, label the route **local fallback**. In the first terminal:

```powershell
.\.venv\Scripts\python.exe -m uvicorn src.serving.api:app --host 127.0.0.1 --port 8000
```

Verify health from a second terminal:

```powershell
Invoke-RestMethod http://127.0.0.1:8000/health
```

## Task 2: Start the Dashboard

In the second terminal:

For the local fallback only, set `$env:API_BASE_URL = "http://127.0.0.1:8000"`
before starting Streamlit. For AWS, keep the value set by the preparation script
and run Streamlit in that same PowerShell session; do not overwrite it with localhost.

```powershell
.\.venv\Scripts\python.exe -m streamlit run dashboard/app.py
```

Open the URL printed by Streamlit. Keep credentials, personal paths, and
unrelated browser tabs out of view.

## Task 3: Verify a Panel

Choose one demand, fare, monitoring, or responsible-AI panel. Identify its data
or API source in `dashboard/`, reproduce one displayed value from the referenced
artifact or endpoint, and record whether the panel is current for this run.

Trace and reconcile the panel in this order:

1. Name the panel, stakeholder question and decision owner.
2. Locate its renderer in `dashboard/sections.py` or `dashboard/app.py`.
3. Follow the call to `dashboard/api_client.py` or `dashboard/data.py`.
4. Record the exact endpoint or artifact path and its source time.
5. Read the raw value and identify sorting, grouping, summing, rounding or
	formatting performed before display.
6. Reproduce the displayed value using only that visible transformation.
7. State cache behavior, represented period and whether the route is AWS or
	local fallback.

If the value cannot be reproduced, preserve the discrepancy. Check source
selection, cache state, rounding and artifact freshness instead of choosing the
more convenient value.

## Task 4: Present the Insight

Use this structure:

```text
Panel and evidence source:
Observed value or pattern:
Stakeholder meaning:
Recommended action and owner:
Limitation or freshness boundary:
```

The action must follow from the displayed evidence. An ALB health response
supports staging reachability at the request time; it does not establish
production freshness, capacity, or service levels.

## Task 5: Stop Both Processes

Press `Ctrl+C` in the Streamlit terminal. For the local fallback, then stop
Uvicorn. The AWS route does not start or stop ECS resources.

## Completion Check

- API health was verified before opening the dashboard.
- The executed route is labeled staging AWS or local fallback.
- For the AWS route, `API_BASE_URL` came from Terraform's `api_url` output.
- ECS service state, ALB target health, and application health were reconciled.
- Any S3-backed panel records its bucket/key metadata and freshness boundary.
- One panel was traced to an artifact or endpoint.
- One displayed value was reproduced or reconciled.
- The stakeholder action names an owner and evidence-based reason.
- AWS/local scope and data-freshness limitations are stated.
