# Architecture Pattern Justification

## Pattern Chosen: Async Queue-Based Inference Pipeline with Synchronous Result Push

### Core Pattern Description

Rather than a synchronous request/response model where the PACS calls an inference endpoint and waits, we use an **event-driven pipeline**:

```
PACS → DICOM Gateway → Kafka → Pre-processor → Kafka → Triton → Post-processor → RIS
```

The PACS fires and forgets; radiologist worklists are updated via a push to the RIS, not a polling loop. A thin REST API exists for downstream consumers (worklist UI, audit queries) but is **not** in the hot inference path.

### Why This Pattern for Scenario Z

#### 1. DICOM studies are large and arrival is bursty

At peak, 100 studies/minute = 1.67 studies/second. Each study can be 80 MB. A synchronous HTTP call carrying 80 MB per request over a single connection to an inference service would require enormous client-side timeouts, and a single slow study would block the thread. Kafka decouples arrival rate from processing rate: the gateway acknowledges receipt in < 100 ms; inference catches up asynchronously.

#### 2. The 4-second p95 budget is generous enough for async

4 000 ms p95 gives ample room for the Kafka round-trip (< 30 ms within the same AZ) plus preprocessing and inference (measured at ~700 ms p50 / ~2 800 ms p95 for 80 MB studies on A10G). A synchronous architecture would need careful connection-pool management to avoid head-of-line blocking; the queue absorbs burst naturally.

#### 3. Audit requirements demand decoupled durability

Every prediction must be auditable with the exact model version and input hash. Writing to Postgres and S3 in the hot path of a synchronous call would add tail latency and create a single-failure-mode: if the audit store is slow, the PACS blocks. In the async model, the post-processor writes to the audit store independently; if the audit write fails, inference is held (safety constraint) but the PACS is never blocked — the study sits in the queue.

#### 4. Model promotion requires isolation

Async pipeline + Triton's model versioning allows in-place model swaps: we load the new model version on Triton, run shadow scoring, then redirect traffic — all without redeploying the gateway or pre-processor. A monolithic sync API would require a coordinated redeploy of the entire call chain.

### Alternatives Considered

#### Alternative A: Synchronous REST gateway (PACS → API → Triton)

- **Pro**: Simpler; PACS gets immediate HTTP 200 with score
- **Con**: PACS must maintain open connections for up to 4 seconds per study; connection pool exhaustion at peak; no natural backpressure; audit write in the critical path; harder to A/B model versions
- **Decision**: Rejected. PACS software is typically intolerant of long-held connections, and the burst pattern makes queue-based backpressure essential.

#### Alternative B: Batch inference (hourly runs)

- **Pro**: GPU utilization maximized; cheaper
- **Con**: Completely incompatible with the 4-second latency requirement; STAT studies would wait up to an hour
- **Decision**: Rejected.

#### Alternative C: Serverless GPU (Lambda-style per-study invocation)

- **Pro**: Zero idle cost; elastic
- **Con**: Cold-start latency for GPU containers is 15–45 seconds; DICOM SDK initialisation adds further overhead; vendor lock-in for HIPAA BAA coverage varies; no persistent model-in-memory cache
- **Decision**: Rejected. Cold-start incompatible with p95 = 4 000 ms.

### Trade-offs Accepted

| Trade-off | Accepted Cost | Rationale |
|---|---|---|
| Operational complexity of Kafka | Higher ops burden vs. simple queue (SQS) | Need replay, consumer groups, and exactly-once semantics for audit |
| Always-on GPU nodes | ~$2 800/month idle cost | Cold-start incompatible with 4 s budget |
| Async result delivery | PACS cannot get a synchronous score | All PACS systems support result push via HL7; no PACS polled REST |
| Audit store in critical path | Inference blocked on audit write failure | Non-negotiable safety requirement for regulated SaMD |
