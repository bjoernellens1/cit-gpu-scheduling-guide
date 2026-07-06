# Gang scheduling for multi-GPU / multi-pod training

Distributed training (PyTorch DDP, TensorFlow `MultiWorkerMirroredStrategy`)
needs **all worker pods scheduled together, or not at all** — a partial
launch (e.g. 2 of 4 workers admitted, the rest Pending) will hang
indefinitely waiting for peers that will never start, wasting the GPUs the
admitted pods are holding.

## Why you need an explicit `PodGroup`

KAI's podgrouper webhook auto-creates a `PodGroup` for pods it schedules,
but for a plain multi-pod `batch/v1` Job, this defaults to `minMember: 1`
per pod — i.e. **no real gang-scheduling guarantee**, each pod is admitted
independently. This was confirmed by reading the upstream KAI Scheduler
source (`pkg/podgrouper/podgrouper/plugins/job/job_grouper.go` in the
NVIDIA/KAI-Scheduler v0.12.10 tag): the Job grouper computes a separate
PodGroup per pod unless "legacy" grouping applies, and `MinAvailable`
defaults to 1 either way.

To get real "all or nothing" gang scheduling, create your own `PodGroup`
and attach every worker pod to it explicitly:

```yaml
apiVersion: scheduling.run.ai/v2alpha2
kind: PodGroup
metadata:
  name: my-training-job-pg
  namespace: <your-namespace>
spec:
  minMember: 4        # total pod count across the whole job
  queue: batch
  priorityClassName: low-priority
```

```yaml
# on each worker pod:
metadata:
  annotations:
    pod-group-name: my-training-job-pg
```

This is the scheduler-level gang membership mechanism, not just a
status-aggregation detail — the scheduler core itself reads the
`pod-group-name` annotation to decide admission. It's also the pattern
KAI's own e2e test helpers use for externally-managed PodGroups
(`test/e2e/modules/resources/rd/pod_group/pod_group.go` in the upstream
repo), so it's a sanctioned, not a workaround, approach.

## Rendezvous pattern: headless Service + Indexed Job

The standard way to give each worker a stable rank/address on Kubernetes,
without a separate StatefulSet:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-training-job
spec:
  clusterIP: None            # headless
  selector:
    job-name: my-training-job
  ports:
    - name: torchrun
      port: 29500
      targetPort: 29500
---
apiVersion: batch/v1
kind: Job
metadata:
  name: my-training-job
spec:
  completionMode: Indexed     # gives each pod a stable ordinal via JOB_COMPLETION_INDEX
  parallelism: 4
  completions: 4
  template:
    spec:
      subdomain: my-training-job   # required for per-ordinal DNS to register
      containers:
        - name: worker
          command: ["torchrun",
            "--nnodes=4", "--nproc_per_node=1",
            "--node-rank=$(JOB_COMPLETION_INDEX)",
            "--master-addr=my-training-job-0.my-training-job",
            "--master-port=29500",
            "train_ddp.py"]
          env:
            - name: JOB_COMPLETION_INDEX
              valueFrom:
                fieldRef:
                  fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
```

`spec.subdomain` matching a headless Service selecting these pods is what
makes `<job-name>-<ordinal>.<subdomain>` resolve in cluster DNS — without
it, the pod's auto-assigned hostname alone does not register.

## Exclusive GPU allocation, not MPS-shared, for real multi-GPU jobs

Real inter-GPU communication (NVLink/PCIe peer-to-peer, used by NCCL
all-reduce) requires **exclusive** GPU allocation
(`resources.limits."nvidia.com/gpu"`), not MPS-shared/fractional
(`gpu-memory`) — the two aren't a supported combination for NCCL
collectives. See [gpu-sharing-modes.md](gpu-sharing-modes.md) for the full
allocation-mode comparison.

**Why this matters — real measured numbers**: a real 2-GPU intra-pod test
(genuine NVLink/PCIe P2P, exclusive allocation) measured **~100-230 GB/s**
effective bandwidth. The same total GPU count, but split across separate
pods forced onto socket/TCP transport instead of direct P2P, measured only
**~0.9-1.4 GB/s** — roughly two orders of magnitude slower. If your
distributed training job needs real speedup from multiple GPUs, exclusive,
same-pod (or gang-scheduled, topology-aware) allocation is not optional —
it's the difference between the job actually working and silently running
100x slower than expected while still "working."

## What `minMember` does and doesn't guarantee

`minMember` on the `PodGroup` guarantees all member pods are admitted
together or not at all. It does **not** by itself guarantee pods land on
the same physical node or within the same NVLink domain — for multi-pod
jobs where intra-node bandwidth matters, you may also need a topology
constraint (`spec.topologyConstraint` on the `PodGroup`, if your KAI
version's CRD supports it) or explicit pod affinity. Verify this for your
own cluster/topology rather than assuming `minMember` alone is sufficient
for locality.

## Job-level restart semantics

Use `restartPolicy: Never` (not `OnFailure`) for gang-scheduled workers — a
mid-training failure on any single rank should bring down the whole
distributed group, not have kubelet silently restart one container in
place while its peers keep running (which would desync the process
group). Recovery should happen at the Job level: fix the issue, delete and
re-apply the Job, and resume from your last checkpoint on shared storage
(see [storage.md](storage.md)).
