# Configuration-aware crop-model comparison of APSIM and WOFOST soybean simulations

This repository contains the simulation workflow, analysis code, configuration records, and selected outputs supporting the manuscript **“Configuration-aware crop-model comparison of APSIM and WOFOST soybean simulations.”**

The study compares APSIM Next Generation and WOFOST 8.1 for rainfed soybean near Ames, Iowa, USA. Its central finding is that similar simulated grain yield does not necessarily imply agreement in simulated crop states: the evaluated model configurations produced comparable mean yield but differed substantially in biomass, leaf area index (LAI), harvest index, seasonal trajectories, and sensitivity allocation.

## Study scope

The comparison was designed as a configuration-controlled diagnostic study, not as a model-ranking exercise.

- Study location: Ames, Iowa, USA (42.03° N, 93.63° W)
- Crop: soybean (*Glycine max* (L.) Merr.)
- Core simulation period: 2014–2024
- Models:
  - APSIM Next Generation 2025.10.7899.0
  - WOFOST 8.1 (`Wofost81_NWLP_CWB_CNB`)
- WOFOST crop package: official Soybean 906
- PCSE runtime: locally patched PCSE 6.0.9
- Weather source: NASA POWER
- Conditions: rainfed, with active water and nitrogen processes

The public Ames yield records used in the manuscript provide external agronomic context only. They were not used for calibration, process-level validation, or model ranking.

## Experimental components

### 1. Core paired comparison

The core design combines:

- 11 weather years (2014–2024)
- 2 sowing dates (DOY 135 and 152)
- 2 nitrogen rates at sowing (50 and 100 kg N ha⁻¹)

This produces 44 scenarios for each model. All scenarios were retained for simulation and Sobol analysis. For endpoint and seasonal-trajectory summaries, nitrogen treatments were averaged within each year-by-sowing combination, producing 22 analysis units.

### 2. Analogous-process Sobol analysis

Six analogous process groups were evaluated:

1. Pre-flowering phenology
2. Post-flowering phenology
3. Canopy expansion
4. Assimilation capacity
5. Biological nitrogen fixation
6. Background nitrogen supply

The Saltelli design used 256 base samples and generated 2,048 parameter combinations per model. Each combination was evaluated across all 44 core scenarios, resulting in 90,112 simulations per model. First-order and total-order Sobol indices were estimated, with uncertainty assessed using 5,000 block-bootstrap replicates.

### 3. Design B output-signature audit

Design B tests whether APSIM and WOFOST retain distinguishable output signatures across a larger generated simulation space:

- 150 baseline scenario groups
  - 25 weather years
  - 3 sowing dates
  - 2 nitrogen rates
- 150 perturbation groups
  - 1 anchor
  - 12 one-factor boundary perturbations
  - 137 Latin Hypercube perturbations
- 2 model outputs per scenario–perturbation combination

The complete design contains 22,500 paired units and **45,000 quality-controlled simulation profiles**.

A deterministic leakage-controlled classifier was trained using 44 approved output and design-summary features. Identifiers, filenames, hashes, target-derived fields, metadata-derived variables, and model-specific state identifiers were excluded. Feature attribution is used descriptively to explain classifier separation; it is not interpreted as causal evidence, agronomic validation, or proof of model superiority.

## Repository structure

```text
.
├── README.md
├── requirements.txt
├── environment.yml
├── configs/
│   ├── apsim_cli_path.env.example
│   ├── cloud_local_switch.md
│   ├── full_simulation_config.json
│   └── workers_32_cloud.json
├── scripts/
│   ├── designb_core.py
│   ├── full_simulation_runner.py
│   ├── preflight_check.py
│   ├── parallel_smoke_test.py
│   ├── resume_full_simulation.py
│   └── qa_full_outputs.py
├── inputs/
│   ├── Baseline_Scenarios/
│   ├── Perturbation_Matrix/
│   └── Weather_Library/
├── wofost_runtime/
├── patch_payload/
├── analysis/
│   ├── features/
│   ├── classifier/
│   └── xai/
└── results/
    ├── simulation/
    ├── features/
    ├── classifier/
    └── xai/
```

Large row-level daily outputs are stored separately from the Git repository. See `DATA_AVAILABILITY.md` for their archive location and checksums.

## Software requirements

### APSIM Next Generation

Install APSIM Next Generation **2025.10.7899.0** and point the workflow to the `Models` executable:

```bash
export APSIM_CLI=/path/to/APSIM/bin/Models
```

Examples are provided in `configs/apsim_cli_path.env.example` and `configs/cloud_local_switch.md`.

APSIM is distributed separately by the APSIM Initiative. Users are responsible for obtaining it under the applicable APSIM licence.

### Python environment

Create the documented environment using either Conda:

```bash
conda env create -f environment.yml
conda activate apsim-wofost-designb
```

or `pip`:

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

