# Speculative Decoding on k3s: Implementation Plan

## Goal

Improve single-request token generation speed on the existing 4× Pi4 k3s cluster by enabling speculative decoding within the `llama-cpp` Helm chart. Each node continues to run a single `llama-server` instance, but each server loads both a target model and a small draft model from the same family. No inter-node networking is required — everything runs in-process on a single node.

---

## Background: How Speculative Decoding Works

Speculative decoding pairs two models on the same node:

- **Draft model** — small, fast, cheap (e.g. Qwen2.5-Coder-0.5B). Generates several candidate tokens speculatively in a burst.
- **Target model** — large, accurate, slow (e.g. Qwen2.5-Coder-7B). Verifies the draft tokens in a single parallel forward pass, accepting correct ones and rejecting incorrect ones.

Because the target model verifies a batch of draft tokens in roughly the same time it would take to generate a single token, accepted draft tokens are essentially "free". The result is identical output quality to running the target model alone, with 1.5–2.5× throughput improvement on tasks with predictable token sequences (code, structured output, templated responses).

### Key constraints

- Draft and target models **must share the same tokenizer vocabulary**. The safest pairs are from the same model family (e.g. Qwen2.5 × Qwen2.5, Llama 3.2 × Llama 3.2).
- Both models load into the **same `llama-server` process** on the same node. Memory budget = target model + draft model, both must fit within ~6 GB usable RAM.
- Speed gain is task-dependent. Code generation and structured/repetitive output benefit most. Open-ended creative prompts benefit least.
- Flash attention (`--flash-attn`) should be **disabled** when using speculative decoding (known incompatibility in llama.cpp).

---

## Model Pairing Plan

The existing fleet maps cleanly to same-family draft pairs. All sizes are Q4_K_M GGUF.

| Node | Target model | Target size | Draft model | Draft size | Total | Headroom |
|---|---|---|---|---|---|---|
| pi4-node01 | Qwen2.5-Coder-7B-Instruct | ~4.5 GB | Qwen2.5-Coder-0.5B-Instruct | ~0.4 GB | ~4.9 GB | ~1.1 GB |
| pi4-node02 | Mistral-7B-Instruct-v0.3 | ~4.2 GB | — (no same-family 0.5B; use ngram) | — | ~4.2 GB | ~1.8 GB |
| pi4-node03 | Llama-3.2-3B-Instruct | ~2.0 GB | Llama-3.2-1B-Instruct | ~0.8 GB | ~2.8 GB | ~3.2 GB |
| pi4-node04 | Phi-4-mini-Instruct | ~2.5 GB | — (no compatible draft; use ngram) | — | ~2.5 GB | ~3.5 GB |

**Notes:**
- Mistral and Phi-4-mini have no maintained same-family sub-1B GGUF drafts. Use `--spec-type ngram-simple` as a no-draft fallback — this gives modest gains (~1.1–1.3×) on repetitive output with no memory overhead.
- Qwen2.5-Coder-0.5B and Llama-3.2-1B are the confirmed compatible draft models from community benchmarks. The 0.5B draft paired with a 7B Qwen target achieves up to 2.5× speedup on coding tasks.
- All models fit comfortably within 8 GB after leaving ~1–2 GB for OS + k3s overhead.

---

## Phase 1: Download Draft Models

### 1.1 Update `download-models-job.yaml`

Add the two draft models to the existing download script. The job already handles downloading to NFS (`/volume2/downloads/models` on `synology-nas01`).

```yaml
# Add to the download script in download-models-job.yaml:
download bartowski/Qwen2.5-Coder-0.5B-Instruct-GGUF Qwen2.5-Coder-0.5B-Instruct-Q4_K_M.gguf
download bartowski/Llama-3.2-1B-Instruct-GGUF Llama-3.2-1B-Instruct-Q4_K_M.gguf
```

### 1.2 Run the download job

```bash
kubectl apply -f apps/llama-cpp/download-models-job.yaml -n ai-lab
kubectl logs -f job/download-models -n ai-lab
```

Verify files are present on NFS before proceeding:

