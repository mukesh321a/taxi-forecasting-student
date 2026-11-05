# L2: Guess the Fare and Feel the Metrics

- **Duration:** 45 to 60 minutes
- **Cloud required:** No
- **Outcome:** A plain-language comparison of fare errors using MAE and RMSE.

## Learning Objectives

1. Distinguish an actual fare from a predicted fare.
2. Calculate signed, absolute, and squared errors.
3. Explain why positive and negative errors can cancel.
4. Interpret MAE and RMSE in dollars.
5. Show why RMSE reacts more strongly to one large miss.

## Part A: Commit Fare Guesses

Complete `Your guess` and `Reason` before viewing Part B.

| Trip | Description | Your guess | Reason |
| --- | --- | ---: | --- |
| A | 1.2 miles, weekday morning, Midtown |  |  |
| B | 2.5 miles, weekday afternoon, Manhattan |  |  |
| C | 4 miles, evening, Manhattan to Brooklyn |  |  |
| D | 8 miles, evening airport route |  |  |
| E | 5.5 miles, late evening, Queens |  |  |

## Part B: Calculate the Shared Errors

Use the convention:

```text
Error = prediction - actual
```

| Trip | Actual fare | Shared prediction | Error | Absolute error | Squared error |
| --- | ---: | ---: | ---: | ---: | ---: |
| A | $10 | $12 |  |  |  |
| B | $15 | $13 |  |  |  |
| C | $20 | $22 |  |  |  |
| D | $30 | $40 |  |  |  |
| E | $25 | $23 |  |  |  |

Complete these calculations:

```text
Sum of signed errors =
Mean signed error =

MAE = sum of absolute errors / 5 =

Mean squared error = sum of squared errors / 5 =
RMSE = square root of mean squared error =
```

## Part C: Interpret the Baseline

1. What information can the mean signed error hide?
2. What does MAE mean for these five predictions?
3. Why is RMSE larger than MAE here?
4. Which trip contributes most to RMSE, and why?
5. Which operational groups should be checked separately before drawing a
   broad conclusion?

### Verify After Calculating

The recording verifies the hand calculation with the project's metric helpers.
Run this from the repository root after completing the baseline arithmetic:

```powershell
@'
import numpy as np
from src.models.common import mae, rmse

actual = np.array([10, 15, 20, 30, 25], dtype=float)
predicted = np.array([12, 13, 22, 40, 23], dtype=float)
print("MAE:", mae(actual, predicted))
print("RMSE:", rmse(actual, predicted))
'@ | .\.venv\Scripts\python.exe -
```

Compare full precision before rounding to cents. Inspect
[the metric helpers](../src/models/common.py) and
[their tests](../tests/unit/test_models.py). Reversing subtraction changes the
signed error, but not MAE or RMSE after absolute values or squaring.

## Part D: Extreme-Miss Comparison

Change only trip D's prediction from `$40` to `$60`.

| Metric | Baseline | Extreme case | Increase |
| --- | ---: | ---: | ---: |
| MAE |  |  |  |
| RMSE |  |  |  |

Record the changed trip-D values:

```text
Signed error =
Absolute error =
Squared error =
```

After calculating the extreme case, rerun the verification block with only the
fourth prediction changed from `40` to `60`, as in the recording. The actuals
and the other four predictions must stay unchanged.

## Exit Ticket

Complete each statement:

1. Mean signed error can mislead because...
2. MAE means...
3. RMSE is larger in the baseline because...
4. MAE and RMSE should be reported with...

## Completion Check

- All five personal fare guesses are recorded before the shared values appear.
- The shared error table contains signed, absolute, and squared errors.
- MAE and RMSE calculations show the working and retain dollar units.
- Repository code confirms the hand-calculated values.
- The extreme-miss comparison changes only trip D.
- The interpretations state sample size, context, and relevant subgroup checks.