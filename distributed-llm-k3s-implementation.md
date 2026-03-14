# Distributed LLM on k3s: Implementation Plan

## Goal

Restore the llama-cpp Helm chart and extend it with RPC-based tensor parallelism so the 4 inference Pi4 nodes (8 GB each) can pool memory for a single 14B model — while keeping the existing per-node replica approach for smaller models. Expose all endpoints through Open WebUI at `chat.apps.mowntan.com`.

---

## Cluster Inventory

| Node | Role | Purpose |
|---|---|---|
| pi4-node01 | inference | RPC leader (llama-server + model) |
| pi4-node02 | inference | RPC worker |
| pi4-node03 | inference | RPC worker |
| pi4-node04 | inference | RPC worker |
| synology-nas01 | NFS | Model storage at `/volume2/downloads/models` |

---

## Phase 1: Restore the llama-cpp Chart

Restore the chart from git history (commit before `a1bc026`) into `apps/llama-cpp/`. The chart was previously working with 4 single-node deployments.

### Files to restore

```
apps/llama-cpp/
├── chart/
│   ├── Chart.yaml              # v2.0.0, appVersion b8148
│   ├── values.yaml
│   └── templates/
│       ├── _helpers.tpl
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ingress.yaml
├── docker/
│   └── Dockerfile              # Already builds llama-server + llama-rpc-server
├── values/
│   ├── values-coder.yaml       # Qwen2.5-Coder-7B → pi4-node01
│   ├── values-llama.yaml       # Llama-3.2-3B → pi4-node03
│   ├── values-mistral.yaml     # Mistral-7B → pi4-node02
│   └── values-phi.yaml         # Phi-4-mini → pi4-node04
├── download-models-job.yaml
└── VERSION
```

### Steps

```bash
git checkout a1bc026^ -- apps/llama-cpp/
```

No modifications needed — the chart, Dockerfile, and per-node values all worked previously.

---

## Phase 2: Extend the Chart for RPC Workers

Add RPC tensor-parallelism support to the existing chart. This is an additive change — when `rpc.enabled: false` (default), the chart behaves exactly as before.

### 2.1 Add RPC values to `chart/values.yaml`

```yaml
# Append to existing values.yaml
rpc:
  enabled: false
  port: 50052
  workers: []
  # Example:
  # workers:
  #   - host: pi4-node02.home.mowntan.com
  #     port: 50052
  #     mem: 7000
```

### 2.2 Modify `chart/templates/deployment.yaml`

Add conditional `--rpc` flags to the leader's command block:

```yaml
# Inside the set -- command block, after extraArgs:
{{- if .Values.rpc.enabled }}
{{- range .Values.rpc.workers }}
set -- "$@" --rpc {{ .host }}:{{ .port }}
{{- end }}
{{- end }}
```

### 2.3 Create `chart/templates/rpc-worker.yaml`

New template — only rendered when `rpc.enabled: true`. Deploys a DaemonSet running `llama-rpc-server` on each worker node.

```yaml
{{- if .Values.rpc.enabled }}
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: {{ include "llama-cpp.fullname" . }}-rpc-worker
  labels:
    {{- include "llama-cpp.labels" . | nindent 4 }}
    app.kubernetes.io/component: rpc-worker
spec:
  selector:
    matchLabels:
      {{- include "llama-cpp.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: rpc-worker
  template:
    metadata:
      labels:
        {{- include "llama-cpp.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: rpc-worker
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
        - name: rpc-worker
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          command:
            - llama-rpc-server
            - --host
            - "0.0.0.0"
            - --port
            - {{ .Values.rpc.port | quote }}
            - --mem
            - {{ .Values.rpc.mem | default "7000" | quote }}
          ports:
            - name: rpc
              containerPort: {{ .Values.rpc.port }}
              protocol: TCP
          startupProbe:
            tcpSocket:
              port: rpc
            failureThreshold: 30
            periodSeconds: 5
          livenessProbe:
            tcpSocket:
              port: rpc
            periodSeconds: 30
          readinessProbe:
            tcpSocket:
              port: rpc
            periodSeconds: 10
          {{- with .Values.rpc.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- with .Values.rpc.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.rpc.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
{{- end }}
```

**Why DaemonSet instead of StatefulSet?** RPC workers are stateless — they don't hold model data, they just expose memory. The leader streams weight data to them at startup. A DaemonSet with `nodeSelector` pins one worker per node with no ordinal management needed.

