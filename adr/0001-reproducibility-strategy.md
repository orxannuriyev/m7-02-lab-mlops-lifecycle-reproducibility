# ADR 0001: Reproducibility strategy for NorthStar models

## Context
NorthStar currently relies on an unstructured model lifecycle where binary artifacts reside in S3 under naming conventions like `eta_v2_FINAL.onnx` with no linked metadata. Because there is no central registry, dataset versioning, or environment pinning, it is currently impossible to trace a production model back to the exact code, data, and environment that generated it.

## Decision

**Environment**
We will guarantee execution environment reproducibility using **Docker**. Every training run and inference endpoint will execute inside a containerized environment defined by a version-controlled `Dockerfile`. Python dependencies will be strictly pinned using `poetry.lock` to ensure all transitive dependencies remain immutable.

**Data**
We will implement data versioning using **DVC (Data Version Control)** configured with our existing S3 buckets as the remote storage backend. Every training run will record the exact DVC hash corresponding to the data snapshot used, ensuring we can reconstruct the exact feature set, time-window, and ground-truth values ingested during that specific run.

**Code**
All experimentation and training runs will be logged using **MLflow Tracking**. To ensure traceability, our automated training pipeline (running via CI/CD) will strictly enforce that no model can be registered from a dirty Git working tree; every registered model will require a definitive, committed **Git SHA**. 

**Randomness**
We will enforce deterministic training by centralizing random seed configuration. A global configuration utility will set fixed seeds across all stochastic libraries utilized in our stack (e.g., `numpy.random.seed()`, `random.seed()`, and framework-specific seeds like `xgboost.set_config()`). Furthermore, we will explicitly configure our ML frameworks to use deterministic algorithms where applicable to prevent floating-point variances across different CPU/GPU architectures.

## Alternatives rejected
* **Relying on S3 object versioning/timestamps for data:** Rejected because S3 timestamps do not provide atomic, verifiable snapshots of complex, multi-file datasets, nor do they natively link to the codebase state at the time of training.
* **Conda environments or basic `requirements.txt`:** Rejected because cross-platform discrepancies and missing system-level binaries often break environment reproducibility; full Docker containerization isolates us from host-OS variables.
* **Manual tracking via wikis or spreadsheets:** Rejected because manual metadata entry is highly error-prone, doesn't scale to our bi-weekly retraining cycles, and cannot be programmatically validated by deployment gates.

## Consequences
* Engineers can no longer trigger ad-hoc training scripts from their local laptops for production. All release-candidate models must be built via the automated CI/CD pipeline to guarantee Git SHA and container integrity.
* Storage costs will increase moderately as DVC retains historical dataset versions and MLflow stores metadata and artifacts for every experiment.
* Feature engineering code must be updated to output deterministic datasets (e.g., sorting queries before saving to ensure row order doesn't change between runs).

## Revisit if
* The team migrates to a real-time streaming feature store where point-in-time "time-travel" queries (e.g., using Delta Lake or Apache Hudi) become a more native architectural fit for data versioning than DVC file snapshots.