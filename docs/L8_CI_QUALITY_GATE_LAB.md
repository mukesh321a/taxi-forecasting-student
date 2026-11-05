# L8: Break and Repair the CI Quality Gate

- **Duration:** 35 to 45 minutes
- **Cloud required:** Optional instructor demonstration; students need no credentials
- **Docker required on the learner machine:** **No**
- **Outcome:** A reversible controlled lint failure, an honestly reported
	repository-wide gate result, and a read-only hosted-build inspection.

> **Docker boundary:** do not install or start Docker for this lab. The learner
> runs Ruff, pytest/coverage, and optionally Terraform through
> `scripts/ci_local.ps1`. References to `docker build`, image push, ECR, and ECS
> describe stages executed by AWS CodeBuild. You inspect those lines in
> `buildspec.yml`; you do not execute them locally.

## Task 1: Inspect the Gates

Open `scripts/ci_local.ps1` and `buildspec.yml`. Both run lint, tests, and
coverage. Terraform formatting/validation checks belong to the local script;
the hosted buildspec does not run Terraform. Image build, push, and the ECS
update belong to the hosted deployment path. The local script does not deploy.

## Task 2: Create a Controlled Lint Failure

First confirm `Test-Path scratch_ci_failure.py` returns `False`. If it already
exists, stop and preserve it rather than overwriting someone else's file.
Create the temporary file containing an unused import:

```powershell
Set-Content scratch_ci_failure.py 'import os'
.\.venv\Scripts\python.exe -m ruff check scratch_ci_failure.py
```

Confirm Ruff reports `F401`. This file is intentionally outside application
code and will be removed in the next task.

## Task 3: Repair and Recheck

```powershell
Remove-Item scratch_ci_failure.py
.\.venv\Scripts\python.exe -m ruff check .
```

Record the repository-wide result separately from repairing the deliberate
scratch error. Verify the temporary file is gone:

```powershell
Test-Path scratch_ci_failure.py
```

The final command should return `False`.

In the finalized R8 recording, the scratch repair is complete while three
separate Ruff findings keep the wider gate red; pytest passes. That is historical
evidence, not the required result of today's checkout. Report current findings
honestly, do not conceal them, and do not repair unrelated code just to force a
green demonstration. A failed required lint stage is not overridden by passing tests.

## Task 4: Run the Local CI Workflow

When Terraform is installed:

```powershell
.\scripts\ci_local.ps1
```

When Terraform is unavailable:

```powershell
.\scripts\ci_local.ps1 -NoTerraform
```

Record which route was used and the final result.

## Task 5: Trace the Cloud-Only Stages

In `buildspec.yml`, identify where the pipeline logs in to ECR, builds the
Docker image, pushes the immutable tag, and requests an ECS service update.
Confirm that this buildspec has no ECS stability waiter or smoke test. Those are
separate post-deployment checks, so a successful CodeBuild run does not by
itself prove that the new task became healthy. Inspect these steps without
running them locally or exposing AWS credentials, account identifiers, or
repository URLs.

## Task 6: Inspect One Hosted Build

The instructor runs this read-only extension with the authorized staging profile. Students should
not enter credentials. Keep account IDs, ARNs, and repository URLs off screen.

```powershell
$Region = "us-east-1"
$Project = terraform -chdir=infra/terraform output -raw cicd_project_name
$BuildId = aws codebuild list-builds-for-project `
	--project-name $Project --region $Region --sort-order DESCENDING `
	--query "ids[0]" --output text
aws codebuild batch-get-builds --ids $BuildId --region $Region `
	--query "builds[0].{Status:buildStatus,Revision:resolvedSourceVersion,Phase:currentPhase,Phases:phases[].{Name:phaseType,Status:phaseStatus}}"
```

Compare the hosted phases with `buildspec.yml`. A successful build supports that those configured
stages ran and the ECS update was requested for one revision; it does not prove
service stability, smoke-test success, or that every future build or runtime
request will succeed.

## Completion Check

- A controlled `F401` failure is visible.
- The temporary file was removed and the repository-wide lint result was
	recorded separately, including any remaining failures.
- The local CI script completed or the failing gate was identified.
- Local quality checks are distinguished from cloud build and deployment.
- One hosted CodeBuild status and its phase results were inspected, or the optional cloud step was
	explicitly skipped.
- No credentials or private AWS identifiers are visible.