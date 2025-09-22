# blueprint-spreadsheet-analyzer-base

Helm chart that deploys:
- vLLM OpenAI GPT-OSS-120B server
- vLLM Qwen2.5-VL-7B-Instruct server
- MinIO object storage

## Requirements
- Kubernetes 1.24+
- Helm 3.11+
- NVIDIA GPU nodes with `runtimeClassName: nvidia`
  - GPT-OSS: 4x GPUs (defaults) per pod
  - Qwen2.5-VL: 1x GPU (defaults) per pod
- A default `StorageClass` for the MinIO PVC (or set `minio.persistence.storageClassName`)

## Install
```bash
# GPU cluster (Kubernetes with NVIDIA runtime)
helm upgrade --install base /Users/ejackson/Workspace/voltage-park/blueprint-spreadsheet-analyzer-base \
  -n oss --create-namespace

# If you only want MinIO locally (no GPUs required), disable model deployments
helm upgrade --install base /Users/ejackson/Workspace/voltage-park/blueprint-spreadsheet-analyzer-base \
  -n oss --create-namespace \
  --set gptoss.replicaCount=0 \
  --set qwen25vl.replicaCount=0
```

Render without installing:
```bash
helm template base /Users/ejackson/Workspace/voltage-park/blueprint-spreadsheet-analyzer-base -n oss
```

Upgrade in place:
```bash
helm upgrade base /Users/ejackson/Workspace/voltage-park/blueprint-spreadsheet-analyzer-base -n oss
```

Uninstall:
```bash
helm uninstall base -n oss
```

## Services and Port-Forwarding
Both model services are `ClusterIP` by default and listen on port 8000 in-cluster.

```bash
# GPT-OSS API → http://localhost:8000
kubectl -n oss port-forward svc/base-blueprint-spreadsheet-analyzer-base-gptoss 8000:8000

# Qwen2.5-VL API → http://localhost:8002 (for a distinct local port)
kubectl -n oss port-forward svc/base-blueprint-spreadsheet-analyzer-base-qwen25vl 8002:8000

# MinIO API and Console → http://localhost:9000 and http://localhost:9001
kubectl -n oss port-forward svc/base-blueprint-spreadsheet-analyzer-base-minio 9000:9000 9001:9001
```

### Notes for local clusters (kind/minikube)
- Ensure a default `StorageClass` exists, or set `--set minio.persistence.storageClassName=standard` (value may differ).
- For GPU testing on a single node, you can scale down resources via `--set` flags, e.g. reduce CPU/memory and set `gptoss.resources.*` and `qwen25vl.resources.*` to fit your node.
- If your cluster has no GPUs, disable the model deployments as shown above and use MinIO only.

Note: Resource names are derived from the release name and chart name. To shorten names you can pass a `nameOverride`, e.g. `--set nameOverride=oss`.

## Configuration
Edit `values.yaml` or use `--set/--set-file/--set-string`.

- Namespace: `namespace: oss`
- Pod settings (shared): `pod.runtimeClassName`, `pod.terminationGracePeriodSeconds`
- GPT-OSS: `gptoss.*`
- Qwen2.5-VL: `qwen25vl.*`
- MinIO: `minio.*`

Example overrides:
```bash
# Change GPT GPU count and Qwen replicas
helm upgrade --install base ./ -n oss \
  --set gptoss.resources.requests.gpu="8" \
  --set gptoss.resources.limits.gpu="8" \
  --set qwen25vl.replicaCount=2

# Set MinIO credentials (use --set-string to preserve special chars)
helm upgrade --install base ./ -n oss \
  --set-string minio.rootUser=myuser \
  --set-string minio.rootPassword='S3cr3t-P@ssw0rd'
```

## Notes
- Default GPT `maxModelLen` and `maxNumBatchedTokens` are opinionated and large; adjust for your capacity.
- Ensure your nodes provide sufficient GPU/CPU/memory to meet the default requests/limits.
- MinIO PVC defaults to 500Gi ReadWriteOnce. Adjust `minio.persistence.size` and optionally `storageClassName`.
