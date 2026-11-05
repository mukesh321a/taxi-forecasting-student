# L11: Review Subgroup Performance

- **Duration:** 40 to 50 minutes
- **Cloud required:** Yes for the primary route; a labeled local fallback is available.
- **Docker required on the learner machine:** **No**
- **Outcome:** A documented disparity with measurable mitigation, versioned S3 evidence, and unresolved limitations.

> **Docker boundary:** all required analysis runs directly with the project
> Python environment. No Docker image or container is used. The primary route
> optionally publishes generated evidence to S3; when AWS is unavailable, keep
> the local artifacts and label S3 publication as not demonstrated.

## What This Review Measures

This lab compares prediction error across **operational slices** in the held-out
test data. It does not measure demographic fairness because the source data has
no direct protected-attribute fields.

| Dimension | Groups used by the analysis | Interpretation boundary |
| --- | --- | --- |
| `borough` | Pickup borough plus `Unknown` lookup rows | Geography is not a demographic identity. |
| `time_of_day` | Overnight, morning, afternoon, evening | A coarse time bucket does not identify an event or explain cause. |
| `day_type` | Weekday and weekend when represented | One represented group means no comparison is available. |
| `holiday` | Holiday and non-holiday when represented | Absence of a holiday group is a coverage limit, not parity. |
| `airport` | Airport and non-airport rows | Different trip or demand scale may contribute to error differences. |

For each qualifying group, retain these fields together:

- `n`: held-out rows used for the metric; groups below 30 rows are excluded.
- `MAE`: typical absolute prediction error.
- `RMSE`: error measure that gives extra weight to large misses.
- `MAPE`: relative error; interpret cautiously when actual values approach zero.
- `disparity_ratio`: worst-group RMSE divided by best-group RMSE.
- `flagged`: `true` when the ratio is greater than the configured `1.5` threshold.

A flag is an investigation trigger. It does not prove intent, causation,
discrimination, or general fairness status. A ratio of `1.0` with only one
qualifying group means there was no between-group comparison.

## Task 1: Generate Responsible-AI Evidence

First complete [setup Phases 1 and 2](SETUP.md): both problems need feature
splits, registered models, and accessible artifacts. An L5 `--no-register` run
or a fare-only L6 registration is insufficient for this two-model analysis.

```powershell
.\.venv\Scripts\python.exe -m scripts.responsible.run_analysis
```

Record the demand and fare outputs under `artifacts/responsible/`.

The command produces complementary evidence:

| Artifact | Question it answers |
| --- | --- |
| `demand_fairness.json`, `fare_fairness.json` | Which represented groups have different measured errors? |
| `*_shap_summary.png` | Which features influenced the fitted model on the analyzed sample? |
| `docs/model_cards/*.md` | What is the model for, how was it evaluated, and what limitations remain? |
| `docs/DATA_STATEMENT.md` | What population and attributes does the source data include or omit? |

SHAP describes fitted-model behavior; it does not explain the real-world cause
of a subgroup difference.

The current fairness JSON does not embed generation time, model version, or an
explicit evaluation period. The model card supplies generation and available
model context, while `docs/DATA_STATEMENT.md` supplies source-period context.
Record any identity or period that remains unavailable as a provenance gap.

## Task 2: Publish Versioned AWS Evidence

For the primary AWS route, confirm the active account without displaying
credentials, then publish the fairness reports, SHAP summaries, and model cards:

```powershell
aws sts get-caller-identity --query Account --output text
.\scripts\aws\publish_fairness_evidence.ps1
```

The script reads `cicd_artifacts_bucket` from Terraform, writes a stable
`dashboard/responsible/latest/` copy plus an immutable UTC-stamped copy under
`dashboard/responsible/versions/`, and retains the immutable objects'
`head-object` results under `artifacts/verification/`.

Record the bucket name, evidence version, S3 object keys, ETags, content lengths,
last-modified values, and custom metadata. Do not display credentials, account
identifiers, or unrelated bucket contents. An S3 upload proves durable object
storage for these generated files; it does not prove that the analysis ran in
AWS or that a mitigation has succeeded.

In the AWS Console, open **S3**, navigate only to
`dashboard/responsible/versions/<recorded-version>/`, and open one fairness
JSON object's **Properties**. Reconcile its key, size, last-modified value, ETag,
and custom metadata with the retained verification JSON. Keep the account menu
closed and crop or blur the account-number suffix in the bucket name.

If AWS credentials or the provisioned bucket are unavailable, keep the failure,
continue with local artifacts, and label the result **local fallback: S3
publication not demonstrated**.

## Task 3: Inspect Subgroup Metrics

For each problem, identify the grouping definition, sample count, error metric,
best- and worst-performing groups, disparity measure, and any threshold or flag.
Do not interpret a small group without reporting its sample size.

Use this reading order for every dimension:

1. Confirm that at least two qualifying groups are present.
2. Read each group's `n`, RMSE and MAE; use MAPE only with target-scale context.
3. Name the highest- and lowest-RMSE groups without assigning a cause.
4. Recalculate or verify `worst RMSE / best RMSE`.
5. Compare the ratio with `1.5` and record the flag.
6. State the operational impact that the measured error could influence.
7. List possible explanations as hypotheses requiring new evidence.

## Task 4: Write One Finding

Use this structure:

```text
For [available model reference, or identity unavailable] and [separately
verified evaluation context], [group] has [metric and value] compared with
[reference]. This is a flagged disparity because [threshold/evidence].
```

Avoid claiming that a geographic or temporal group represents a protected
attribute unless the data and analysis explicitly establish that relationship.

## Task 5: Propose Measurable Mitigation

State an owner, action, success metric, review date or trigger, and rollback or
escalation condition. Preserve the disparity as unresolved until new evidence
shows the stated success criterion has been met.

## Task 6: Check the Model Cards

Open the demand and fare model cards under `docs/model_cards/`. Confirm that the
finding, intended use, limitations, and mitigation language are consistent with
the generated fairness evidence.

## Completion Check

- Demand and fare subgroup evidence was inspected.
- The executed route is labeled AWS or local fallback.
- For the AWS route, immutable S3 keys and retained object metadata were verified.
- One versioned object was reconciled in the S3 Console with account details masked.
- The finding includes metric values and sample context.
- Proxy groups are not mislabeled as protected attributes.
- Mitigation has a measurable success criterion and owner.
- A flagged disparity is not presented as fairness clearance.
