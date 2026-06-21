# Architecture

## System Context

The Radiology Triage Assistant sits between the hospital PACS (Picture Archiving and Communication System) and radiologist worklist management. It does **not** replace radiologist judgment — it reorders their reading queue so the most urgent studies are read first.

## Component Diagram

```mermaid
flowchart TD
    subgraph hospital["Hospital Network (on-prem or VPN)"]
        PACS["PACS\n(DICOM C-STORE / STOW-RS)"]
        RIS["RIS / Worklist\n(HL7 v2.5 or FHIR R4)"]
    end

    subgraph ingestion["Ingestion Layer (us-east-1)"]
        GW["DICOM Gateway\nOrthanc 23.x\nTLS mutual auth\nPHI de-id at boundary"]
        IQ["Kafka Topic\ndicom.raw\n(retention 24 h, 3 replicas)"]
    end

    subgraph processing["Processing Layer"]
        PRE["Pre-processor\n- DICOM parse (pydicom)\n- Windowing + resize 512×512\n- Pixel normalization\n- Study manifest assembly"]
        MQ["Kafka Topic\ndicom.ready\n(retention 2 h)"]
    end

    subgraph serving["Serving Layer"]
        INF1["Triton Pod 0\nA10G GPU\nEfficientNet-V2-M TRT FP16\nconcurrency=8"]
        INF2["Triton Pod 1\nA10G GPU\n(active–active)"]
        LB["Internal Load Balancer\n(round-robin, health-checked)"]
    end

    subgraph postproc["Post-Processing & Routing"]
        POST["Score Aggregator\n- Slice-level → study-level score\n- Threshold classification\n  STAT: ≥85 / URGENT: 60–84 / ROUTINE: <60\n- Routing decision"]
        WL_PUSH["Worklist Pusher\n(HL7 ORM / FHIR SR)"]
        AUDIT_W["Audit Writer\n(append-only)"]
    end

    subgraph storage["Durable Storage"]
        PGAUDIT["Postgres 15\naudit_predictions table\n(immutable via trigger)"]
        S3["S3 Bucket\nraw DICOM archive\nserver-side encryption\nObject Lock 10-year"]
        REGISTRY["MLflow Registry\n(model artifacts + metadata)"]
    end

    subgraph observability["Observability"]
        PROM["Prometheus\n(scrape interval 15 s)"]
        GRAF["Grafana\n(dashboards)"]
        PD["PagerDuty\n(on-call alerts)"]
        SIEM["SIEM / CloudTrail\n(security audit)"]
    end

    PACS -->|"DICOM TLS 1.3"| GW
    GW -->|"de-identified JSON + pixel blob"| IQ
    IQ --> PRE
    PRE --> MQ
    MQ --> LB
    LB --> INF1
    LB --> INF2
    INF1 --> POST
    INF2 --> POST
    POST --> WL_PUSH
    POST --> AUDIT_W
    WL_PUSH -->|"HL7 / FHIR"| RIS
    AUDIT_W --> PGAUDIT
    AUDIT_W --> S3
    INF1 --> PROM
    INF2 --> PROM
    POST --> PROM
    PRE --> PROM
    PROM --> GRAF
    PROM --> PD
    PROM --> SIEM
    REGISTRY -->|"model pull on startup"| INF1
    REGISTRY -->|"model pull on startup"| INF2
```

## Data Flow (per study)

```
T+0 ms    PACS sends DICOM C-STORE to DICOM Gateway
T+50 ms   Gateway de-identifies PHI, publishes to dicom.raw
T+200 ms  Pre-processor consumes, decompresses JPEG2000/RLE,
          assembles up to N slices into a 3D tensor (or picks top-k slices)
T+350 ms  Normalized tensor published to dicom.ready (≤ 2 MB after resize)
T+500 ms  Triton consumes, runs TRT FP16 inference, returns slice scores
T+700 ms  Score aggregator computes study-level urgency score
T+750 ms  Worklist push to RIS; audit record written to Postgres + S3
─────────────────────────────────────────────────────────────
p50 total: ~900 ms   p95 total: ~3 200 ms   budget: 4 000 ms
```

## Failure Modes & Mitigations

| Failure | Detection | Mitigation |
|---|---|---|
| GPU pod crash | Triton health probe fails | K8s restarts pod; second GPU pod absorbs load |
| Kafka lag spike | `kafka_consumer_lag > 500` alert | Auto-scale pre-processor pods (HPA) |
| Model version mismatch | `X-Model-Version` header check in post-processor | Alert `ModelVersionMismatch`; block promotion |
| PACS connectivity loss | Gateway connection count drops to 0 | Alert; studies queue in PACS until reconnect |
| Score threshold config error | p95 latency or STAT rate anomaly | Automated rollback runbook triggers |
| Audit write failure | Postgres write error rate > 0 | Inference blocked until audit store recovers (safety-critical path) |


## Backpressure & Queue Handling

- Kafka consumer lag is continuously monitored
- Horizontal autoscaling is triggered based on lag thresholds
- Maximum queue delay SLO: 2 seconds
- If delay exceeds threshold, alerts are triggered and traffic is throttled
  

## Security Boundaries

- PHI is de-identified **at the DICOM Gateway** before entering Kafka or any cloud-native service.
- All inter-service communication uses mTLS inside the cluster (Istio service mesh).
- S3 bucket has Object Lock (WORM) with 10-year retention — audit records cannot be deleted.
- Model registry access requires OIDC token with `ml-engineer` role; production promotion requires two approvals.
- All API endpoints require `Authorization: Bearer <jwt>` with `aud: radiology-triage`.
