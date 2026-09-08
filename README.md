# Acute Care Pathway — Discrete Event Simulation

A discrete event simulation (DES) of the acute medical admissions pathway through an
Emergency Department (ED), built with [SimPy](https://simpy.readthedocs.io/). The model
tracks individual patients from arrival through triage, ED/SDEC assessment, referral,
medical and consultant review, and admission to the Acute Medical Unit (AMU) or discharge.
It is designed to explore how demand and capacity changes affect flow, waiting times, and
4-/12-hour breach rates, and to evaluate alternative operational policies.

This repository accompanies the paper and contains the model, its input distributions, and
the scripts used to reproduce the baseline and scenario runs.

## Modelled pathway

```
arrival → triage (ambulance / walk-in) → SDEC referral
                                            ├─ accepted → SDEC (discharge/exit)
                                            └─ rejected → ED assessment → disposition:
                                                 ├─ discharge
                                                 ├─ refer other specialty (surgical bed delay)
                                                 └─ refer medicine → medical assessment
                                                       → consultant review
                                                       → AMU admission / discharge
```

Arrivals, NEWS2, acuity, referral probabilities, SDEC/AMU capacity, and staffing are all
driven by time-varying distributions (by day-of-week and hour). Doctor availability follows
configurable rota patterns with handover and break rules.

## Scenarios

The model has three variants, selected in `run.py` via `MODEL_VARIANT`:

| Variant  | Class (in `src/`)            | Description |
|----------|------------------------------|-------------|
| `baseline` | `Trial` → `Model`           | Standard ED-to-acute-medicine pathway. |
| `alt_1`    | `AltTrial` → `AltModel`     | **Direct triage to Medicine.** When SDEC is unavailable, clinically eligible patients (NEWS2 ≤ 4, acuity ≠ 1, medicine referral intent, optional top-X% admission-risk threshold) are referred straight to Medicine, bypassing ED assessment. |
| `alt_2`    | `AltTrial2` → `AltModel2`   | **ED-to-consultant direct route.** Medicine referrals decided 09:00–20:00 skip the post-assessment ED decision delay and attempt direct consultant assessment; if not started by 21:00 they hand over to the standard medical pathway. Includes an optional, off-by-default medical→ED staffing redirection. |

## Repository layout

```
run.py                          # main entry point (choose MODEL_VARIANT, runs N replications)
run_sensitivity_*.py            # one-factor sensitivity sweeps (see below)
requirements.txt

src/
  model.py                      # baseline DES model
  model_alt.py                  # alt_1 model (direct triage to medicine)
  model_alt2.py                 # alt_2 model (ED-to-consultant direct)
  trial.py / trial_alt.py / trial_alt2.py   # replication runners + output aggregation
  patient.py                    # patient entity
  global_parameters.py          # parameter container + input file paths
  base_params.py                # baseline parameter values, rotas, master seed
  helper.py                     # distributions, rota maths, I/O helpers

data/                           # aggregated input distributions (see below)
output/                         # simulation output (git-ignored, created on run)
documentation/                  # process map
```

## Installation

Requires **Python 3.10+** (developed on 3.12).

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

Dependencies: `numpy`, `pandas`, `scipy`, `simpy`.

## Running

From the repository root:

```bash
python run.py
```

Set the scenario at the top of `run.py`:

```python
MODEL_VARIANT = "baseline"   # or "alt_1", "alt_2"
```

By default this runs **50 replications**. Each replication simulates 56,160 minutes
(39 days) with a 7-day burn-in and 2-day cool-down, leaving a 30-day observation window.
The master seed (`MASTER_SEED` in `src/base_params.py`) is spawned into independent per-run,
per-process streams so results are fully reproducible.

### Sensitivity analyses

Each script sweeps one factor across `alt_1` and `alt_2` and writes results under
`output/des_output/sensitivity/`:

| Script | Factor varied |
|--------|---------------|
| `run_sensitivity_consultant_assessment_time.py` | consultant assessment time (µ, σ) |
| `run_sensitivity_consultant_discharge.py`        | post-consultant discharge probability |
| `run_sensitivity_referrals.py`                   | referral rate (odds multiplier) |
| `run_sensitvity_uptake_prob.py`                  | uptake of the direct pathway among eligible patients |

## Inputs

All model inputs are **aggregated distributions** under `data/` — no patient-level records
are included.

- `generator_distributions/` — arrival rate, booked-appointment probability, AMU bed release
  rate, and SDEC slot rate, each by day-of-week and hour.
- `staffing_resource/` — ED and medical doctor staffing by hour.
- `patient_attributes/` — NEWS2 distribution and the calibrated referral-probability
  distribution (`p_raw`, `p_cal`, `weight`) sampled per adult patient.

## Outputs

Running a scenario creates a timestamped batch folder under
`output/des_output/<scenario>/batch_<timestamp>/` containing aggregated CSVs across all
replications, including:

- `*_results.csv` — patient-level results (one row per patient, tagged with run number)
- `*_event_log.csv` — event log (arrival, assessment start/end, referral, admission, …)
- `*_summary_complete_allruns.csv` — summary measures averaged across runs
- `*_queue_*.csv` — queue-length time series (ED, medical, consultant, AMU)
- `*_resource_monitor.csv` — resource utilisation
- `*_seed_manifest.csv` — the RNG seeds used for each run

`output/` is git-ignored; results are regenerated by running the model.

## Reproducibility

A single master seed is expanded (via NumPy `SeedSequence`) into four independent streams
per replication (arrivals, service times, probabilities, resources). The exact seeds are
recorded in each batch's `*_seed_manifest.csv`.