**Why hostNetwork?** Our distributed-llama experience showed that CNI overlay can cause socket issues during large transfers. `hostNetwork: true` uses the host's network stack directly, which is simpler and more reliable for the sustained TCP connections RPC uses. The leader references workers by hostname (e.g., `pi4-node02.home.mowntan.com:50052`), bypassing Kubernetes service DNS entirely.

### 2.4 Modify the leader Deployment for RPC mode

When `rpc.enabled`, the leader needs:
- `hostNetwork: true` (to reach workers on host network)
- `dnsPolicy: ClusterFirstWithHostNet`
- A longer startup probe (model distribution takes time over GbE)

Add to `deployment.yaml`:

```yaml
spec:
  template:
    spec:
      {{- if .Values.rpc.enabled }}
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      {{- end }}
      # ... rest of spec
```

And increase the startup probe for RPC mode:

```yaml
startupProbe:
  httpGet:
    path: /health
    port: http
  failureThreshold: {{ if .Values.rpc.enabled }}240{{ else }}60{{ end }}
  periodSeconds: 15
  timeoutSeconds: 10
```

### 2.5 Security

RPC has **no authentication**. The DaemonSet uses `hostNetwork`, so port 50052 is bound on the host interface. Ensure:
- UFW on each Pi4 blocks port 50052 from external networks
- Or use `--host 10.x.x.x` (cluster-internal IP) instead of `0.0.0.0`

---

## Phase 3: Values Files for the 14B Distributed Model

### 3.1 Create `values/values-qwen14b-distributed.yaml`

This is the leader + workers config, deployed as a single Helm release:

```yaml
fullnameOverride: llama-qwen14b

model:
  path: /models/Qwen2.5-14B-Instruct-Q4_K_M.gguf
  name: "Large (Qwen2.5-14B)"
  contextSize: 4096

rpc:
  enabled: true
  port: 50052
  mem: 7000
  workers:
    - host: pi4-node02.home.mowntan.com
      port: 50052
    - host: pi4-node03.home.mowntan.com
      port: 50052
    - host: pi4-node04.home.mowntan.com
      port: 50052
  nodeSelector:
    node-role.kubernetes.io/inference: ""
  tolerations:
    - key: node.longhorn.io/storage
      operator: Exists
      effect: NoSchedule

tolerations:
  - key: node.longhorn.io/storage
    operator: Exists
    effect: NoSchedule

# Leader pinned to node01
nodeSelector:
  kubernetes.io/hostname: pi4-node01.home.mowntan.com

persistence:
  models:
    nfs:
      server: synology-nas01.home.mowntan.com
      path: /volume2/downloads/models
```

**Memory budget (Qwen2.5-14B-Instruct-Q4_K_M):**

| Component | Size |
|---|---|
| Model weights (Q4_K_M) | ~8.5 GB |
| Per-node share (4 nodes) | ~2.1 GB |
| KV cache (4096 ctx) | ~0.5 GB per node |
| OS + k3s overhead | ~1.5 GB |
| **Available per node** | **~3.9 GB headroom** |

This fits comfortably within 8 GB per Pi4.

---

## Phase 4: Download the 14B Model

### 4.1 Update `download-models-job.yaml`

Add one line to the download script:

```bash
download bartowski/Qwen2.5-14B-Instruct-GGUF Qwen2.5-14B-Instruct-Q4_K_M.gguf
```

This is an 8.5 GB file. It will be stored on NFS and only the leader needs to read it (RPC handles distributing weights to workers).

### 4.2 Run the job

```bash
sudo kubectl apply -f apps/llama-cpp/download-models-job.yaml
sudo kubectl logs -f job/download-models
```

---

## Phase 5: Deploy

### 5.1 Build and push the Docker image (if not already on ghcr.io)

```bash
cd apps/llama-cpp/docker

# Check if image exists
docker manifest inspect ghcr.io/mowntan/llama-cpp:b8148 2>/dev/null && echo "exists" || echo "needs build"

# Build for arm64
docker buildx build --platform linux/arm64 -t ghcr.io/mowntan/llama-cpp:b8148 --push .
```

### 5.2 Deploy the distributed 14B model

Single Helm release deploys both the RPC workers (DaemonSet) and the leader (Deployment):

