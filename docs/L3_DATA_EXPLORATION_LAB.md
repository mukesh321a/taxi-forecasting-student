# L3: Explore the Data and Trace Its Lineage

- **Cloud required:** No; initial source download requires internet.
- **Outcome:** Three findings about cleaning, demand, and feature/split behavior,
  each grounded in the executed notebook or persisted pipeline evidence.
- **Recording companion:** Follow the finalized L3 route below. L4 revisits the
  persistence and consistency checks; it does not replace this exploratory walkthrough.

## 1. Read the Complete Route

Open the [lineage diagram](data-lineage-route.png). Trace source, import,
cleaning, the demand/fare branch, chronological splits, and downstream consumers.
Distinguish transformation arrows from reference/evidence arrows.

Keep the three row meanings explicit:

- Raw trips: one published completed trip.
- Demand: one observed pickup zone during one hour, including inserted zero counts.
- Fare: one accepted completed trip after cleaning, targeting `fare_amount`.

## 2. Verify the Official Source

Open the [NYC TLC Trip Record Data page](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page).
Identify Yellow Taxi trip Parquet and the taxi-zone lookup CSV. The recording
uses January 2024; this is the configured teaching snapshot, not a claim that it
is the latest publication. Zones are broad labels, not identities or GPS traces.

## 3. Connect Source to Code

Inspect [configuration](../config/config.yaml),
[ingestion](../src/data/ingestion.py), and
[the pipeline entry point](../scripts/run_pipeline.py), in that order. Record
the configured period, source URLs, filenames, and local destinations. Separate
download/load responsibility from validation and feature engineering.

## 4. Run and Open the Notebook

Complete [environment setup](SETUP.md) first. With raw inputs already present,
the recorded persistence command is:

```powershell
.\.venv\Scripts\python.exe -m scripts.run_pipeline --skip-download
```

If a raw file is missing, acquire it using the recorded fallback:

```powershell
.\.venv\Scripts\python.exe -m scripts.run_pipeline
```

Open [the exploration notebook](../notebooks/01_explore_dataset.ipynb) and select
the project `.venv` kernel. The recording preparation also rebuilds this notebook
with `python -m scripts.build_explore_notebook`. That command overwrites the
generated notebook, so use it only when intentionally regenerating it, not after
adding work you need to retain. No rebuild is required merely to open the notebook.

## 5. Execute the Notebook in Order

Run one code cell at a time and interpret its output before continuing. Follow
the recorded sequence, using section names if cell positions have changed:

| Notebook segment | Evidence to inspect |
| --- | --- |
| Setup and source inventory | Project root, interpreter, provider, files, and source period |
| Raw inputs and schema | Trip preview, required fields, types, and zone lookup |
| Cleaning | Unsuitable values, ordered rules, retained/removed counts and reasons |
| Demand aggregation | Unique zone-hour keys, dense dimensions, zero rows, hourly profile, busiest zones |
| Demand features | Calendar inputs, past-only lags/rolling statistics, and history warm-up |
| Fare features | One clean trip per row, distance/duration/passengers/zones/calendar, and fare-distance plot |
| Chronological splits | All six train/validation/test outputs and their time boundaries |
| Artifacts and lineage checks | Saved paths, row-count relationships, unique keys, and assertions |

The notebook calculates stages in memory to explain them. The pipeline writes
the official persisted outputs. Do not claim a saved file was refreshed merely
because its in-memory equivalent was recalculated.

## 6. Reconcile Persisted Evidence

Check the two raw inputs, clean trips, hourly demand, pipeline summary, both
complete feature tables, and all six split files: thirteen expected artifacts
in the recorded route. File existence proves persistence, not correctness.

Run the notebook's final assertions. Confirm cleaning did not add trips, fare
rows match clean rows, demand zone-hour keys are unique, and split times move
forward. Inspect the lag/rolling code separately to establish past-only inputs.

## 7. Present Three Findings

Use current outputs, not memorized recording values:

1. **Cleaning:** raw, retained, and removed row counts; largest removal reason.
2. **Demand:** zones, hours, zone-hour rows, zero-count rows, and busiest zone.
3. **Features and splits:** fare row preservation, demand history warm-up, and
   chronological split boundaries.

For each finding record the notebook output or artifact, its configured period,
what it supports, and one limit. Counts can differ when input data or configuration
changes; preserve and investigate the difference instead of forcing a match.

## Completion Check

- Source and configuration are identified before interpreting results.
- All notebook sections were executed or a specific failed/skipped step is recorded.
- The three row meanings and the two feature branches remain distinct.
- Pipeline outputs and notebook calculations were reconciled.
- Three findings cite observed evidence.
- Limitations cover one-month Yellow Taxi scope, broad geography, no direct
  demographic attributes, project cleaning assumptions, and no currentness guarantee.