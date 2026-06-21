# Load Test Plan — Radiology Triage Inference Service

## Purpose

Verify that the system meets all SLOs defined in `serving/slos.yaml` under steady-state, peak, and burst load conditions before any production deployment or model version promotion.

## Test Tool

**k6** (Grafana k6 v0.50+) with the k6-kafka extension for producing synthetic DICOM study events directly into Kafka.

Alternative: Locust with a custom DICOM producer plugin (fallback if k6-kafka is unavailable in the test environment).

## Test Environment

- **Cluster:** Staging (identical GPU node type: g5.2xlarge × 2)
- **Data:** Synthetic de-identified DICOM studies generated from a reference distribution matching production pixel statistics (mean/std of pixel intensity, slice count distribution)
- **Isolation:** Test Kafka topics (`dicom.raw.loadtest`, `dicom.ready.loadtest`) separate from production
- **Model version:** Same Production model version as target deployment
- **Duration:** Each scenario runs for minimum 15 minutes after ramp-up

## Test Scenarios

### Scenario 1: Steady-State Baseline

Validates normal operating conditions.

```
Target: 30 studies/min (0.5/sec)
Ramp: 0 → 30/min over 2 min
Hold: 15 min
Ramp down: 30 → 0/min over 2 min

Pass criteria (from SLO definitions):
  - SLO-LAT-01: p95 e2e latency ≤ 4 000 ms  ✓ target
  - SLO-LAT-02: p99 e2e latency ≤ 7 000 ms  ✓ target
  - SLO-ERR-01: error rate < 0.5%
  - SLO-AUD-01: audit completeness = 100%
  - SLO-MOD-01: X-Model-Version header = Production version in 100% of responses
  - SLO-QUE-01: queue delay p95 ≤ 2s
```

### Scenario 2: Peak Load

Validates capacity at peak throughput (SLO-THR-01).

```
Target: 100 studies/min (1.67/sec)
Ramp: 0 → 100/min over 5 min
Hold: 15 min  (must sustain ≥ 10 min per SLO-THR-01)
Ramp down: 100 → 0/min over 2 min

Pass criteria:
  - SLO-LAT-01: p95 e2e latency ≤ 4 000 ms
  - SLO-THR-01: < 0.1% study drop rate
  - SLO-ERR-01: error rate < 0.5%
  - SLO-QUE-01: queue delay p95 ≤ 2s
  - Kafka lag must remain within threshold defined in monitoring/alerts.yaml
  - CPU pre-processor HPA must scale to ≥ 4 pods within 5 min of peak start
```

### Scenario 3: Burst Spike

Validates queue absorption and recovery from a sudden traffic spike.

```
Baseline: 30 studies/min for 5 min
Spike: 200 studies/min for 2 min (2× peak)
Return: 30 studies/min for 10 min

Pass criteria:
  - No studies dropped during spike (all queued in Kafka)
  - p95 latency returns to ≤ 4 000 ms within 5 min of spike end
  - No error rate increase during recovery
  - Queue delay may exceed 2s during spike but must recover ≤ 2s within 5 min
```

### Scenario 4: Worst-Case Study Size

Validates latency with maximum-slice studies.

```
100% of synthetic studies: 48 slices, 80 MB each
Rate: 30 studies/min
Duration: 10 min

Pass criteria:
  - SLO-LAT-01: p95 e2e latency ≤ 4 000 ms
  - SLO-LAT-03: p95 GPU inference latency ≤ 1 500 ms
  - No GPU OOM errors
```

### Scenario 5: Single GPU Pod Failure (Resilience)

Validates N+1 redundancy.

```
Load: 50 studies/min (midpoint between average and peak)
At T+5 min: kill one Triton GPU pod (kubectl delete pod)
Duration after kill: 10 min

Pass criteria:
  - K8s restarts killed pod within 3 min (liveness probe)
  - Temporary SLO-LAT-01 breach is expected; system must recover within 2 min
  - Zero studies dropped
  - After pod recovery (T+5+3 min): p95 returns to ≤ 4 000 ms within 2 min
```

### Scenario 6: Model Version Mismatch Detection

Validates monitoring/alerts.yaml `ModelVersionMismatch` alert.

```
Manually deploy a Triton pod with a non-Production model version tag
Run 30 studies/min for 5 min

Pass criteria:
  - ModelVersionMismatch alert fires within 60 seconds
  - Alert includes the mismatched version string in annotations
  - Alert resolves within 60 seconds after pod is reverted to Production version
  - SLO-MOD-01 must drop below 100% within 60s of mismatch
```

## Metrics to Collect

All metrics scraped from Prometheus during test run:

| Metric | SLO |
|---|---|
| `histogram_quantile(0.95, rate(study_e2e_latency_ms_bucket[5m]))` | SLO-LAT-01 |
| `histogram_quantile(0.99, rate(study_e2e_latency_ms_bucket[5m]))` | SLO-LAT-02 |
| `histogram_quantile(0.95, rate(triton_inference_request_duration_us_bucket[5m])) / 1000` | SLO-LAT-03 |
| `histogram_quantile(0.95, rate(kafka_message_delay_seconds_bucket[5m]))` | SLO-QUE-01 |
| `rate(study_inference_errors_total[1m]) / rate(study_inference_total[1m]) * 100` | SLO-ERR-01 |
| `rate(audit_records_written_total[1m]) / rate(study_scores_dispatched_total[1m]) * 100` | SLO-AUD-01 |
| `kafka_consumer_group_lag{topic="dicom.ready"}` | Capacity |
| `kube_deployment_status_replicas_available{deployment="preprocessor"}` | HPA |
| `nvidia_gpu_utilization_percent` | GPU utilization |

## Pass/Fail Gate

All Scenarios 1, 2, 4 must pass before a model version is promoted from Staging to Production (in addition to the registry approval process in lifecycle/lifecycle.md). Scenario results are attached to the MLflow model version as artifacts.

Scenario 5 and 6 are run quarterly as part of game-day exercises, not as a gate for every deployment.

## Test Data Generation

```python
# Pseudocode for synthetic DICOM study generator
import pydicom
import numpy as np

def generate_synthetic_study(slice_count=12, size_px=512) -> list[pydicom.Dataset]:
    """Generate a synthetic DICOM study with realistic pixel statistics."""
    slices = []
    # Sample pixel intensity distribution from production reference stats
    # mean=128, std=45 (approximates chest X-ray distribution)
    for i in range(slice_count):
        ds = pydicom.Dataset()
        ds.Rows = size_px
        ds.Columns = size_px
        ds.BitsAllocated = 16
        ds.PixelData = (
            np.random.normal(128, 45, (size_px, size_px))
            .clip(0, 4095)
            .astype(np.uint16)
            .tobytes()
        )
        ds.StudyInstanceUID = pydicom.uid.generate_uid()
        ds.SOPInstanceUID = pydicom.uid.generate_uid()
        slices.append(ds)
    return slices
```
