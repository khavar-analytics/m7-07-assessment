# Rollback Runbook — Radiology Triage Inference Service

**Owner:** ML Platform On-Call  
**Last reviewed:** 2026-06-07  
**Audience:** On-call engineers, ML engineers with `radiology-triage-deploy` access  

---

## Overview

This runbook covers rollback procedures for the Radiology Triage Inference Service. A rollback reverts the serving model to the previous Production version in the MLflow registry. It does **not** roll back the application container (handled separately via standard K8s rollout undo).

All rollback trigger thresholds listed here correspond directly to alert definitions in `monitoring/alerts.yaml`. Do not change thresholds here without updating `monitoring/alerts.yaml` and `serving/slos.yaml`.

---

## Automatic Rollback Triggers

The following conditions must trigger an immediate rollback without waiting for manual investigation (unless the on-call engineer has a specific reason to hold):

| Condition | Alert | Threshold | Source |
|---|---|---|---|
| p95 e2e latency | `E2ELatencySLOBreach` | > 4 000 ms for ≥ 3 min | serving/slos.yaml SLO-LAT-01 |
| Inference error rate | `InferenceErrorRateCritical` | > 5% for ≥ 2 min | serving/slos.yaml SLO-ERR-01 |
| Model version mismatch | `ModelVersionMismatch` | Any mismatch for ≥ 2 min | serving/slos.yaml SLO-MOD-01 |
| Audit completeness loss | `AuditRecordMissing` | < 99.99% for ≥ 1 min | serving/slos.yaml SLO-AUD-01 |
| All GPU pods down | `InferencePodDown` | 0 available pods for ≥ 1 min | serving/slos.yaml SLO-AVL-02 |
| Service availability drop | `ServiceAvailabilityDrop` | < 99% for ≥ 5 min | serving/slos.yaml SLO-AVL-01 |
| Kafka queue delay | `KafkaQueueDelayHigh` | > 2s p95 for ≥ 3 min | serving/slos.yaml SLO-QUE-01 |

**Canary-specific triggers** (roll back canary only, do not affect production stable):

| Condition | Alert | Threshold |
|---|---|---|
| Canary p95 latency | `CanaryLatencyBreach` | > 4 000 ms for ≥ 5 min |
| Canary error rate | `CanaryErrorRateHigh` | > 2% for ≥ 3 min |

---

## Pre-Rollback Checklist

Before executing rollback, confirm:

- [ ] Which alert fired? (Note the alert name — it maps to the trigger table above)
- [ ] What is the current Production model version? (`kubectl get deployment triton-inference-stable -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="MODEL_VERSION")].value}'`)
- [ ] What was the previous Production model version? (Check MLflow registry Archived stage — most recent archive is the previous version)
- [ ] Is this a canary rollback (canary pod only) or a full production rollback?
- [ ] Is an audit incident active? (`AuditRecordMissing` alert) — if yes, escalate to compliance officer simultaneously

Estimated rollback time: **5–8 minutes** (model mount from S3 + pod readiness)

---

## Procedure A: Canary Rollback

Use when only the canary pod is affected (`CanaryLatencyBreach` or `CanaryErrorRateHigh`).

```bash
# 1. Route all traffic back to stable (0% canary)
kubectl apply -f k8s/virtualservice-stable-100pct.yaml

# 2. Scale down canary deployment
kubectl scale deployment triton-inference-canary --replicas=0 \
  --namespace radiology-triage

# 3. Verify stable is serving 100% of traffic
kubectl get virtualservice radiology-triage-vs \
  --namespace radiology-triage -o yaml | grep -A5 weight

# 4. Confirm latency and error rate return to normal (< 3500 ms p95, < 1% errors)
# Watch Grafana: https://grafana.internal/d/radiology-triage-canary
# Expected recovery: < 2 minutes after traffic shift

# 5. Mark canary deployment as failed in CI/CD
# Update the GitHub deployment environment status via API or re-run workflow
```

After canary rollback, update the MLflow model version from `Staging` to `Archived` with reason:
```bash
python scripts/ops/archive_model_version.py \
  --version "<canary-version>" \
  --reason "Canary rollback: <alert_name> fired at <timestamp>"
```

---

## Procedure B: Full Production Rollback

Use when the stable production deployment is causing SLO breaches.

### Step 1: Identify the rollback target version

```bash
# List archived versions (most recent = previous production)
python scripts/ops/list_registry_versions.py --stage Archived --limit 5
```

The rollback target is the most recent version with `stage: Archived` and a non-empty `promoted_to_production_at` timestamp.

### Step 2: Execute rollback

```bash
# Set the previous version (replace with actual version tag)
ROLLBACK_VERSION="efficientnet-v2m-triage-v1.3.1"

# Update stable deployment to previous version
kubectl set env deployment/triton-inference-stable \
  MODEL_VERSION=${ROLLBACK_VERSION} \
  --namespace radiology-triage

kubectl set image deployment/triton-inference-stable \
  triton=123456789.dkr.ecr.us-east-1.amazonaws.com/radiology-triage/inference:${ROLLBACK_VERSION} \
  --namespace radiology-triage

# Wait for rollout
kubectl rollout status deployment/triton-inference-stable \
  --namespace radiology-triage \
  --timeout=180s
```