The WOFOST analysis requires the documented PCSE 6.0.9 patch. Apply it only within a dedicated environment:

```bash
bash scripts/apply_pcse_patch.sh
```

Patch files, diffs, and audit records are retained under `patch_payload/` and `wofost_runtime/`.

## Reproducing Design B

Run all commands from the repository root.

### 1. Preflight validation

```bash
python scripts/preflight_check.py
```

This checks the APSIM executable, Python/PCSE runtime, scenario and perturbation counts, weather coverage, WOFOST crop repository, soil file, and expected package structure.

### 2. WOFOST runtime smoke test

```bash
python scripts/run_wofost_single_smoke_after_patch.py
```

### 3. Parallel smoke test

```bash
bash scripts/run_parallel_smoke_after_patch.sh
```

Do not start the full experiment unless the smoke-test and input-QA checks pass.

### 4. Full simulation

```bash
python scripts/full_simulation_runner.py --mode full
```

The runner supports resumable execution. If a run is interrupted, use:

```bash
python scripts/resume_full_simulation.py
```

### 5. Merge and validate outputs

```bash
python scripts/merge_status.py
python scripts/qa_full_outputs.py
```

The expected completed Design B package contains 45,000 profiles: 22,500 APSIM and 22,500 WOFOST profiles.

## Reproducing the classifier and feature-attribution analysis

After simulation QA has passed, run:

```bash
python analysis/features/phase5_2_feature_builder.py
python analysis/classifier/phase5_3_classifier.py
python analysis/xai/phase5_4_xai.py
```

Run the unit tests with:

```bash
python -m unittest discover -s analysis/features -p 'test_*.py'
python -m unittest discover -s analysis/classifier -p 'test_*.py'
python -m unittest discover -s analysis/xai -p 'test_*.py'
```

Key reproducibility outputs include:

- `results/simulation/full_simulation_results.csv`
- `results/simulation/full_simulation_summary.json`
- `results/features/full_simulation_feature_table.csv`
- `results/features/full_simulation_feature_dictionary.csv`
- `results/features/full_simulation_feature_qa.json`
- `results/features/full_simulation_leakage_audit.csv`
- `results/classifier/full_simulation_metrics.json`
- `results/classifier/full_simulation_cv_results.csv`
- `results/classifier/full_simulation_predictions.csv`
- `results/xai/full_simulation_shap_global_importance.csv`
- `results/xai/full_simulation_shap_group_summary.csv`
- `results/xai/full_simulation_shap_subset_audit.csv`
- `results/xai/full_simulation_shap_xai_summary.json`

## Main reported findings

Across the 44 core scenarios:

- Mean grain yield:
  - APSIM: 2,527.1 kg ha⁻¹
  - WOFOST: 2,658.1 kg ha⁻¹
- Mean maximum above-ground biomass:
  - APSIM: 6,249.9 kg ha⁻¹
  - WOFOST: 4,300.7 kg ha⁻¹
- Mean maximum LAI:
  - APSIM: 3.086
  - WOFOST: 1.890
- Mean harvest index:
  - APSIM: 0.404
  - WOFOST: 0.617

These results support the manuscript’s central conclusion: **yield agreement without crop-state agreement** under the evaluated configurations.

## Interpretation boundaries

This repository supports a bounded comparison of one APSIM configuration and the official WOFOST Soybean 906 package at one site. The analyses do not:

- establish universal properties of APSIM or WOFOST;
- rank the models by field realism or predictive skill;
- isolate the effect of individual equations or parameters;
- treat classifier performance as agronomic validation; or
- treat feature attribution as causal process evidence.

Matched in-season observations of biomass, LAI, soil water, nitrogen status, flowering, and physiological maturity were not available for the simulated site-years. Crop-state results should therefore be interpreted as configuration-controlled diagnostic evidence rather than field validation.

## Data availability

Repository-scale inputs, configurations, scripts, QA summaries, and selected analysis outputs are included here. Large daily and endpoint simulation archives are available at:

> **Data archive:** `[ADD ZENODO/OSF DOI OR DOWNLOAD URL]`

File sizes and SHA-256 checksums should be recorded in `DATA_AVAILABILITY.md`.

## Citation

If you use this repository, please cite the associated manuscript and software release:

> `[ADD FINAL AUTHORS, YEAR, ARTICLE TITLE, JOURNAL, DOI]`

Citation metadata should also be provided in `CITATION.cff` after the manuscript DOI and repository release are finalized.

## Licence

Repository code licence: `[ADD SELECTED CODE LICENCE]`.

APSIM, PCSE, WOFOST crop data, NASA POWER data, and other third-party materials remain subject to their respective licences and terms of use. The repository licence does not replace or extend those third-party terms.

## Contact

For questions about the workflow or data, open a GitHub issue or contact:

> `[ADD CORRESPONDING AUTHOR NAME AND EMAIL]`
