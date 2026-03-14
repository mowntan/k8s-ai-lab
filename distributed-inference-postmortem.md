# Distributed Inference on Pi4 k3s: Postmortem

## Final Deployment

4 single-node llama-cpp instances, each running one model on one Pi4 (8 GB RAM), fronted by Open WebUI at `chat.apps.mowntan.com`.

| Node | Model | GGUF Size | Service |
|---|---|---|---|
| pi4-node01 | Qwen2.5-Coder-7B (Q4_K_M) | ~4.5 GB | `llama-coder:8080` |
| pi4-node02 | Mistral-7B (Q4_K_M) | ~4.2 GB | `llama-mistral:8080` |
| pi4-node03 | Llama-3.2-3B (Q4_K_M) | ~2.0 GB | `llama-llama:8080` |
| pi4-node04 | Phi-4-mini (Q4_K_M) | ~2.5 GB | `llama-phi:8080` |

Open WebUI runs on a homelab node (pi4-node07) and round-robins requests across all 4 endpoints via `OPENAI_API_BASE_URLS`.

### Why this approach

We attempted two different distributed inference strategies to pool all 4 nodes for a single larger model (Qwen2.5-14B). Both failed due to fundamental networking issues on Pi4 k3s. The single-node replica approach is the only reliable option — each model must fit entirely in one node's 8 GB RAM (~6 GB usable after OS/k3s overhead).

---

## Attempt 1: distributed-llama (dllama)

### What it is

A purpose-built distributed inference binary. Workers hold contiguous shards of model layers. The root node reads the full model, distributes weight slices to workers over TCP, then orchestrates the forward pass.

### What we built

- Custom Docker image from source (arm64), pushed to ghcr.io
- Helm chart with StatefulSet workers, headless service, root Deployment
- `ghcr-secret` for private image pulls
- Tried models: Qwen3-30B-A3B (18 GB) and Qwen3-14B (10.9 GB)

### Issues encountered

#### 1. ghcr.io push denied — missing `write:packages` scope
**Symptom:** `docker push` to ghcr.io returned 403.
**Fix:** `gh auth refresh -h github.com -s write:packages` with device flow authentication.

#### 2. ImagePullBackOff — private ghcr.io package
**Symptom:** Pods couldn't pull the image after push.
**Fix:** Created a `ghcr-secret` docker-registry secret and added `imagePullSecrets` to the pod templates.

#### 3. Worker DNS not resolving
**Symptom:** Root couldn't reach workers by their StatefulSet DNS names.
**Fix:** Added `publishNotReadyAddresses: true` to the headless service. Without this, pods that haven't passed readiness checks don't get DNS A records.

#### 4. Unknown option `--weights-float-type`
**Symptom:** The `main` branch build didn't recognize this flag from the v0.16.5 docs.
**Fix:** Made the flag conditional in the Helm template — skip it when the value is empty.

#### 5. Workers crash-looping with exponential backoff
**Symptom:** Workers crashed during weight loading, then k8s backed off restarts (30s, 60s, 120s...).
**Fix:** Wrapped the worker command in a `while true` restart loop so it retries in 1 second instead of waiting for k8s backoff.

#### 6. Liveness probe killing workers during weight loading (KEY DISCOVERY)
**Symptom:** Workers were killed at exactly the liveness probe timeout during weight transfer. The TCP port was busy receiving weights and couldn't respond to probes.
**Fix:** Added a startup probe with `failureThreshold: 360, periodSeconds: 10` (1 hour window). The liveness probe only kicks in after the startup probe succeeds.

#### 7. Root crashes with `NnTransferSocketException` during weight distribution
**Symptom:** The root consistently crashed at approximately block 17-18 of 48 layers with `Error writing to socket`. This happened regardless of:
- Model size (30B at 5513 MB/node OR 14B at 2234 MB/node)
- Storage backend (NFS or local hostPath on the node)
- Network mode (k3s CNI overlay or `hostNetwork: true`)
- Available memory (6900+ MB free, no OOM kills in `dmesg`)

**Root cause:** Unknown. The same model (Qwen3-30B-A3B) was reported working on 4x Pi5 8GB using bare metal (not Kubernetes) in GitHub discussion #255. The containerized environment (k3s networking, cgroups, etc.) appears to interfere with dllama's raw TCP socket transfers.

**Outcome:** Abandoned distributed-llama after exhausting all options.

---

## Attempt 2: llama.cpp RPC

### What it is

llama.cpp's built-in RPC backend. Workers (`rpc-server`) expose node memory as a remote GGML backend over TCP port 50052. The leader (`llama-server`) connects to workers via `--rpc host:port` and distributes model weights and KV cache proportionally to advertised memory.

### What we built

- Extended the existing llama-cpp Helm chart with `rpc.enabled` toggle
- DaemonSet for RPC workers with node affinity (only on worker nodes)
- Headless service for worker pod discovery
- Init container on the leader to wait for all workers before starting
- Updated Docker image from b8148 to b8340

### Issues encountered

#### 1. `--mem` flag doesn't exist
**Symptom:** Workers crashed with `error: unknown argument: --mem`.
**Fix:** Removed `--mem` from the worker command. The rpc-server in b8148/b8340 doesn't have this flag — it auto-detects available memory.