### Step 3: Verify rollback health

```bash
# 1. Check that Triton health endpoint returns 200
curl -f https://api.radiology-triage.internal/v1/health

# 2. Verify X-Model-Version header matches rollback target
# (This is the same check as the CI smoke test and SLO-MOD-01 / ModelVersionMismatch alert)
curl -s -I https://api.radiology-triage.internal/v1/model/info \
  | grep X-Model-Version
# Expected: X-Model-Version: efficientnet-v2m-triage-v1.3.1

# 3. Confirm ModelVersionMismatch alert clears in Alertmanager
# Expected: resolves within 60 seconds

# 4. Confirm E2ELatencySLOBreach alert clears
# Expected: p95 latency returns to < 3500 ms within 5 minutes
# (p95 normalises once Triton has processed ~50 studies on the new pod)

# 5. Confirm InferenceErrorRateCritical alert clears
# Expected: error rate drops below 1% within 3 minutes

# 6. Confirm audit completeness restored (if AuditRecordMissing was active)
# Query: SELECT COUNT(*) FROM audit_predictions WHERE processed_at > NOW() - INTERVAL '10 minutes'
```

### Step 4: Update registry

```bash
# Mark the failed version as Archived
python scripts/ops/archive_model_version.py \
  --version "<failed-version>" \
  --reason "Production rollback: <alert_name> fired at <timestamp>. Incident: <INC-XXXX>"

# The previous version (now serving) stays as Archived — do not re-promote it
# to Production automatically. A new promotion cycle is required.
```

### Step 5: Notify stakeholders

- [ ] Post to `#ml-incidents` Slack channel: model version rolled back, reason, ETA for root-cause analysis
- [ ] If `AuditRecordMissing` was active: notify compliance officer (email template: `templates/compliance-incident-notification.md`)
- [ ] Update PagerDuty incident with rollback outcome
- [ ] Create post-incident review task (due within 48 hours)

---

## Procedure C: GPU Pod Recovery (No Rollback)

Use when `InferencePodDown`, `LatencyBudgetDegraded`, or `KafkaQueueDelayHigh` is caused by infrastructure bottlenecks (not model issues)

```bash
# Check pod status
kubectl get pods -n radiology-triage -l app=triton-inference

# Check events for OOM or eviction
kubectl describe pod <pod-name> -n radiology-triage

# Force pod restart (K8s will restart automatically via liveness probe,
# but this forces immediate restart if probe hasn't triggered)
kubectl delete pod <pod-name> -n radiology-triage

# Monitor recovery
kubectl rollout status deployment/triton-inference-stable \
  --namespace radiology-triage \
  --timeout=120s

# Expected recovery: pod restarts, downloads model from S3 (~15-30 s),
# then Triton /v2/health/ready returns 200. Total: < 2 minutes.
```

---

## Rollback Decision Tree

```
Alert fires
    │
    ├─ Is it a canary alert (CanaryLatencyBreach / CanaryErrorRateHigh)?
    │   └─ YES → Procedure A: Canary Rollback
    │
    ├─ Is it ModelVersionMismatch?
    │   └─ YES → Check if caused by mis-deploy or pod restart with wrong env var
    │            If pod: restart pod with correct MODEL_VERSION
    │            If deploy: Procedure B: Full Production Rollback
    │
    ├─ Is it AuditRecordMissing?
    │   └─ YES → Escalate to compliance officer FIRST, then Procedure B
    │
    ├─ Is it InferencePodDown with model version correct?
    │   └─ YES → Procedure C: GPU Pod Recovery
    │
    ├─ Is it KafkaQueueDelayHigh?
    │   └─ YES → Check Kafka consumer lag and GPU utilization
    │            If sustained backlog → Procedure B: Full Production Rollback
    │            If infrastructure bottleneck → Procedure C: GPU Pod Recovery
    └─ Is it E2ELatencySLOBreach or InferenceErrorRateCritical?
        └─ Check: did a new model version deploy in the last 24h?
            YES → Procedure B: Full Production Rollback
            NO  → Investigate infrastructure (GPU OOM, Kafka lag, network)
                  Procedure C if GPU-related
```

---

## Post-Rollback Validation Checklist

After any rollback is complete, verify all of the following before closing the incident:

- [ ] `E2ELatencySLOBreach` alert cleared
- [ ] `InferenceErrorRateCritical` alert cleared
- [ ] `ModelVersionMismatch` alert cleared (X-Model-Version header correct)
- [ ] `AuditRecordMissing` alert cleared (if it was active)
- [ ] Kafka consumer lag on `dicom.ready` < 100 (normal)
- [ ] Radiologist worklist is receiving urgency scores again
- [ ] Post-incident review scheduled

---

## Contact Escalation

| Situation | Contact |
|---|---|
| Standard rollback | ML Platform On-Call (PagerDuty) |
| Audit completeness breach | Compliance Officer + Engineering Lead |
| GPU hardware failure | AWS Support (if EC2 hardware) + Platform Lead |
| Regulatory incident | CISO + Legal + Compliance Officer |
| > 30 min downtime | VP Engineering + Clinical Champion |
