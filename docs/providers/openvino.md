# OpenVINO Model Server

OpenVINO Model Server (OVMS) is a high-performance inference server developed by Intel. It exposes an OpenAI-compatible API at `/v3` and is optimized for Intel CPUs, GPUs, and VPUs.

---

## When to use OVMS

- You are running on Intel hardware (CPU, GPU, or VPU)
- You need high-throughput inference for production workloads
- You want to serve models with Intel optimization without managing a custom server
- You require a containerized or Kubernetes-deployable inference endpoint

---

## Install OVMS

### Docker (recommended)

```bash
# Pull the latest OVMS image
docker pull openvino/model-server:latest

# Start the server with a model
docker run -d --rm -p 8000:8000 \
  openvino/model-server:latest \
  --model_name my_model \
  --model_path /models/my_model \
  --port 8000
```

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ovms
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ovms
  template:
    metadata:
      labels:
        app: ovms
    spec:
      containers:
        - name: ovms
          image: openvino/model-server:latest
          args: ["--model_name", "my_model", "--model_path", "/models/my_model", "--port", "8000"]
          ports:
            - containerPort: 8000
          volumeMounts:
            - name: model-store
              mountPath: /models
      volumes:
        - name: model-store
          persistentVolumeClaim:
            claimName: model-pvc
```

### Install models

Download models from Hugging Face or convert them to OpenVINO format:

```bash
# Install OpenVINO model converter
pip install optimum[openvino]

# Convert a model to OpenVINO format
optimum-cli export openvino \
  --model meta-llama/Llama-3.2-1B-Instruct \
  --task text-generation-with-past \
  ./ovmmodels/llama-3.2-1b
```

---

## Nanobot configuration

```json
{
  "agents": {
    "defaults": {
      "model": "llama-3.2-1b",
      "provider": "ovms"
    }
  },
  "providers": {
    "ovms": {
      "api_key": "EMPTY",
      "api_base": "http://localhost:8000/v3"
    }
  }
}
```

**Remote OVMS server:**
```json
{
  "providers": {
    "ovms": {
      "api_key": "EMPTY",
      "api_base": "http://ovms.internal.company.com:8000/v3"
    }
  }
}
```

> **API key:** OVMS does not require authentication by default. Set `api_key` to any non-empty string (e.g., `"EMPTY"`).

---

## Model format

OVMS expects models in its native format. Use `optimum-intel` to convert Hugging Face models:

| Format | Tool | Notes |
|--------|------|-------|
| OpenVINO IR (`.xml` + `.bin`) | `optimum-cli export openvino` | Native OVMS format |
| ONNX (`.onnx`) | `optimum-cli export onnx` | Alternative format |

---

## Detection logic

Nanobot recognizes OVMS when:
1. `provider` is set to `"ovms"`
2. Model names include `openvino` or `ovms`
3. `api_base` points to an OVMS endpoint (typically port 8000)

OVMS uses the OpenAI-compatible `/v3` endpoint.

---

## Further reading

- Provider overview: [providers/index.md](./index.md)
- Other local providers: [providers/local.md](./local.md)
- Official docs:
  - OpenVINO Model Server (docs.openvino.ai)
  - optimum-intel (huggingface.co/docs/optimum-intel)