```bash
ls /volume2/downloads/models/ | grep -E "0.5B|1B-Instruct"
# Expected:
# Qwen2.5-Coder-0.5B-Instruct-Q4_K_M.gguf
# Llama-3.2-1B-Instruct-Q4_K_M.gguf
```

---

## Phase 2: Extend the Helm Chart

### 2.1 Add speculative decoding values to `chart/values.yaml`

Append the following block to the existing `values.yaml`. When `speculative.enabled: false` (default), chart behaviour is unchanged.

```yaml
# Speculative decoding configuration
speculative:
  enabled: false
  # Path to the draft model GGUF on the same volume as the target model
  draftModelPath: ""
  # Number of tokens to speculatively generate per iteration (default 16 is llama.cpp default)
  draftMax: 16
  draftMin: 1
  # Minimum draft acceptance probability threshold (0.0–1.0, default 0.75)
  draftPMin: 0.75
  # Fallback spec type when no draft model is provided
  # Options: none | ngram-simple | ngram-cache | ngram-map-k
  specType: "none"
```

### 2.2 Modify `chart/templates/deployment.yaml`

Extend the `llama-server` command block to conditionally add speculative decoding flags. Add this block after the existing `extraArgs` section:

```yaml
{{- if .Values.speculative.enabled }}
{{- if .Values.speculative.draftModelPath }}
set -- "$@" --model-draft {{ .Values.speculative.draftModelPath | quote }}
set -- "$@" --draft-max {{ .Values.speculative.draftMax | default 16 }}
set -- "$@" --draft-min {{ .Values.speculative.draftMin | default 1 }}
set -- "$@" --draft-p-min {{ .Values.speculative.draftPMin | default "0.75" }}
{{- else if ne .Values.speculative.specType "none" }}
set -- "$@" --spec-type {{ .Values.speculative.specType }}
{{- end }}
{{- end }}
exec llama-server "$@"
```

**Important:** Ensure `--flash-attn` is **not** set when `speculative.enabled: true`. Add a guard:

```yaml
{{- if not .Values.speculative.enabled }}
{{- if .Values.flashAttention }}
set -- "$@" --flash-attn
{{- end }}
{{- end }}
```

### 2.3 Update `chart/Chart.yaml`

Bump the chart version to reflect the new capability:

```yaml
version: 2.1.0
```

---

## Phase 3: Create Updated Values Files

### 3.1 `values/values-coder.yaml` — Qwen2.5-Coder-7B with 0.5B draft

```yaml
fullnameOverride: llama-coder

model:
  path: /models/Qwen2.5-Coder-7B-Instruct-Q4_K_M.gguf
  name: "Coder (Qwen2.5-7B)"
  contextSize: 4096  # Keep at 4096 initially — KV cache doubles at 8192 (~500 MB → ~1 GB), eating into the ~1.1 GB headroom on node01. Bump to 8192 only after confirming memory is stable.

speculative:
  enabled: true
  draftModelPath: /models/Qwen2.5-Coder-0.5B-Instruct-Q4_K_M.gguf
  draftMax: 16
  draftMin: 1
  draftPMin: 0.75

tolerations:
  - key: node.longhorn.io/storage
    operator: Exists
    effect: NoSchedule

nodeSelector:
  kubernetes.io/hostname: pi4-node01.home.mowntan.com

persistence:
  models:
    nfs:
      server: synology-nas01.home.mowntan.com
      path: /volume2/downloads/models
```

### 3.2 `values/values-llama.yaml` — Llama-3.2-3B with 1B draft

```yaml
fullnameOverride: llama-llama

model:
  path: /models/Llama-3.2-3B-Instruct-Q4_K_M.gguf
  name: "Llama (3.2-3B)"
  contextSize: 4096  # Bump to 8192 after confirming memory headroom post-deploy.

speculative:
  enabled: true
  draftModelPath: /models/Llama-3.2-1B-Instruct-Q4_K_M.gguf
  draftMax: 8
  draftMin: 1
  draftPMin: 0.75

tolerations:
  - key: node.longhorn.io/storage
    operator: Exists
    effect: NoSchedule

nodeSelector:
  kubernetes.io/hostname: pi4-node03.home.mowntan.com

persistence:
  models:
    nfs:
      server: synology-nas01.home.mowntan.com
      path: /volume2/downloads/models
```

