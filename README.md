# Satellite-Bench

<p align="center">
  <img src="https://img.shields.io/badge/status-concept%20%2F%20early%20development-orange" alt="Status: Early Development">
  <img src="https://img.shields.io/badge/Python-3.9%2B-blue" alt="Python 3.9+">
  <img src="https://img.shields.io/badge/License-Apache%202.0-green" alt="License: Apache 2.0">
</p>

<p align="center">
  <b>A benchmark for early-season crop yield prediction from sparse SAR-optical satellite time series in cloud-prone, data-scarce regions</b>
</p>

---

## Overview

Operational crop yield forecasting must inform decisions before harvest — for agricultural insurance, supply chain logistics, and food security early warning. Yet most satellite yield benchmarks evaluate models using the full growing season, including observations near or after physiological maturity. This masks a critical operational gap: **can models predict yield when only early-to-mid season imagery is available?**

Furthermore, existing benchmarks concentrate on large-scale mechanized agriculture in North America, Brazil, and Western Europe, where cloud-free optical imagery is abundant and field boundaries are clean. **Smallholder systems in cloud-prone tropical and subtropical regions** — where food security risks are highest and ground data is scarcest — remain largely excluded from standardized evaluation.

**Satellite-Bench** proposes a benchmark for **early-season yield prediction** using **sparse, irregular satellite time series** in data-scarce agricultural systems. The benchmark emphasizes two under-tested capabilities: (1) prediction from partial temporal observations, and (2) robustness to cloud-induced data gaps via **Sentinel-1 SAR and Sentinel-2 optical fusion**.

---

## Motivation

### The Operational Problem

Food security agencies and agricultural insurers require yield estimates weeks to months before harvest. By the time the full satellite time series is available, the operational window for intervention has closed. Early-season prediction is harder because:

- **Phenological signals are immature** — Biomass accumulation and stress indicators are weaker in vegetative stages
- **Cloud occlusion is front-loaded** — Tropical wet seasons overlap with early crop growth, leaving optical sensors with severe data gaps precisely when prediction is most needed
- **Ground truth is sparse** — Field-level yield data in smallholder systems is rarely collected systematically

Existing benchmarks do not test this regime. They provide complete, gap-filled time series and evaluate end-of-season accuracy, which is agronomically trivial and operationally late.

### Why Existing Benchmarks Are Insufficient

| Benchmark | Temporal Coverage | Cloud Handling | Smallholder Focus | Early-Season Task |
|-----------|-------------------|----------------|-------------------|-------------------|
| **USDA NASS** | County-level aggregates | N/A | No | No |
| **Radiant Earth Crop Type** | Full season composites | Masked/gap-filled | No | No |
| **CropHarvest** | Full season | Composite | Partial | No |
| **NASA Harvest** | Full season | Gap-filled | Limited | No |
| **Single-region yield datasets** | Full season | Varies | No | No |

None provide a **standardized early-season prediction task** with **raw, irregular time series** and **explicit evaluation at multiple forecast lead times** (e.g., 30, 60, 90 days before harvest).

---

## Proposed Benchmark Design

### Task Definition

**Primary Task (Regression):**  
Given a satellite time series for an agricultural field **truncated at a specified forecast date** (e.g., 60 days before typical harvest), predict the field-level crop yield.

**Secondary Task (Lead-Time Generalization):**  
Evaluate model performance across multiple forecast horizons (e.g., 90, 60, 30 days before harvest) to characterize how prediction accuracy degrades as temporal information decreases.

**Tertiary Task (Sensor Robustness):**  
Given optical data gaps, evaluate whether models can maintain accuracy using **Sentinel-1 SAR backscatter** as a cloud-penetrating alternative or complement.

### Dataset Requirements (Proposed)

| Criterion | Specification |
|-----------|---------------|
| **Agricultural system** | Smallholder and mixed-scale systems in cloud-prone regions |
| **Field size** | Mixed, including sub-pixel fields typical of smallholder mosaics |
| **Sensors** | Sentinel-2 (Level-2A surface reflectance) and Sentinel-1 (GRD backscatter) |
| **Temporal coverage** | From planting through harvest; raw observations only |
| **Cloud handling** | No gap-filling, no compositing; cloud masks provided; missing timestamps preserved |
| **Crops** | Multiple staple crops (e.g., maize, rice, soybean, millet) |
| **Regions** | Multiple cloud-prone agro-ecological zones |
| **Yield labels** | Field-level or sub-field-level ground truth |
| **Forecast horizons** | Fixed evaluation points: 90, 60, and 30 days before harvest |
| **Splits** | Spatial split (unseen regions); temporal split (unseen years) |

### Evaluation Protocol

Models are evaluated at each forecast horizon independently.

| Metric | Purpose |
|--------|---------|
| **RMSE** | Absolute error at each horizon |
| **MAE** | Robustness to outliers |
| **R²** | Variance explained |
| **rRMSE** | RMSE normalized by mean yield (cross-crop comparison) |
| **Skill score** | (Baseline RMSE − Model RMSE) / Baseline RMSE, against a naive persistence or climatology model |

**Cross-validation:** Fixed spatial split and temporal split. Random splits are provided for diagnostic use only.

### Planned Baselines

1. **Climatology mean** — Predict average historical yield for the region/crop
2. **Persistence model** — Predict yield using the most recent available NDVI value
3. **Linear regression on phenological features** — Hand-crafted features (green-up rate, peak VI) from available observations
4. **Classical machine learning** — Random Forest, XGBoost on temporal summaries
5. **Temporal deep learning** — LSTM or Transformer on irregular time series with missing-data handling
6. **SAR-optical fusion** — Early-fusion or cross-attention architectures combining Sentinel-1 and Sentinel-2

