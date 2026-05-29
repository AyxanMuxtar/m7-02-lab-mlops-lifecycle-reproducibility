# ETA Model — MLOps Lifecycle

```mermaid
flowchart TD
    A["🗄️ Raw Data Sources\n(Open-Meteo API, GPS telemetry,\ndelivery DB)"]
    B["📦 Versioned Dataset\nartifact: dataset_<sha256>.parquet\nstored: s3://northstar-data/eta/\ntool: DVC"]
    C["🧪 Experimentation\nMLflow tracking server\nartifact: run_<run_id>/\n(params.yaml, metrics.json)"]
    D["🏋️ Training Run\nartifact: run_<run_id>/model.onnx\ngit_sha pinned, seed fixed\ndocker image: northstar-trainer:<git_sha>"]
    E["📊 Evaluation\nartifact: eval_<run_id>.json\nfrozen holdout: eval_set_2024Q4.parquet\nMAE, per-city P90 latency error"]
    F["🟡 Registry: STAGING\nURI: models:/eta/<version>\nlineage: run_id + dataset_hash + git_sha"]
    G["✅ Registry: PRODUCTION\nURI: models:/eta/Production\ncanary: 10% → 100% traffic"]
    H["🪦 Registry: ARCHIVED\nretained for 10 versions\nthen deleted from registry\nartifact in S3 kept 90 days"]
    I["🚀 Inference Service\nKubernetes deployment\nartifact: k8s manifest + model URI\nSLA: p95 < 150 ms, ~200 RPS"]
    J["📡 Monitoring\nPrometheus + Evidently\nsignals: MAE drift, p95 latency,\nfeature distribution shift (PSI > 0.2)"]
    K["🔁 Retraining Trigger\ndrift_signal: PSI threshold breach\nor scheduled: every 14 days"]

    A -->|"ingest job (daily)\noutput: raw_snapshot_<date>.parquet"| B
    B -->|"dataset_<sha256> registered in DVC\n[automatic]"| C
    C -->|"analyst selects run_id for promotion\n[manual]"| D
    D -->|"CI pipeline executes eval script\n[automatic]"| E
    E -->|"MAE ≤ 4.2 min AND per-city ≤ 7 min\n[automatic gate]"| F
    F -->|"ML lead + product owner sign off\n[manual approval]"| G
    G -->|"canary deploy → shadow compare\n[automatic → manual confirm]"| I
    I -->|"live predictions logged\n[continuous]"| J
    J -->|"drift_signal emitted to Airflow DAG\n[automatic]"| K
    K -->|"triggers new ingest + label job\n[automatic]"| B
    G -->|"superseded by newer Production version\n[automatic]"| H
    E -->|"MAE gate FAILS → rejected, stays Candidate\n[automatic block]"| C

    style F fill:#FFF9C4,stroke:#F9A825,color:#5D4037
    style G fill:#C8E6C9,stroke:#388E3C,color:#1B5E20
    style H fill:#ECEFF1,stroke:#90A4AE,color:#455A64
```

## Annotation key

| Symbol | Meaning |
|--------|---------|
| `[automatic]` | Triggered by CI/CD or Airflow DAG — no human action |
| `[manual]` | Requires named approver to click approve in MLflow UI |
| `[automatic gate]` | Metric threshold checked by eval script; blocks promotion on failure |

## Artifacts at each stage

| Stage | Artifact | Location |
|-------|----------|----------|
| Raw data | `raw_snapshot_<date>.parquet` | `s3://northstar-data/raw/` |
| Versioned dataset | `dataset_<sha256>.parquet` | DVC remote (`s3://northstar-data/dvc/`) |
| Training run | `run_<run_id>/model.onnx` + `params.yaml` + `metrics.json` | MLflow artifact store |
| Evaluation | `eval_<run_id>.json` | MLflow run artifact |
| Staging model | `models:/eta/<version>` | MLflow Model Registry |
| Production model | `models:/eta/Production` | MLflow Model Registry |
| Deployed config | `k8s/eta-deployment-<version>.yaml` | Git repo |
| Monitoring signal | `drift_report_<date>.json` | Prometheus + Evidently export |