**Note:** `draftMax: 8` for the 3B/1B pairing. The size ratio is smaller (3× vs 14×), so shorter draft windows are more efficient — the efficiency crossover is around 5 tokens for a 1B draft.

### 3.3 `values/values-mistral.yaml` — Mistral-7B with ngram fallback

```yaml
fullnameOverride: llama-mistral

model:
  path: /models/Mistral-7B-Instruct-v0.3-Q4_K_M.gguf
  name: "Mistral (7B)"
  contextSize: 4096  # Bump to 8192 after confirming memory headroom post-deploy.

speculative:
  enabled: true
  specType: "ngram-simple"
  # No draftModelPath — ngram uses token history, no second model loaded

tolerations:
  - key: node.longhorn.io/storage
    operator: Exists
    effect: NoSchedule

nodeSelector:
  kubernetes.io/hostname: pi4-node02.home.mowntan.com

persistence:
  models:
    nfs:
      server: synology-nas01.home.mowntan.com
      path: /volume2/downloads/models
```

### 3.4 `values/values-phi.yaml` — Phi-4-mini with ngram fallback

```yaml
fullnameOverride: llama-phi

model:
  # Verify exact filename on NFS — deployed model uses a microsoft_ prefix.
  # Check with: ls /volume2/downloads/models/ | grep -i phi
  path: /models/microsoft_Phi-4-mini-instruct-Q4_K_M.gguf
  name: "Phi (4-mini)"
  contextSize: 4096  # Bump to 8192 after confirming memory headroom post-deploy.

speculative:
  enabled: true
  specType: "ngram-simple"

tolerations:
  - key: node.longhorn.io/storage
    operator: Exists
    effect: NoSchedule

nodeSelector:
  kubernetes.io/hostname: pi4-node04.home.mowntan.com

persistence:
  models:
    nfs:
      server: synology-nas01.home.mowntan.com
      path: /volume2/downloads/models
```

---

## Phase 4: Deploy

### 4.1 Deploy in order, verifying each before proceeding

Start with the Llama node (smallest models, largest safety margin) to validate the chart changes before touching the more heavily used Coder instance.

```bash
# Node 3 first — smallest memory footprint, lowest risk
helm upgrade --install llama-llama apps/llama-cpp/chart \
  -f apps/llama-cpp/values/values-llama.yaml \
  -n ai-lab

# Verify it loaded both models
kubectl logs deployment/llama-llama -n ai-lab | grep -E "model|draft|spec"
# Expected lines:
#   llm_load_print_meta: model type = 3B
#   llm_load_print_meta: model type = 1B   ← draft model
#   srv  speculative decoding enabled

# Node 1 — Coder with 0.5B draft
helm upgrade --install llama-coder apps/llama-cpp/chart \
  -f apps/llama-cpp/values/values-coder.yaml \
  -n ai-lab

# Node 2 — Mistral with ngram
helm upgrade --install llama-mistral apps/llama-cpp/chart \
  -f apps/llama-cpp/values/values-mistral.yaml \
  -n ai-lab

# Node 4 — Phi with ngram
helm upgrade --install llama-phi apps/llama-cpp/chart \
  -f apps/llama-cpp/values/values-phi.yaml \
  -n ai-lab
```

### 4.2 Smoke test each endpoint

```bash
for svc in llama-coder llama-mistral llama-llama llama-phi; do
  echo "--- $svc ---"
  kubectl run curl-test --rm -it --image=curlimages/curl --restart=Never -n ai-lab -- \
    curl -s http://${svc}:8080/v1/models | jq '.data[].id'
done
```

---

## Phase 5: Benchmark and Tune

### 5.1 Measure baseline vs speculative throughput

Use `llama-bench` or the server's `/metrics` endpoint to compare token generation speed before and after. The server exposes speculative decoding statistics in its logs:

