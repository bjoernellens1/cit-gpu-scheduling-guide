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

## Gotcha: `EXCLUSIVE_PROCESS` compute mode collides with plain-exclusive pods on an MPS-shared cluster

**Symptom**: a plain-exclusive GPU pod (`nvidia.com/gpu: N`, no fractional
annotation, never routes through MPS) fails deterministically with
`RuntimeError: CUDA error: CUDA-capable device(s) is/are busy or
unavailable`, even in complete isolation (a single pod, no other workload,
no other process shown by `nvidia-smi --query-compute-apps` other than
`nvidia-cuda-mps-server`). `nvidia-smi` itself (query-only) still works
fine, and ECC/throttle status is clean — this is not a hardware fault.

**Root cause**: if your cluster runs GPUs in `Exclusive_Process` compute
mode (commonly required/assumed for an MPS control daemon's own server
context), only **one** CUDA context is allowed per GPU at a time. An MPS
control daemon's `nvidia-cuda-mps-server` process holds that one slot
**continuously, independent of whether any MPS client is currently
connected** — it doesn't release the slot just because no fractional job
happens to be running right now. If your scheduler treats GPUs as an
elastic pool where any physical GPU can serve either a fractional
(MPS-shared) or a plain-exclusive pod at different times, a plain-exclusive
pod scheduled onto a GPU with *no active MPS client* still collides with
the resident server process and fails. This is a structural
sharing-model/compute-mode mismatch, not a rare race tied to pod deletion,
crashes, or any specific lifecycle event — it will recur on any GPU the
scheduler hands from MPS-shared use back to exclusive use, for as long as
the MPS daemon itself is running on that node in `Exclusive_Process` mode.

(An earlier version of this note attributed the symptom to force-deleting
a pod leaving a stale driver lock. That was disproved on a live cluster:
the same failure recurred on a node where every prior GPU-using pod had
exited cleanly days earlier, with no force-delete anywhere in between.
Restarting the MPS daemon "fixed" it by coincidence — the restart
recreates the server's context, transiently freeing the slot — not because
force-deletion was ever the actual trigger.)

**Fix**: switch GPU compute mode from `Exclusive_Process` to `Default`.
`Default` mode allows multiple CUDA contexts on one GPU simultaneously, so
the resident MPS server and a separate exclusive-mode client no longer
fight over a single slot. This was verified live to not weaken MPS memory
enforcement (`CUDA_MPS_PINNED_DEVICE_MEM_LIMIT` still OOMs a client at
exactly its configured limit) and to not create a new double-booking risk
for well-behaved pods, *provided* your scheduler's GPU reservation
mechanism tracks a real Kubernetes extended resource that kubelet's own
device manager enforces (so a second pod can't get the same physical GPU
regardless of which scheduler placed it — verify this holds for your
specific scheduler/sharing setup before relying on it).

**Persistence gotcha**: setting compute mode with a one-off `nvidia-smi -c
DEFAULT` is not durable if your MPS daemon runs as a container/pod — most
MPS control daemon implementations reassert `Exclusive_Process` as part of
their own startup, so the setting reverts on every restart (pod delete,
node reboot, image upgrade). A one-shot "wait for the daemon to be ready,
then set `Default` once" approach can also lose a race if any readiness
marker your wrapper checks persists across restarts (e.g. on a hostPath
volume) — the daemon's own asynchronous mode-setting can still run after
your one-shot check passes, and wins. A background loop that re-asserts
`Default` every few seconds for the daemon container's whole lifetime
sidesteps this and self-heals against any future reassertion:

```sh
your-mps-control-daemon-binary &
daemon_pid=$!
while kill -0 "$daemon_pid" 2>/dev/null; do
  nvidia-smi -c DEFAULT >/dev/null 2>&1
  sleep 5
done &
wait "$daemon_pid"
```

Check whether your GPU/device-plugin stack exposes a supported config
option for this before reaching for a wrapper like the above — at time of
writing, neither NVIDIA GPU Operator's `ClusterPolicy`/`NVIDIADriver` CRDs
nor the device-plugin's MPS sharing config schema expose a compute-mode
setting, so this had to be handled at the container-command level instead.

## How KAI schedules (but doesn't enforce) `gpu-memory`

KAI bin-packs `gpu-memory` requests against a physical GPU's advertised
memory capacity — this is a **scheduling/admission-time** decision. KAI
itself does not enforce the limit at runtime; that's entirely the job of
`CUDA_MPS_PINNED_DEVICE_MEM_LIMIT` above. If you skip the MPS wiring, KAI
will still correctly *schedule* the pod as if it only uses the declared
memory, but nothing stops the process from actually using more — which can
starve or crash co-scheduled MPS clients on the same physical GPU.
