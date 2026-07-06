# GPU sharing modes: exclusive vs. MPS-shared

KAI Scheduler v0.12.10 supports two **alternative, non-combinable**
allocation paths for GPUs on a single pod:

1. **Exclusive**: `resources.limits."nvidia.com/gpu": N` — a normal
   Kubernetes extended-resource request for N whole physical GPUs.
2. **MPS-shared / fractional**: the unprefixed `gpu-memory` pod annotation
   (a MiB value), which KAI schedules and bin-packs as a fraction of a
   physical GPU's memory, backed by NVIDIA MPS (Multi-Process Service) for
   actual sharing.

**Do not set both on the same pod.** Setting `nvidia.com/gpu` alongside a
`gpu-memory` annotation is invalid/undefined behavior in KAI v0.12.10 — pick
one path per pod.

## When to use which

| | Exclusive (`nvidia.com/gpu`) | MPS-shared (`gpu-memory`) |
|---|---|---|
| Real NVLink/PCIe peer-to-peer between GPUs | ✅ | ❌ — not a supported combination with NCCL collectives |
| Multiple small jobs sharing one physical card | ❌ | ✅ |
| Predictable, isolated compute throughput | ✅ | ⚠️ MPS does not partition compute/bandwidth — a noisy neighbor can degrade co-located jobs |
| Works with `nvidia.com/gpu: 2+` in one pod | ✅ | ❌ — MPS-shared requests are capped at 1 GPU-equivalent per pod |

If your job does distributed training across multiple GPUs (DDP,
MultiWorkerMirroredStrategy) and needs real inter-GPU bandwidth, use
**exclusive** allocation — see [gang-scheduling.md](gang-scheduling.md).

If your job is small, interactive, or embarrassingly parallel and doesn't
need the whole card, use **MPS-shared** to improve utilization.

## The MPS enforcement gotcha (read this before using MPS-shared)

This is the single most common mistake when setting up MPS-shared jobs, and
it fails **silently** — no error, no warning, just an unenforced limit.

Setting `CUDA_MPS_PINNED_DEVICE_MEM_LIMIT` on a container is **not enough**
to enforce a VRAM limit. That environment variable only has an effect on a
process that has actually connected to the MPS control daemon as a client.
That connection requires **two more environment variables** and **matching
volume mounts**:

```yaml
env:
  - name: CUDA_MPS_PINNED_DEVICE_MEM_LIMIT
    value: "0=5120M"          # must match your gpu-memory annotation (MiB)
  - name: CUDA_MPS_PIPE_DIRECTORY
    value: "/mps/nvidia.com/gpu/pipe"
  - name: CUDA_MPS_LOG_DIRECTORY
    value: "/mps/nvidia.com/gpu/log"
volumeMounts:
  - name: mps-pipe
    mountPath: /mps/nvidia.com/gpu/pipe
  - name: mps-log
    mountPath: /mps/nvidia.com/gpu/log
volumes:
  - name: mps-pipe
    hostPath:
      path: /run/nvidia/mps/nvidia.com/gpu/pipe
      type: DirectoryOrCreate
  - name: mps-log
    hostPath:
      path: /run/nvidia/mps/nvidia.com/gpu/log
      type: DirectoryOrCreate
```

**Without the pipe/log wiring**, CUDA silently falls back to direct GPU
access — no MPS, no limit, no error message anywhere. A test that omitted
this wiring allocated **16x its configured limit** with zero errors, in a
real, verified test. The identical pod, with the pipe/log mounts added,
correctly failed with a CUDA out-of-memory error right at the configured
limit.

**How to verify enforcement is actually working, not just assumed**:
deliberately try to exceed the configured limit from inside the pod (e.g. a
small PyTorch script allocating in a loop) and confirm it fails with a CUDA
OOM error at approximately the configured limit — not far beyond it. Don't
trust the env var alone; test it.

> **Note on hostPath**: the exact host path (`/run/nvidia/mps/nvidia.com/gpu`)
> depends on how your cluster's MPS control daemon is configured/deployed —
> confirm it against your own cluster's MPS daemon pod's volume mounts
> before assuming this path is correct for you.

## How KAI schedules (but doesn't enforce) `gpu-memory`

KAI bin-packs `gpu-memory` requests against a physical GPU's advertised
memory capacity — this is a **scheduling/admission-time** decision. KAI
itself does not enforce the limit at runtime; that's entirely the job of
`CUDA_MPS_PINNED_DEVICE_MEM_LIMIT` above. If you skip the MPS wiring, KAI
will still correctly *schedule* the pod as if it only uses the declared
memory, but nothing stops the process from actually using more — which can
starve or crash co-scheduled MPS clients on the same physical GPU.