```
draft acceptance rate = 0.68 (340 accepted / 500 generated)
```

A healthy acceptance rate for same-family pairs is 0.55–0.75+. Below 0.4 means the draft model is hurting more than helping for that prompt type.

### 5.2 Tuning `draftMax` per workload

| Workload | Recommended `draftMax` | Expected acceptance rate |
|---|---|---|
| Code completion (Coder model) | 16 | 0.65–0.75 |
| Code refactoring (long predictable blocks) | 24 | 0.70–0.80 |
| General chat (Llama/Mistral) | 8 | 0.50–0.65 |
| Short factual Q&A | 4–6 | 0.45–0.60 |

If acceptance rate is consistently below 0.45, reduce `draftMax`. If it's consistently above 0.70, try increasing it.

### 5.3 Tuning `draftPMin`

The default `0.75` means the draft token must have at least 75% probability of being correct (as estimated by the target model's top prediction) to be accepted. Lowering this to `0.5` accepts more tokens and increases throughput on high-quality drafts, but may marginally affect output quality. Start at `0.75` and tune down if acceptance rate is low.

---

## Phase 6: Rollback Plan

Speculative decoding is a flag-level change — rolling back is a single Helm upgrade with the flag removed.

```bash
# Disable speculative decoding on a single release
helm upgrade llama-coder apps/llama-cpp/chart \
  -f apps/llama-cpp/values/values-coder.yaml \
  --set speculative.enabled=false \
  -n ai-lab

# Or restore the pre-speculative values files entirely
helm upgrade --install llama-coder apps/llama-cpp/chart \
  -f apps/llama-cpp/values/values-coder-pre-speculative.yaml \
  -n ai-lab
```

Keep a copy of the current working values files as `values-*-pre-speculative.yaml` before starting Phase 4.

---

## Summary

| Node | Change | Expected speedup |
|---|---|---|
| pi4-node01 (Coder) | Add Qwen2.5-Coder-0.5B draft | 1.5–2.5× on code tasks |
| pi4-node02 (Mistral) | ngram-simple fallback | 1.1–1.3× on structured output |
| pi4-node03 (Llama) | Add Llama-3.2-1B draft | 1.4–1.8× on general tasks |
| pi4-node04 (Phi) | ngram-simple fallback | 1.1–1.3× on structured output |

This approach requires no inter-node networking, no new Kubernetes primitives, no changes to the Docker image (speculative decoding is a standard `llama-server` flag since build b8000+), and no changes to Open WebUI or the ingress. The only risk is an unexpected memory pressure on a node if model size estimates are off — mitigated by deploying the smallest-model node first and checking `kubectl top nodes` before proceeding to node01.

---

## Open Questions

1. **Qwen2.5-Coder-0.5B memory footprint on arm64** — The 0.4 GB estimate is based on Q4_K_M quantization. Verify actual RSS after loading: `kubectl exec deployment/llama-coder -n ai-lab -- cat /proc/meminfo | grep MemAvailable`. If headroom drops below 800 MB, switch to a Q2_K draft to reduce model size further.

2. **ngram effectiveness on Mistral/Phi** — ngram-simple provides modest gains only on repetitive or structured prompts. If real-world acceptance rate stays below 1.2× on typical chat workloads, disabling it (setting `speculative.enabled: false`) is preferable to the small overhead cost.

3. **Flash attention interaction** — The test suite confirms flash attention and speculative decoding are incompatible. If the current values files have `--flash-attn` enabled, ensure the chart guard in Phase 2.2 removes it when `speculative.enabled: true`. Check current flags with:
   ```bash
   kubectl exec deployment/llama-coder -n ai-lab -- ps aux | grep llama-server
   ```

---

## References

- llama.cpp speculative decoding docs: `docs/speculative.md` in the llama.cpp repository
- Community benchmarks (Qwen2.5 draft pairings): `github.com/ggml-org/llama.cpp/discussions/10466`
- llama-server `--model-draft` flag: `tools/server/README.md`
