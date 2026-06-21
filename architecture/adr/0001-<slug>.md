# ADR 0001 — GPU vs CPU Serving for Inference

**Status:** Accepted  
**Date:** 2026-06-07  
**Deciders:** ML Platform Lead, Radiology Product Lead, Infrastructure Lead  

---

## Context

EfficientNet-V2-M with a custom classification head has ~28 million parameters. In FP32, a single forward pass on a 512×512×3 tensor (single slice) takes approximately:

- **CPU (c6i.4xlarge, 16 vCPU):** 380–520 ms per slice
- **GPU A10G (TRT FP16):** 8–14 ms per slice

A multi-slice study (average 12 slices, worst-case 48 slices) requires running all slices through the model. Slice-level parallelism within one study is possible on GPU but not practical on CPU without many more instances.

The system must serve 100 studies/minute at peak with p95 ≤ 4 000 ms end-to-end. Pre-processing and routing consume roughly 700 ms of the budget, leaving ~3 300 ms for inference.

---

## Decision

**Use NVIDIA A10G GPU nodes (2 instances, active–active) as the primary inference backend, with TensorRT FP16 optimization.**

Model artifact is **mounted at runtime** from the MLflow Registry (not baked into the Docker image) — see [container/README.md](../../container/README.md) for rationale.

---

## Rationale

### GPU path math

| Metric | Value |
|---|---|
| A10G TRT FP16 latency per slice | ~10 ms (p50) |
| Max slices per study | 48 |
| Max single-study inference time | 48 × 10 ms = 480 ms |
| Triton dynamic batching (up to 8 concurrent) | Amortizes GPU memory bandwidth |
| p95 inference budget consumed | ~1 400 ms (leaving 1 900 ms margin to 4 000 ms) |

### CPU path math (rejected for primary)

| Metric | Value |
|---|---|
| CPU latency per slice (c6i.4xlarge) | ~450 ms |
| Max single-study (48 slices, sequential) | 21 600 ms — **exceeds entire budget** |
| With 16 parallel slice threads | ~1 350 ms per study — **marginal, no burst headroom** |
| Instances needed to match 100 studies/min | ≥ 14 c6i.4xlarge — 2.3× more expensive than 2× A10G |

CPU cannot meet the p95 = 4 000 ms requirement for worst-case 48-slice studies without an economically impractical number of instances.

### GPU is also cheaper at this scale

2× g5.2xlarge (A10G) ≈ $2.16/hr on-demand = ~$1 555/month  
14× c6i.4xlarge ≈ $2.72/hr total = ~$1 958/month  
GPU is cheaper **and** faster at peak load.

---

## CPU Fallback

A CPU-only fallback path is maintained for:
- Regulatory testing in environments without GPU
- Smoke tests in CI (no GPU available)
- Emergency degraded mode: CPU mode triggers a `LatencyBudgetDegraded` alert (p95 expected to reach 6 000–8 000 ms); radiologists are notified that automated triage is operating in reduced-accuracy / reduced-speed mode.

CPU fallback uses ONNX Runtime (not TRT) and serves only studies with ≤ 12 slices to stay within an extended 10 000 ms budget. Studies > 12 slices are queued until GPU capacity is restored.

---

## Consequences

**Positive:**
- p95 inference comfortably within budget even for worst-case studies
- GPU enables future model scale-up (larger architectures) without architectural change
- TRT FP16 reduces GPU memory footprint, allowing 2 concurrent model versions on one card during canary deployment

**Negative:**
- GPU node pool requires dedicated node group in Kubernetes (cannot mix with CPU workloads efficiently)
- A10G availability in some regions may require capacity reservations
- TRT engine build is environment-specific; engine must be rebuilt if GPU driver or CUDA version changes (handled in CI pipeline)

---

## Revisit Trigger

If average study size drops below 8 slices due to workflow changes, or if new-generation CPU inference (e.g., AMX instructions on Sapphire Rapids) closes the gap, revisit this decision. Re-evaluate annually or when hardware generation changes.
