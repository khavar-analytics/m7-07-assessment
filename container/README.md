# Container Image Plan

## Base Image

`nvcr.io/nvidia/tritonserver:24.04-py3`

- CUDA 12.4 / TensorRT 8.6 / cuDNN 8.9
- ONNX Runtime 1.17 (GPU build)
- Python 3.11 backend
- Source: NVIDIA NGC Registry (requires free NGC account; no licensing cost)

## Bake vs Mount Decision

**Decision: Model artifacts are MOUNTED at runtime, not baked into the image.**

### Rationale

| Factor | Bake in Image | Mount at Runtime | Winner |
|---|---|---|---|
| Image size | +110 MB (TRT engine) per model version | No change | Mount |
| Model update cost | Full image rebuild + push (~8 GB base) | Update S3 + K8s env var | Mount |
| Rollback speed | Pull previous image tag | Update env var, restart pod | Mount |
| Audit: model-image coupling | Tight (good for audit trail) | Loose (registry provides coupling) | Bake |
| Cold-start latency | Zero (model already in layer cache) | +15–30 s (S3 download on startup) | Bake |
| CI rebuild required for model swap? | Yes | No | Mount |

The dominant factors are **model update velocity** and **rollback speed**. In a regulated environment, we promote model versions frequently (shadow scoring, canary deploys). Requiring an 8 GB image push for every model update would make the 24-hour canary soak impractical. The cold-start penalty (15–30 s) is acceptable because:
1. Triton pods are not scaled to zero — the node is always warm
2. K8s `startupProbe` delays traffic until Triton's `/v2/health/ready` returns 200
3. The startup probe `start_period` is set to 60 s in the Dockerfile

**Audit coupling** is maintained by the model registry: every deployed pod's `MODEL_VERSION` env var matches the registry Production tag, and the `X-Model-Version` header in every API response returns this value. The registry provides the audit linkage independent of image baking.

### How the Mount Works

```bash
# Executed by /entrypoint.sh at container startup:
aws s3 cp \
  s3://radiology-triage-mlflow-artifacts-us-east-1/efficientnet-v2m-triage/${MODEL_VERSION}/model_trt_fp16_a10g.plan \
  /model-store/triage/1/model.plan

aws s3 cp \
  s3://radiology-triage-mlflow-artifacts-us-east-1/efficientnet-v2m-triage/${MODEL_VERSION}/config.pbtxt \
  /model-store/triage/config.pbtxt

exec tritonserver ...
```

`MODEL_VERSION` is injected via the K8s Deployment manifest and corresponds to the registry version tag (e.g., `efficientnet-v2m-triage-v1.4.2`). This tag is identical to the image tag set by the CI pipeline.

## Image Size Estimate

| Layer | Size |
|---|---|
| Triton base (`24.04-py3`) | ~7.8 GB |
| Python dependencies (pydicom, pyarrow, kafka-python, boto3, prometheus-client) | ~120 MB |
| Application source (`src/`, `triton_config/`, entrypoint) | ~2 MB |
| **Total image size** | **~7.95 GB** |
| **Model artifact (mounted, not in image)** | 110 MB (TRT FP16) |

The large base image size is driven by CUDA/TensorRT libraries — this is unavoidable for GPU inference. The image is cached in the ECR pull-through cache within the VPC; after the first pull, layer caching means pod restarts pull only changed layers.

## Registry

- **ECR Repository:** `123456789.dkr.ecr.us-east-1.amazonaws.com/radiology-triage/inference`
- **Tag scheme:** `{model-family}-v{semver}` — matches the model registry scheme exactly.
  - Examples: `efficientnet-v2m-triage-v1.4.2`, `efficientnet-v2m-triage-v1.5.0-rc1`
  - `latest` tag is **not used** in production deployments to ensure reproducibility.
- **Immutability:** ECR tag immutability is enabled — pushed tags cannot be overwritten.
- **Scanning:** ECR Enhanced Scanning (Inspector) runs on every push; critical CVEs block deployment in CI.
- **Retention:** Images retained for 24 months; Archived model versions retain their images indefinitely.

## Build Context

```
container/
├── Dockerfile
├── README.md
├── entrypoint.sh          # startup script: S3 pull + tritonserver exec
├── requirements.txt       # Python deps for post-processor sidecar
└── triton_config/
    └── triage/
        └── config.pbtxt   # Triton model config (backend, input/output shapes, batching)
```

## Security Notes

- Image runs as root (Triton GPU requirement for NVML access). Sidecar post-processor pod runs as UID 1000.
- `--read-only` filesystem enabled except for `/model-store` (tmpfs) and `/tmp`.
- Secrets (AWS credentials for S3 model pull) are injected via IRSA (IAM Roles for Service Accounts) — no long-lived credentials in the image or environment variables.
- SBOM generated at build time by `syft` and attached to the image manifest.
