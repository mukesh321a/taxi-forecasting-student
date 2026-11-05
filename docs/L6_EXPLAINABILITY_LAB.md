# L6: Explain Model Behavior

- **Duration:** 35 to 45 minutes
- **Cloud required:** No
- **Outcome:** A plain-language explanation grounded in SHAP and model-card evidence.

## Prerequisites

This preparation supports the recorded four-task walkthrough; it is not a new
training exercise inside L6.

Complete the feature pipeline and register a fare model. L5's `--no-register`
comparison creates tracked runs, but not the registered model that this analysis
loads. If no fare model is registered, run this bounded teaching configuration:

```powershell
.\.venv\Scripts\python.exe -m scripts.train_models --problem fare --n-trials 1
```

In the MLflow UI, confirm `taxi-fare-predictor` has a registered version and
accessible model artifacts. A one-trial model is sufficient for this exercise,
not production approval. If a suitable version already exists, reuse it and
record its identity instead of retraining. As in the finalized recording, record
the configured reference `models:/taxi-fare-predictor/latest` unless configuration
differs. The generated card currently shows registry version `n/a`; do not infer
a numeric version from `latest`. Add a resolved version only when separately
verified, and otherwise retain the identity gap.

## Task 1: Generate Fare Analysis

From the repository root:

```powershell
.\.venv\Scripts\python.exe -m scripts.responsible.run_analysis --problem fare
```

Record the command result and the artifact paths it creates or refreshes.

## Task 2: Inspect Explainability Evidence

Open the fare outputs under `artifacts/responsible/` and the fare model card
under `docs/model_cards/`. Identify the most influential features, the direction
or magnitude the evidence supports, and the model/version represented.

Follow the recorded global beeswarm reading order:

1. Vertical feature order summarizes mean absolute SHAP influence over the sample.
2. Horizontal position is signed contribution relative to the model baseline.
3. Color represents the feature's low-to-high input value, not good/bad outcomes.
4. Each dot is one sampled row for one feature; density and overlap show spread.
5. Read numeric location IDs as labels, not geographic distance or ordered value.

The global plot does not explain one named trip. Use a single-row explanation
for that question. Correlated inputs can share attribution, and neither the plot
nor its ranking establishes causality, fairness, or production readiness.

## Task 3: Write a Bounded Explanation

Use this structure:

```text
For this analyzed model and data sample, [feature] has [observed influence].
This suggests [plain-language interpretation]. The evidence does not prove
[causal or individual-level claim].
```

SHAP describes how model inputs contribute to predictions. It does not prove
that changing a feature causes the real-world outcome to change.

## Task 4: Connect the Model Card

Record one intended use, one limitation, and one monitoring or review action
from the model card. Explain how each item changes the way the SHAP result
should be communicated.

## Completion Check

- The responsible-analysis command completed or its failure was explained.
- The explanation cites a generated artifact and the represented model.
- Feature influence is described without claiming causation.
- Intended use and limitations from the model card are included.
- The conclusion stays within the analyzed sample and model version.