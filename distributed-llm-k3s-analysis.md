# Distributed LLM Inference on k3s: 4× Raspberry Pi 4 (arm64, 8 GB RAM)

## Context

This document covers approaches for distributing a Qwen-family LLM across a 4-node k3s cluster of Raspberry Pi 4 boards, each with 8 GB RAM. The existing codebase already runs `llama.cpp` single-node inference via a custom Helm chart, with Open WebUI aggregating multiple model endpoints via `OPENAI_API_BASE_URLS`. The goal is to evaluate whether and how to pool the four nodes for a single larger inference workload.

---

## Core Concept: Two Distinct Parallelism Strategies

### Tensor parallelism (distributed inference)

A single request is split across multiple nodes. Each node holds a subset of the model's layers or tensors. The leader node tokenises the prompt and orchestrates the forward pass; workers execute their assigned layers and return activations over the network.

- **Tools:** llama.cpp RPC, distributed-llama (dllama)
- **Effect:** Reduces per-node memory requirement; does not improve single-request latency on GbE — it worsens it due to network round-trips per token
- **Best for:** Models too large to fit on one node

### Replica parallelism (load balancing)

Each node runs a full copy of the model independently. A load balancer (Open WebUI, Traefik, or a dedicated proxy) routes concurrent requests across nodes.

- **Tools:** Multiple `llama-server` instances + Open WebUI `OPENAI_API_BASE_URLS`
- **Effect:** Scales concurrent throughput linearly; per-request latency is unchanged
- **Best for:** Models that fit on one node; multi-user homelab workloads

---

## Approach 1: llama.cpp RPC (tensor parallelism)

### How it works

The `rpc-server` binary (already built in the existing `Dockerfile`) exposes a node's CPU memory as a remote GGML backend over TCP (default port `50052`). The `llama-server` leader connects to one or more `rpc-server` workers at startup via `--rpc <host>:<port>` flags. The leader distributes model weights and KV cache across all devices proportionally to available memory.

```
llama-server --model /models/model.gguf \
  --rpc worker1:50052 \
  --rpc worker2:50052 \
  --rpc worker3:50052
```

### Kubernetes integration: LeaderWorkerSet (LWS)

The upstream LWS project (sigs.k8s.io/lws) provides a Kubernetes controller purpose-built for this topology:

- Deploys a leader pod and N worker pods as a named group
- Leader gets a stable ClusterIP Service; workers get stable DNS via a headless StatefulSet Service
- Workers start first; leader starts after workers are ready

LWS is the recommended Kubernetes-native approach for llama.cpp RPC. It requires installing the LWS CRD and controller into the cluster before deploying.

### Extending the existing Helm chart

The existing `llama-cpp` chart already builds `rpc-server`. To add RPC support, extend `values.yaml` with an `rpc:` block:

```yaml
# values-qwen14b-distributed.yaml
fullnameOverride: llama-qwen14b

model:
  path: /models/Qwen2.5-14B-Instruct-Q4_K_M.gguf
  name: "Large (Qwen2.5-14B)"
  contextSize: 4096

rpc:
  enabled: true
  workers:
    - host: pi4-node02.home.mowntan.com
      port: 50052
      mem: 7000   # MB to advertise to the leader
    - host: pi4-node03.home.mowntan.com
      port: 50052
      mem: 7000
    - host: pi4-node04.home.mowntan.com
      port: 50052
      mem: 7000

nodeSelector:
  kubernetes.io/hostname: pi4-node01.home.mowntan.com

persistence:
  models:
    nfs:
      server: synology-nas01.home.mowntan.com
      path: /volume2/downloads/models
```

In `deployment.yaml`, extend the leader's command block:

```yaml
command:
  - sh
  - -c
  - |
    set -- --host 0.0.0.0 --port 8080 \
           --model {{ .Values.model.path }} \
           --ctx-size {{ .Values.model.contextSize }}
    {{- if .Values.model.name }}
    set -- "$@" --alias {{ .Values.model.name | quote }}
    {{- end }}
    {{- if .Values.rpc.enabled }}
    {{- range .Values.rpc.workers }}
    set -- "$@" --rpc {{ .host }}:{{ .port }}
    {{- end }}
    {{- end }}
    exec llama-server "$@"
```

The RPC workers themselves need a separate Deployment or DaemonSet running `llama-rpc-server`:

```yaml
command:
  - llama-rpc-server
  - --host
  - "0.0.0.0"
  - --port
  - "50052"
  - --mem
  - "7000"
```

### Security note

The RPC protocol has no authentication. Workers must only be reachable from within the cluster network. Do not expose port `50052` via Ingress or NodePort to external networks.

### Tradeoffs

| Factor | Assessment |
|---|---|
| Single-request latency | Worse than single-node — network round-trips per token |
| Max model size | Pooled: ~28 GB across 4× 8 GB nodes |
| Concurrent throughput | 1 request at a time (serial per leader) |
| Kubernetes complexity | Medium — LWS CRD, or manual StatefulSet |
| Fits existing chart | Yes — incremental extension |

---

## Approach 2: distributed-llama (dllama)

### How it works

dllama is a purpose-built distributed inference binary. Workers each hold a contiguous shard of the model's layers. The root node holds the tokeniser and embedding layer; each worker processes its assigned transformer layers and passes activations to the next worker over TCP.