#### 2. DaemonSet placed a worker on the leader node
**Symptom:** 4 workers deployed (one per inference node), but one landed on node01 alongside the leader.
**Fix:** Added `nodeAffinity` to the DaemonSet template that restricts scheduling to only the hostnames listed in `rpc.workers[]`.

#### 3. `--rpc` flag only accepts comma-separated values
**Symptom:** `DEPRECATED: argument '--rpc' specified multiple times, use comma-separated values instead (only last value will be used)`. The leader only connected to the last worker.
**Fix:** Changed the template from multiple `--rpc` flags to a single `--rpc host1:port,host2:port,host3:port`.

#### 4. Init container `nc -z` didn't work with busybox
**Symptom:** Init container hung forever waiting for workers, even though workers were running.
**Fix:** Busybox `nc` doesn't support `-z` properly. The RPC server accepts the connection then closes it (not a valid RPC client), causing `nc` to exit with code 1. Changed the check to treat exit codes 0 and 1 both as "port is open" (`[ "$rc" -le 1 ] && return 0`).

#### 5. llama-server hangs silently with `hostNetwork` (CRITICAL)
**Symptom:** With `hostNetwork: true`, `llama-server --rpc <remote-ip>:50052` hung indefinitely with zero log output. Only 15 MB RSS — the process never started loading the model.

**Investigation:**
- `nc` from the same pod connected to workers successfully
- Worked fine to localhost (127.0.0.1) and own IP (192.168.50.5)
- Failed to any remote IP (192.168.50.6/7/8) even with raw IPs (no DNS)
- Same behavior as root and as non-root user
- Same behavior on both b8148 and b8340

**Root cause found via strace:** The RPC client uses a **blocking `connect()` with no timeout**. The `connect()` syscall sent a SYN to the remote worker but never received a SYN-ACK back — it hung in `SYN_SENT` state forever. `nc` works because it uses non-blocking connect (`EINPROGRESS`) with a timeout.

The SYN packets are being silently dropped by the k3s host network stack (likely kube-proxy iptables rules or conntrack) for `hostNetwork` pod-to-pod communication. This only affects outbound connections from `hostNetwork` pods to other `hostNetwork` pods on different nodes. Same-node connections bypass the network stack via loopback.

#### 6. Switched to pod network — model loads but inference crashes
**Symptom:** After removing `hostNetwork` and using the pod network (CNI), the RPC connections succeeded. The model loaded and distributed across all 4 nodes. But every inference request crashed with:

```
ggml-rpc.cpp:792: Remote RPC server crashed or returned malformed response
```

**Details:**
- Model loading (weight distribution) completed successfully
- Server started and listened on :8080
- First inference request triggered the crash every time
- Workers showed no errors on their end
- The 3B model was used — memory was not the issue (~2 GB model, ~6 GB free per node)

**Root cause:** The k3s CNI overlay network (flannel/VXLAN) is unreliable for the tight, latency-sensitive tensor computation traffic that RPC requires. Packets are being lost or corrupted during the compute phase, causing the RPC protocol to see malformed responses.

**Outcome:** Abandoned distributed inference entirely. The Pi4 k3s network stack (both host network and CNI overlay) cannot reliably support the sustained, high-bandwidth, low-latency TCP connections required by distributed inference protocols.

---

## Summary of Root Causes

Both distributed inference attempts failed for the same fundamental reason: **the Pi4 k3s network is not reliable enough for distributed tensor operations.**

| Layer | Issue |
|---|---|
| Host network (hostNetwork) | kube-proxy iptables rules silently drop SYN packets for inter-node hostNetwork pod traffic. llama.cpp's blocking `connect()` with no timeout hangs forever. |
| Pod network (CNI/flannel) | VXLAN overlay adds encapsulation overhead. Packets are dropped or corrupted during high-bandwidth RPC compute operations, causing `Remote RPC server crashed or returned malformed response`. |
| distributed-llama | Raw TCP socket transfers crash with `NnTransferSocketException` during weight distribution — same network reliability issue. |

### Why bare metal works but k3s doesn't

Reports show distributed inference working on bare-metal Pi clusters (no Kubernetes). The difference:
- **Bare metal:** Direct TCP socket between processes on the host network. No iptables rules, no VXLAN encapsulation, no conntrack, no kube-proxy.
- **k3s hostNetwork:** The pod uses the host's network stack, but k3s still applies iptables rules (kube-proxy, service routing, masquerading) that interfere with raw TCP connections between nodes.
- **k3s pod network:** Adds VXLAN encapsulation (flannel) on top of everything, further degrading reliability for sustained high-bandwidth transfers.

### Possible future approaches

1. **Bare metal daemons:** Run rpc-server/dllama workers as systemd services directly on the Pi4s, outside of Kubernetes. Use k8s only for the leader and ancillary services.
2. **Calico or direct routing:** Replace flannel/VXLAN with a CNI that uses direct routing (no encapsulation). May improve pod network reliability.
3. **kube-proxy bypass:** Use `hostNetwork` with custom iptables rules that exempt the RPC port from kube-proxy processing.
4. **Wait for fixes:** The blocking `connect()` in llama.cpp RPC is a bug. If fixed upstream (non-blocking with timeout), `hostNetwork` might work.
5. **Pi5 upgrade:** Pi5 has faster networking and CPU. The extra headroom might make the difference.
