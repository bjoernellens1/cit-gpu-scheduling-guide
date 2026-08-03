# Scheduling PyTorch & TensorFlow GPU Jobs on KAI Scheduler

A practical, field-tested guide to running GPU training jobs — single-GPU,
MPS-shared/fractional, multi-GPU gang-scheduled, and multi-node distributed —
on a Kubernetes cluster managed by [NVIDIA KAI Scheduler](https://github.com/NVIDIA/KAI-Scheduler).

Every fact and gotcha in here was verified live against a real cluster, not
copied from documentation. Where something is a NVIDIA KAI implementation
detail (not just generic Kubernetes), the source is cited.

## Assumptions

This guide assumes:

- **KAI Scheduler v0.12.10+** installed and running (`schedulerName:
  kai-scheduler`, `Queue`/`PodGroup` CRDs available).
- **NVIDIA GPU Operator** running the device plugin in **plain
  (non-MIG) mode** — i.e. `nvidia.com/gpu` reports the real physical GPU
  count per node, not a MIG-partitioned or device-plugin-replicated count.
  MIG-based sharing is a different model with different tradeoffs and is
  out of scope here.
- **NVIDIA MPS** (Multi-Process Service) available on GPU nodes, used for
  fractional/shared-GPU workloads via KAI's `gpu-memory` annotation.
- A CNI that doesn't put multi-node NCCL traffic on a fast path by default
  (e.g. Flannel VXLAN) — if your cluster has RDMA/InfiniBand or a CNI with
  native fast networking, skip the `hostNetwork` advice in the multi-node
  section.

If your cluster differs (MIG-based sharing, a different scheduler,
RDMA fabric), the concepts still mostly apply but some specifics won't.

## Quick start

| I want to... | Use | Queue semantics |
|---|---|---|
| Run a single-GPU job that doesn't need to share the card | [`examples/single-gpu/pytorch-exclusive.yaml`](https://github.com/bjoernellens1/cit-gpu-scheduling-guide/blob/main/examples/single-gpu/pytorch-exclusive.yaml) | `nvidia.com/gpu: 1` |
| Run a small/cheap job that can share a GPU with others | [`examples/single-gpu/pytorch-mps-shared.yaml`](https://github.com/bjoernellens1/cit-gpu-scheduling-guide/blob/main/examples/single-gpu/pytorch-mps-shared.yaml) | `gpu-memory` annotation |
| Train with real multi-GPU speedup (DDP / MultiWorkerMirroredStrategy) on one node | [`examples/multi-gpu/`](https://github.com/bjoernellens1/cit-gpu-scheduling-guide/tree/main/examples/multi-gpu) | gang-scheduled `PodGroup` |
| Scale training across multiple physical nodes | [`examples/multi-node/`](https://github.com/bjoernellens1/cit-gpu-scheduling-guide/tree/main/examples/multi-node) | gang-scheduled `PodGroup` + NCCL tuning |
| Run a hyperparameter sweep/ablation (many related jobs), or skip writing YAML entirely | [`ablator`](https://github.com/bjoernellens1/ablator) — see [ablator.md](ablator.md) | generates the manifests below for you |

## Table of contents

- [gpu-sharing-modes.md](gpu-sharing-modes.md) — exclusive vs.
  MPS-shared allocation, and the **MPS enforcement gotcha** that silently
  disables VRAM limits if you get it wrong
- [gang-scheduling.md](gang-scheduling.md) — why you need an
  explicit `PodGroup`, and how DDP/MultiWorkerMirroredStrategy rendezvous
  works on Kubernetes
- [multi-node-networking.md](multi-node-networking.md) — NCCL
  tuning for non-RDMA Ethernet fabrics, with real measured before/after
  numbers
- [priority-queues.md](priority-queues.md) — queues, priority
  classes, and the preemption threshold gotcha
- [storage.md](storage.md) — checkpoint/dataset storage patterns
  that survive rescheduling and preemption
- [ablator.md](ablator.md) — using
  [`ablator`](https://github.com/bjoernellens1/ablator) to generate and
  submit these manifests for you, for sweeps/ablations or when you'd
  rather not hand-write YAML
- [cluster-access.md](cluster-access.md) — getting `kubectl` access set up
  against a KAI Scheduler cluster

## Examples

```
examples/
├── single-gpu/
│   ├── pytorch-exclusive.yaml         # whole-GPU, PyTorch
│   ├── pytorch-mps-shared.yaml        # fractional/MPS-shared, PyTorch
│   ├── tensorflow-exclusive.yaml      # whole-GPU, TensorFlow
│   └── tensorflow-mps-shared.yaml     # fractional/MPS-shared, TensorFlow
├── multi-gpu/
│   ├── pytorch-ddp-2gpu.yaml          # 2-GPU DDP, single node
│   ├── pytorch-ddp-multi-pod.yaml     # 4+ GPU DDP, gang-scheduled across pods
│   └── tensorflow-multiworker.yaml    # TensorFlow MultiWorkerMirroredStrategy
└── multi-node/
    ├── pytorch-ddp-multi-node.yaml    # DDP spanning multiple physical nodes
    └── tensorflow-multiworker-multi-node.yaml
```

Every manifest is a **template** — replace `your-registry/your-image`,
`<your-namespace>`, storage class names, and queue/priority-class names
with your own cluster's values. Placeholders are marked clearly in comments.

Browse the full `examples/` tree on
[GitHub](https://github.com/bjoernellens1/cit-gpu-scheduling-guide/tree/main/examples).

## Contributing

Corrections and additions welcome, especially real measured numbers from
other clusters/fabrics — this guide is only as good as what's actually been
verified.

## License

MIT — see
[LICENSE](https://github.com/bjoernellens1/cit-gpu-scheduling-guide/blob/main/LICENSE).