```bash
helm upgrade --install llama-qwen14b apps/llama-cpp/chart \
  -f apps/llama-cpp/values/values-qwen14b-distributed.yaml
```

### 5.3 Verify

```bash
# Check RPC workers are running
sudo kubectl get pods -l app.kubernetes.io/component=rpc-worker

# Check leader starts and connects
sudo kubectl logs deployment/llama-qwen14b -f

# Smoke test
sudo kubectl run curl --rm -it --image=curlimages/curl -- \
  curl -s http://llama-qwen14b:8080/v1/models
```

---

## Phase 6: Wire Up Open WebUI

### 6.1 Deploy Open WebUI

Restore or deploy Open WebUI with the 14B endpoint added to `OPENAI_API_BASE_URLS`:

```yaml
env:
  - name: OPENAI_API_BASE_URLS
    value: "http://llama-qwen14b:8080/v1"
```

If also running per-node smaller models (Phase 7), include all endpoints:

```yaml
env:
  - name: OPENAI_API_BASE_URLS
    value: "http://llama-qwen14b:8080/v1;http://llama-coder:8080/v1;http://llama-mistral:8080/v1;http://llama-llama:8080/v1;http://llama-phi:8080/v1"
```

### 6.2 Ingress

Ensure the Traefik IngressRoute for `chat.apps.mowntan.com` points to Open WebUI.

---

## Phase 7: (Optional) Restore Per-Node Replica Models

If you also want the smaller models running alongside the distributed 14B, re-deploy the original 4 per-node releases. However, this would conflict since all 4 inference nodes are used as RPC workers for the 14B.

**Option A: Dedicated 14B cluster (all 4 nodes)**
- All nodes run RPC workers + one leader
- Only the 14B model is available
- Maximum memory for the distributed model

**Option B: Mixed — 1 leader + 3 RPC workers, no small models on inference nodes**
- Same as Option A but acknowledges the tradeoff explicitly

**Option C: Run small models on homelab nodes instead**
- Move Coder/Mistral/Llama/Phi to pi4-node05 through pi4-node08
- Keep all 4 inference nodes for the 14B
- Requires homelab nodes to have enough RAM for the smaller models

---

## Rollback Plan

If RPC doesn't work on the Pi4 cluster (as distributed-llama didn't), fall back to replica parallelism:

```bash
# Remove the distributed deployment
helm uninstall llama-qwen14b

# Restore individual per-node deployments
helm upgrade --install llama-coder apps/llama-cpp/chart -f apps/llama-cpp/values/values-coder.yaml
helm upgrade --install llama-mistral apps/llama-cpp/chart -f apps/llama-cpp/values/values-mistral.yaml
helm upgrade --install llama-llama apps/llama-cpp/chart -f apps/llama-cpp/values/values-llama.yaml
helm upgrade --install llama-phi apps/llama-cpp/chart -f apps/llama-cpp/values/values-phi.yaml
```

---

## Key Differences from distributed-llama Attempt

| Issue we hit with dllama | How llama.cpp RPC differs |
|---|---|
| Custom binary, custom protocol, no error recovery | llama.cpp is mature, widely tested, has proper error handling |
| Workers held model shards — crash = reload everything | RPC workers are stateless memory pools — leader retries are cheaper |
| No existing Helm chart — built everything from scratch | Extending our existing, proven chart |
| NnTransferSocketException during weight distribution | RPC uses GGML's backend abstraction, not raw socket transfers |
| `hostNetwork` was a late fix attempt | `hostNetwork` is designed in from the start for RPC workers |
| Docker image had to be custom-built | Docker image already builds `llama-rpc-server` alongside `llama-server` |

---

## Open Questions

1. **llama.cpp RPC on arm64 Pi4** — Has anyone successfully run RPC tensor parallelism on Raspberry Pi 4? The Arm Learning Path docs cover Graviton (server-class arm64) but not Pi4 specifically.
2. **GbE bandwidth** — At ~125 MB/s theoretical max, distributing 8.5 GB of weights takes ~68 seconds minimum. Is the RPC protocol tolerant of slow links?
3. **KV cache distribution** — Does RPC spread the KV cache across workers, or does only the leader hold it? This affects the effective context window at 4096.
4. **Startup ordering** — RPC workers must be ready before the leader starts. The current chart doesn't enforce this. Options: init container (as dllama chart had), or rely on llama-server's retry logic.