- Supports Qwen 3 0.6B, 1.7B, 8B, 14B natively
- Has explicit Raspberry Pi support and documentation
- Exposes an OpenAI-compatible API via `dllama-api`

```bash
# On each worker node:
dllama worker --port 9999 --nthreads 4

# On the root node:
dllama-api --model /models/qwen3_8b_q40 \
  --tokenizer /models/qwen3_8b_q40/tokenizer.bin \
  --workers 192.168.1.2:9999 192.168.1.3:9999 192.168.1.4:9999 \
  --nthreads 4
```

### Kubernetes integration

No upstream Helm chart exists. You would need to build:

1. A Docker image for `dllama` (arm64) — compile from source in a multi-stage Dockerfile
2. A DaemonSet for worker pods (one per non-root node), pinned by `nodeSelector`
3. A Deployment for the root/API pod, pinned to the root node
4. A headless Service for worker-to-worker DNS, and a ClusterIP Service for the API

### Tradeoffs

| Factor | Assessment |
|---|---|
| Single-request latency | Worse than single-node |
| Max model size | Pooled across all nodes |
| Concurrent throughput | 1 request at a time |
| Kubernetes complexity | High — no existing chart, requires custom build |
| Fits existing chart | No — greenfield work |

---

## Approach 3: Replica parallelism (recommended for current setup)

### How it works

This is exactly what the existing chart already does. Each node runs a complete copy of `llama-server` with a full model loaded. Open WebUI's `OPENAI_API_BASE_URLS` semicolon-delimited list round-robins requests across all four endpoints.

```yaml
# Already in values-homelab.yaml
env:
  - name: OPENAI_API_BASE_URLS
    value: "http://llama-coder:8080/v1;http://llama-mistral:8080/v1;http://llama-llama:8080/v1;http://llama-phi:8080/v1"
```

To run the same model on multiple nodes for higher concurrency, deploy it with multiple `values-*.yaml` files, each pinned to a different node via `nodeSelector`.

### Memory headroom per Pi4 (8 GB)

| Model | GGUF size (Q4_K_M) | Headroom for KV cache + OS |
|---|---|---|
| Qwen2.5-7B-Instruct | ~4.5 GB | ~3.5 GB |
| Llama-3.2-3B-Instruct | ~2.0 GB | ~6.0 GB |
| Phi-4-mini | ~2.5 GB | ~5.5 GB |
| Qwen2.5-14B-Instruct | ~8.5 GB | does not fit alone |

### Tradeoffs

| Factor | Assessment |
|---|---|
| Single-request latency | Best — no network overhead |
| Max model size | ~6–7 GB per node (leaves OS headroom) |
| Concurrent throughput | Scales linearly with node count |
| Kubernetes complexity | Low — already implemented |
| Fits existing chart | Yes — zero changes required |

---

## Recommended Architecture

### For models ≤ 7B (current workloads)

Keep the existing replica approach. Each Pi4 runs one model. Open WebUI load-balances across all four. This gives 4× concurrent throughput with no extra complexity.

### For a 14B model (future extension)

Use llama.cpp RPC with the following topology:

```
pi4-node01  →  llama-server (leader, API on :8080)
               --rpc pi4-node02:50052
               --rpc pi4-node03:50052
               --rpc pi4-node04:50052

pi4-node02  →  llama-rpc-server (worker, :50052)
pi4-node03  →  llama-rpc-server (worker, :50052)
pi4-node04  →  llama-rpc-server (worker, :50052)
```

Total pooled memory: ~28 GB, sufficient for Qwen2.5-14B-Instruct-Q4_K_M (~8.5 GB) with KV cache spread across nodes.

Add the new endpoint to Open WebUI's `OPENAI_API_BASE_URLS` alongside the existing smaller models.

---

## Implementation Checklist

### Phase 1: extend the Helm chart for RPC

- [ ] Add `rpc.enabled`, `rpc.workers[]` to `chart/values.yaml`
- [ ] Add conditional `--rpc` flags to the `llama-server` command in `deployment.yaml`
- [ ] Add a second container or separate Deployment template for `llama-rpc-server` workers
- [ ] Add a headless Service for the RPC worker pods (port 50052)
- [ ] Ensure RPC port is not exposed via Ingress

### Phase 2: model download

Add to `download-models-job.yaml`:

```yaml
download bartowski/Qwen2.5-14B-Instruct-GGUF Qwen2.5-14B-Instruct-Q4_K_M.gguf
```

### Phase 3: deploy and test

```bash
# Deploy RPC workers first (they must be ready before the leader)
helm upgrade --install llama-qwen14b-workers ./apps/llama-cpp/chart \
  -f apps/llama-cpp/values/values-qwen14b-workers.yaml \
  -n ai-lab

# Then deploy the leader
helm upgrade --install llama-qwen14b ./apps/llama-cpp/chart \
  -f apps/llama-cpp/values/values-qwen14b-distributed.yaml \
  -n ai-lab

# Smoke test
curl http://llama-qwen14b.ai-lab.svc:8080/v1/models
```

---

## References

- llama.cpp RPC documentation: `tools/rpc/README.md` in the llama.cpp repository
- LeaderWorkerSet examples: `lws.sigs.k8s.io/docs/examples/llamacpp`
- distributed-llama: `github.com/b4rtaz/distributed-llama`
- Arm Learning Path (distributed inference on Graviton): `learn.arm.com/learning-paths/servers-and-cloud-computing/distributed-inference-with-llama-cpp`
