# Capacity Plan — Radiology Triage Inference Service

## Workload Parameters

These parameters are the single source of truth. All other documents (slos.yaml, load-test-plan.md, serving/capacity-plan.md) derive their numbers from here.

| Parameter | Value |
|---|---|
| Average throughput | 30 studies / min (0.5 studies/sec) |
| Peak throughput | 100 studies / min (1.67 studies/sec) |
| Max study size | 80 MB (raw DICOM) |
| Average study size | 35 MB |
| Average slice count | 12 slices |
| Max slice count | 48 slices |
| p95 end-to-end latency budget | 4 000 ms |
| p99 end-to-end latency budget | 7 000 ms |
| Daily study volume | ~18 000 studies/day (peak 08:00–12:00, 13:00–17:00) |

The system operates as a near-real-time (asynchronous) pipeline due to large DICOM payload sizes and preprocessing requirements.

## Latency Budget Decomposition

Total p95 budget: **4 000 ms**

| Stage | p50 | p95 | Notes |
|---|---|---|---|
| DICOM Gateway receipt + de-id | 30 ms | 80 ms | Network + CPU; scales horizontally |
| Kafka publish (`dicom.raw`) | 5 ms | 20 ms | Same-AZ, MSK |
| Pre-processor (decompress, resize, normalize) | 120 ms | 400 ms | CPU-bound; HPA enabled |
| Kafka publish (`dicom.ready`) | 5 ms | 20 ms | |
| Triton inference (TRT FP16, A10G) | 90 ms | 1 400 ms | Dominated by max-slice worst-case |
| Post-processor (score aggregation, routing) | 20 ms | 60 ms | |
| Kafka consumer-to-result propagation | 10 ms | 30 ms | |
| Postgres audit write | 15 ms | 80 ms | Indexed insert |
| RIS worklist push (HL7) | 50 ms | 200 ms | External system; async retry on failure |
| **Total (sum of p95 not additive; measured e2e)** | **~900 ms** | **~3 200 ms** | **Margin: 800 ms vs 4 000 ms budget** |

Queue delay is bounded to ≤ 2 seconds under normal operation and is monitored via Kafka consumer lag metrics.

## GPU Sizing

### Per-study inference cost (A10G, TRT FP16)

| Scenario | Slices | GPU time per study | Throughput per GPU |
|---|---|---|---|
| Average study | 12 | ~120 ms | ~500 studies/min |
| Worst-case study | 48 | ~480 ms | ~125 studies/min |
| Dynamic batch (8 studies simultaneously) | 12 each | ~180 ms total | ~2 600 studies/min |

### Required GPU capacity at peak (100 studies/min)

With Triton dynamic batching (batch size 8, max queue delay 50 ms):
- Effective throughput per A10G: ~400 studies/min (conservative, accounting for worst-case studies in batch)
- **1 A10G is sufficient for throughput**, but provides no redundancy
- **2 A10G (active–active)**: 800 studies/min capacity = **8× headroom over peak**

Two-pod deployment provides:
- N+1 redundancy (one pod can absorb full peak during GPU maintenance/restart)
- Safe canary deployment (route 5% traffic to new model version on second pod)
- Zero-downtime model version upgrades

### GPU Instance Selection

| Instance | GPU | VRAM | On-Demand/hr | Spot/hr |
|---|---|---|---|---|
| g5.2xlarge | A10G 24 GB | 24 GB | $1.212 | ~$0.45 |
| g5.4xlarge | A10G 24 GB | 24 GB | $1.624 | ~$0.60 |
| p3.2xlarge | V100 16 GB | 16 GB | $3.06 | ~$0.92 |

**Choice: 2× g5.2xlarge** — A10G provides the best TRT FP16 performance per dollar. V100 is older with worse FP16 throughput. g5.4xlarge provides double CPU/RAM but GPU is identical; not needed.

**Spot instances:** Not used for GPU pods. Spot interruption would cause Triton restart + 15–30 s model download, breaching p95 SLO during the interruption. Reserved Instance (1-year) pricing ~$0.79/hr reduces cost 35%.

## Degraded Mode

In case of GPU failure:

- CPU fallback is activated for inference
- System continues processing with reduced throughput
- Latency SLO is relaxed to p95 ≤ 7 seconds
- Alerts are triggered for operator intervention

