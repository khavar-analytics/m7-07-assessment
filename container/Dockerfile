# ============================================================
# Radiology Triage — Inference Service Container
# Base: NVIDIA Triton Inference Server (TRT + ONNX backends)
# Model artifact: MOUNTED at runtime from MLflow (not baked in)
# See container/README.md for full rationale and size estimates.
# ============================================================

# ── Stage 1: Python dependency builder ──────────────────────
FROM python:3.11-slim-bookworm AS py-builder

WORKDIR /build

# Install build tools needed for native Python extensions
RUN apt-get update && apt-get install -y --no-install-recommends \
        gcc \
        libgomp1 \
        curl \
    && rm -rf /var/lib/apt/lists/*

# Copy and install Python deps into an isolated prefix
COPY requirements.txt .
RUN pip install --prefix=/install --no-cache-dir -r requirements.txt


# ── Stage 2: Triton Inference Server base ───────────────────
# nvcr.io/nvidia/tritonserver:24.04-py3 includes:
#   - TensorRT 8.6 / CUDA 12.4
#   - ONNX Runtime 1.17 (GPU)
#   - Python backend
#   - Triton HTTP/gRPC endpoints
FROM nvcr.io/nvidia/tritonserver:24.04-py3 AS triton-base

LABEL org.opencontainers.image.title="radiology-triage-inference"
LABEL org.opencontainers.image.description="EfficientNet-V2-M triage inference service"
LABEL org.opencontainers.image.vendor="Radiology Triage Platform"
# Version label is injected by CI at build time via --build-arg
ARG MODEL_VERSION=unset
ARG GIT_SHA=unset
LABEL org.opencontainers.image.version="${MODEL_VERSION}"
LABEL org.opencontainers.image.revision="${GIT_SHA}"

# ── Stage 3: Application layer ──────────────────────────────
FROM triton-base AS app

# Copy Python packages from builder
COPY --from=py-builder /install /usr/local

# Application source: post-processor, health-check, Kafka consumer
WORKDIR /app
COPY src/ ./src/
COPY triton_config/ ./triton_config/

# ── Model mount point ────────────────────────────────────────
# The TRT engine and ONNX model are NOT baked into this image.
# They are pulled at container startup from MLflow artifact store
# by the entrypoint script, which:
#   1. Reads MODEL_VERSION env var (set by K8s deployment manifest)
#   2. Downloads model.plan + config.pbtxt from S3 to /model-store/
#   3. Starts tritonserver pointing at /model-store/
#
# This design allows model version updates without rebuilding or
# re-pushing the ~8 GB Triton base image.
# See container/README.md §Bake vs Mount for full rationale.
RUN mkdir -p /model-store /model-store/triage/1

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

# ── Non-root user ────────────────────────────────────────────
# Triton requires running as root for some GPU operations;
# post-processor sidecar runs as non-root via separate deployment.
# This container runs as root (Triton requirement) with read-only FS
# where possible.

# ── Health check ─────────────────────────────────────────────
# Triton exposes /v2/health/ready on port 8000
HEALTHCHECK --interval=10s --timeout=5s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8000/v2/health/ready || exit 1

# Ports: 8000 HTTP, 8001 gRPC, 8002 metrics (Prometheus)
EXPOSE 8000 8001 8002

ENTRYPOINT ["/entrypoint.sh"]

# Default CMD — overridden by K8s deployment if needed
CMD ["tritonserver", \
     "--model-repository=/model-store", \
     "--model-control-mode=explicit", \
     "--load-model=triage", \
     "--allow-metrics=true", \
     "--metrics-port=8002", \
     "--http-port=8000", \
     "--grpc-port=8001", \
     "--log-verbose=0"]
