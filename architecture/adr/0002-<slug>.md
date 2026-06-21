# ADR 0002 — Async Queue-Based Pipeline vs Synchronous REST for Inference Ingestion

**Status:** Accepted  
**Date:** 2026-06-07  
**Deciders:** ML Platform Lead, Integration Lead, CISO  

---

## Context

When a DICOM study arrives from the PACS, the system has two broad choices for how to get pixels from the hospital boundary into the inference engine:

**Option A — Synchronous REST:** PACS (or gateway) POSTs the study to an inference endpoint and waits for the urgency score in the HTTP response.

**Option B — Async queue:** Gateway publishes a message to Kafka; a consumer chain (pre-processor → inference → post-processor) processes the study and pushes the result to the RIS worklist.

This ADR records why we chose Option B.

---

## Decision

**Use an asynchronous Kafka-based pipeline.** The DICOM Gateway publishes to `dicom.raw`; the REST API (`/studies`, `/studies/{studyId}/result`) is a read-only interface for downstream consumers only, not the inference hot path.

---

## Exactly-Once Audit Semantics

Exactly-once guarantees are implemented via:

- Idempotent Kafka producers
- Transactional consumers
- Audit database uniqueness constraint on (study_id, model_version)
- Offset commit only after successful audit write


## Arguments for Synchronous REST (Option A)

1. Simpler operational model — fewer moving parts, no broker to manage.
2. PACS gets an immediate HTTP response confirming the score was received.
3. Standard HTTP error codes map naturally to PACS retry logic.
4. Easier to trace a single request end-to-end.

---

## Arguments for Async Queue (Option B) — Winning

### 1. PACS connection model

PACS systems send DICOM via C-STORE (a stateful DICOM association), not HTTP. The DICOM Gateway translates C-STORE to an internal event. Extending the gateway to also wait for a score and inject it back into the DICOM association as a C-FIND response would couple the gateway's transport layer to the inference latency — fragile and non-standard.

### 2. Burst absorption

100 studies/minute arrives non-uniformly (post-morning-rounds spike, shift-change spike). Kafka absorbs bursts without dropping requests. A synchronous REST endpoint with a 4-second timeout and a 100 RPS burst needs a connection pool of ≥ 400 slots per inference pod — a capacity headache. Kafka's consumer-lag metric gives us natural backpressure visibility.

### 3. Exactly-once audit semantics

Kafka transactions + Postgres `ON CONFLICT DO NOTHING` give exactly-once delivery for the audit record. With synchronous REST, a client retry on a 5xx could cause duplicate audit entries requiring deduplication logic at the application layer.

### 4. Replay and reprocessing

If we promote a new model version and need to re-score the last 24 hours of studies (e.g., for post-market surveillance), Kafka's offset reset allows trivial replay from `dicom.raw`. With synchronous REST, reprocessing requires a separate batch pipeline or re-sending from PACS.

### 5. Decoupled component lifecycle

Pre-processor, inference service, and post-processor can be deployed, updated, and scaled independently. A synchronous chain requires coordinated rolling deploys to avoid mid-request version mismatches.

---

## Trade-offs Accepted

| Cost | Mitigation |
|---|---|
| Kafka operational burden (ZooKeeper / KRaft, replication, monitoring) | Managed Kafka (MSK) eliminates broker management; team already operates MSK |
| Result is not synchronous — PACS must trust RIS push | All PACS in target hospital network support HL7 result push; validated in integration testing |
| Consumer lag can grow under extended GPU outage | `kafka_consumer_lag > 500` alert triggers on-call; GPU pod restarts auto-healed by K8s |
| Kafka adds ~30 ms round-trip latency | Well within 4 000 ms budget; negligible |

---

## Consequences

**Positive:**
- System is resilient to inference-layer restarts without PACS-side impact
- Replay capability supports regulatory re-scoring requirements
- Kafka lag is an early warning system for capacity problems

**Negative:**
- Team must maintain MSK cluster configuration and monitor consumer groups
- End-to-end tracing requires correlation IDs propagated through Kafka message headers (implemented via `X-Correlation-ID` header convention)
- Debugging latency requires correlating timestamps across three Kafka topics + Postgres

---

## Revisit Trigger

If the hospital network migrates to a FHIR-native PACS that natively supports long-polling or webhooks, a hybrid model (async internally, synchronous result callback externally) may be more appropriate. Re-evaluate at next major PACS upgrade cycle.
