# Sentiment Analysis — End-to-End MLOps Project

A production-style ML system built to prove out a full MLOps lifecycle — not to chase the best possible model. Given a sentiment classification model as the payload, this project answers a different question: **can this model reliably go from raw data to a monitored, containerized, cloud-deployed service, with tests and CI/CD gating every step?**

If you're only interested in modeling technique, this isn't that project. If you're evaluating whether someone can ship and operate an ML system in production, this is.

## Why this project exists

Most ML portfolios stop at a Jupyter notebook with a good accuracy score. This one deliberately keeps the model simple so the pipeline, deployment, and operations work can be the focus — the parts of the job that actually break in production and that most learning resources skip.

## Architecture

```
Raw Data → Ingestion → Preprocessing → Feature Engineering → Model Training → Evaluation → Registration
                                                                                      │
                                                                                      ▼
                                                                    Flask UI (serves predictions)
                                                                                      │
                                                                                      ▼
                                                        Docker image → Amazon ECR (registry)
                                                                                      │
                                                                                      ▼
                                                CI/CD pipeline (GitHub Actions) runs tests, builds, pushes
                                                                                      │
                                                                                      ▼
                                                        Amazon EKS (Kubernetes) — orchestrates the deployed service
                                                                                      │
                                                                                      ▼
                                          Amazon EC2 (compute) + Amazon S3 (artifact / model storage)
                                                                                      │
                                                                                      ▼
                                    Prometheus (metrics scraping) + Grafana (dashboards)
                                    — running on a dedicated EC2 instance, monitoring the live service
```

## Pipeline stages (DVC-orchestrated)

| Stage | What it does |
|---|---|
| `data_ingestion` | Pulls and splits the raw dataset (`test_size` configurable via `params.yaml`) |
| `data_preprocessing` | Cleans and normalizes raw text into an interim dataset |
| `feature_engineering` | Vectorizes text into model-ready features; persists the fitted vectorizer (`models/vectorizer.pkl`) |
| `model_building` | Trains the classifier on the engineered features |
| `model_evaluation` | Scores the trained model and writes metrics (`reports/metrics.json`) |
| `model_registration` | Registers the evaluated model as a versioned artifact for deployment |

Every stage is defined in `dvc.yaml` and reproducible with a single `dvc repro` — no manual, undocumented steps between "raw data" and "trained model."

## Serving & deployment

- **Flask UI** — a lightweight interface (`flask_app/`) that loads the registered model and vectorizer to serve live predictions.
- **Docker** — the Flask service is containerized (see `Dockerfile`) so it runs identically in dev and in the cluster.
- **CI/CD** — GitHub Actions (`.github/workflows/`) runs the test suite, builds the image, and pushes it to **Amazon ECR** on every change — the pipeline doesn't ship untested code.
- **Amazon EKS** — the container is deployed and orchestrated on a Kubernetes cluster (`deployment.yaml`), rather than run as a single manually-managed instance.
- **Amazon EC2** — underlying compute for the cluster / supporting services.
- **Amazon S3** — artifact and model storage.
- **Monitoring** — a dedicated EC2 instance runs **Prometheus** (scraping live service metrics) and **Grafana** (dashboards), wired to the deployed Flask/EKS service — so the system is observable in production, not just tested before deployment.

## Experiment tracking & reproducibility

- **DVC** versions data and pipeline stages, so any historical run can be reproduced exactly.
- **MLflow** tracks experiments — parameters, metrics, and model versions — so model choices are auditable rather than anecdotal.

## Testing

Automated tests (`tests/`) run as part of the CI/CD workflow before any image is built or pushed, so a broken pipeline stage or a broken API contract fails the build instead of reaching deployment.

## Tech stack

`Python` · `scikit-learn` · `DVC` · `MLflow` · `Flask` · `Docker` · `GitHub Actions` · `Amazon ECR` · `Amazon EKS` · `Amazon EC2` · `Amazon S3` · `Prometheus` · `Grafana`

## Results

- Model metric(s): `[ADD: pull the actual numbers from reports/metrics.json — e.g. accuracy / F1 — before publishing this]`
- Live demo: `[ADD: link if the EKS deployment is still running, otherwise note "torn down after demo to avoid ongoing AWS cost — see instructions below to redeploy"]`
- Monitoring: `[ADD: a screenshot of the Grafana dashboard here — even a static image is strong proof if the EC2 monitoring instance is no longer running]`

## Running it locally

```bash
git clone https://github.com/parth-kachhadiya/MLOps-capstone-project.git
cd MLOps-capstone-project
pip install -r requirements.txt

# Reproduce the full pipeline
dvc repro

# Run the Flask app
python flask_app/app.py
```

## What I'd improve next

- Add alerting rules on top of the existing Grafana dashboards (e.g. latency/error-rate thresholds), not just visibility
- Add model-quality drift detection (prediction distribution shift over time), not just infra metrics
- Expand feature engineering beyond a small fixed vocabulary for better generalization
- Add a staging environment gate in CI/CD before promoting to the EKS deployment

---

*Part of a broader MLOps practice — see [my other pipelines](https://github.com/parth-kachhadiya) for a variant sourcing from MongoDB Atlas with a fully modular training pipeline.*