---

## Current Status

This repository is in **early development**.

| Component | Status |
|-----------|--------|
| Literature survey for field-level yield data | In progress |
| Data schema & loaders | Implemented |
| Temporal truncation and horizon splitting | Implemented |
| Climatology baseline | Implemented |
| Random Forest baseline | In progress |
| SAR-optical fusion protocol | Planned |
| Temporal deep learning baselines | Planned |
| Leaderboard | Planned |

---

## Dataset

### Proposed Schema

```python
{
    "field_id": str,                  # Unique field identifier
    "crop_type": str,                 # e.g., "maize", "rice", "millet"
    "region": str,                    # Agro-ecological zone or country
    "season_year": int,               # Growing season year
    "planting_date": str,             # ISO date
    "expected_harvest_date": str,     # ISO date (for horizon calculation)
    "actual_harvest_date": str,       # ISO date
    "yield_tons_per_ha": float,       # Target: field-level yield
    "geometry": dict,                 # GeoJSON polygon of field boundary
    "satellite_series": [             # Raw, ungapfilled observations
        {
            "sensor": str,            # "sentinel-1" or "sentinel-2"
            "date": str,              # ISO date
            "days_after_planting": int,
            "bands": dict,            # Reflectance or backscatter values
            "cloud_coverage_pct": Optional[float],  # None for SAR
            "valid_mask": bool,       # False if cloudy or noisy
        }
    ],
    "forecast_horizons": [90, 60, 30],  # Days before harvest for evaluation
    "source": str,                    # Data provider or survey
    "source_doi": Optional[str],      # Literature provenance
}
```

### Data Access

```python
from satellite_bench import load_dataset

# Load current development snapshot
data = load_dataset("early_season_dev")
```

*The full benchmark dataset will be released after curation, alignment, and validation. A development snapshot is available for methodology testing.*

---

## Installation

```bash
git clone https://github.com/your-org/Satellite-Bench.git
cd Satellite-Bench

python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt
pip install -e .
```

### Dependencies

- Python ≥ 3.9
- pandas, numpy
- scikit-learn ≥ 1.3.0
- rasterio, xarray (optional, for geospatial I/O)
- PyTorch ≥ 2.0 (optional, for deep learning baselines)

---

## Usage

### Running the Reference Baseline

```python
from satellite_bench.datasets import EarlySeasonDataset
from satellite_bench.models import ClimatologyBaseline
from satellite_bench.evaluation import BenchmarkEvaluator

dataset = EarlySeasonDataset(horizon_days=60, split="spatial")
model = ClimatologyBaseline()

evaluator = BenchmarkEvaluator(model, dataset=dataset)
results = evaluator.run()
print(results)
```

### Evaluating a Custom Model

```python
from satellite_bench.evaluation import BenchmarkEvaluator

# Your model must implement fit(X, y) and predict(X)
evaluator = BenchmarkEvaluator(your_model, dataset="early_season_dev", horizon_days=60)
results = evaluator.run(folds=5)
```

---

## Repository Structure

```
Satellite-Bench/
├── satellite_bench/
│   ├── datasets/          # Data loaders, temporal truncation, splitters
│   ├── models/            # Baseline implementations
│   ├── features/          # Spectral indices, SAR features, temporal summaries
│   ├── evaluation/        # Metrics, horizon-based evaluation protocols
│   └── utils/             # Helper functions
├── data/
│   ├── raw/               # Raw satellite and yield data (with attribution)
│   └── processed/         # Aligned, standardized datasets
├── experiments/           # Training scripts for baselines
├── tests/                 # Unit tests
├── notebooks/             # Exploratory data analysis
├── requirements.txt
├── setup.py
├── LICENSE
└── README.md
```

---

## Roadmap

| Milestone | Deliverable |
|-----------|-------------|
| **v0.1** | Frozen dataset for 1–2 crops in 1–2 cloud-prone regions; climatology + RF baselines; 60-day horizon evaluation |
| **v0.2** | Expanded crop and regional coverage; SAR-optical fusion protocol; LSTM/Transformer baselines |
| **v0.3** | Multi-horizon evaluation (90/60/30 days); sensor ablation studies; public leaderboard |
| **v1.0** | Final dataset freeze; benchmark paper submission; community challenge |

---

## Citation

```bibtex
@software{satellite_bench,
  title = {Satellite-Bench: A Benchmark for Early-Season Crop Yield Prediction from Sparse SAR-Optical Satellite Time Series},
  author = {TBD},
  year = {2026},
  url = {https://github.com/your-org/Satellite-Bench},
  note = {Concept and early development release}
}
```

---

## Contributing

We welcome contributions in:

- **Dataset curation:** Field-level yield data with georeferenced boundaries; SAR-optical alignment pipelines
- **Cloud-prone region focus:** Data from tropical and subtropical smallholder systems
- **Baselines:** Reference implementations for irregular time series, missing data, or SAR-optical fusion
- **Evaluation:** Metrics for lead-time degradation; protocols for spatial and temporal generalization

Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

---

## License

This project is licensed under the Apache License 2.0. See [LICENSE](LICENSE) for details.

---

## Contact

- **Issues:** [GitHub Issues](https://github.com/your-org/Satellite-Bench/issues)
- **Discussions:** [GitHub Discussions](https://github.com/your-org/Satellite-Bench/discussions)

---

<p align="center">
  <i>This is a living research document. The benchmark design, dataset, and protocols are open to community feedback prior to the v0.1 freeze.</i>
</p>
```