This ensures system availability at the cost of latency during GPU outages.

## CPU Sizing (Pre-processor, Post-processor, Gateway)

| Component | Instance | Count | vCPU per pod | RAM per pod |
|---|---|---|---|---|
| DICOM Gateway | c6i.xlarge | 2 (HA) | 4 | 8 GB |
| Pre-processor | c6i.2xlarge | 2–6 (HPA) | 8 | 16 GB |
| Post-processor | c6i.large | 2 (HA) | 2 | 4 GB |
| API service | c6i.large | 2 (HA) | 2 | 4 GB |
| Kafka (MSK) | kafka.m5.large | 3 brokers | managed | managed |
| Postgres (RDS) | db.r6g.large | 1 primary + 1 replica | managed | managed |

### Pre-processor HPA

```yaml
minReplicas: 2
maxReplicas: 6
metric: kafka_consumer_lag{topic="dicom.raw"} > 100
scaleUp: +1 pod per 60s
scaleDown: -1 pod per 300s (conservative to avoid thrashing)
```

At peak (100 studies/min), each pre-processor pod handles ~20 studies/min (4 pods total). Each study requires decompressing up to 80 MB JPEG2000, resizing, normalizing: ~200 ms CPU time. 4 pods × (1000 ms / 200 ms) = 20 studies/min/pod × 4 = 80 studies/min capacity — HPA will scale to 6 at sustained peak.

## Storage Sizing

| Store | Usage | Retention | Size estimate |
|---|---|---|---|
| S3 (raw DICOM archive) | 80 MB/study × 18 000/day | 10 years | ~5.2 PB over 10 years |
| S3 (model artifacts) | ~500 MB/version × 20 versions | 10 years | ~10 GB |
| Postgres (audit_predictions) | ~1 KB/prediction × 18 000/day | 10 years | ~65 GB |
| MSK (Kafka, dicom.raw) | 80 MB × 100 studies × 24 h | 24 h retention | ~170 GB |
| MSK (Kafka, dicom.ready) | 2 MB × 100 studies × 2 h | 2 h retention | ~1.4 GB |

S3 Intelligent-Tiering moves data to Glacier after 90 days — estimated cost drops from $0.023/GB to $0.004/GB after 90 days.

## Cost Breakdown

- GPU (2× A10G): ~$1,138/month
- CPU compute (all services): ~$1,220/month
- Kafka (MSK): ~$330/month
- Database (RDS): ~$350/month
- Storage (S3 + Glacier): ~$80/month
- Misc (networking, logging, ECR): ~$211/month

Total: ~$3,329/month

## Monthly Cost Estimate

| Resource | Count | Unit cost | Monthly |
|---|---|---|---|
| g5.2xlarge (1-yr RI) | 2 | $569/mo | $1 138 |
| c6i.xlarge (gateway) | 2 | $122/mo | $244 |
| c6i.2xlarge (preprocessor, avg 3 pods) | 3 | $244/mo | $732 |
| c6i.large (post-proc, API) | 4 | $61/mo | $244 |
| MSK kafka.m5.large (3 brokers) | 3 | $110/mo | $330 |
| RDS db.r6g.large (primary + replica) | 2 | $175/mo | $350 |
| S3 (new data per month, 43 GB/day) | ~1.3 TB | $0.023/GB | $30 |
| S3 Glacier (historical) | grows over time | $0.004/GB | $50 (yr 1) |
| ECR, CloudWatch, VPC, NAT | misc | — | $150 |
| MLflow tracking server (c6i.large) | 1 | $61/mo | $61 |
| **Total** | | | **~$3 329/mo** |

*Approximately $4 200/mo when amortizing one-time setup, support plans, and data transfer costs.*

## Scaling Strategy

**Vertical scaling (GPU):** If average study slice count increases beyond 20, consider g5.4xlarge or g5.12xlarge (4× A10G) for batch parallelism — no architectural change required.

**Horizontal scaling (GPU):** Triton supports multi-instance model loading. Adding a third A10G pod requires only updating the HPA maxReplicas — Kafka consumer group handles load distribution automatically.

**Multi-region:** Currently single-region (us-east-1). For multi-hospital deployment in different geographies, replicate the full stack per region. Model registry is single-region; model artifacts are synced to regional S3 buckets via S3 Cross-Region Replication.
