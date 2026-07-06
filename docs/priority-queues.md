# Priority queues and preemption

KAI Scheduler arbitrates GPU access across competing workloads using two
layered concepts: **Queues** (quota/burst accounting) and
**PriorityClasses** (preemption ordering).

## Example queue setup (adapt names/values to your cluster)

A common pattern for a cluster serving multiple workload tiers — e.g.
course/teaching work, interactive research, and best-effort batch:

```yaml
apiVersion: scheduling.run.ai/v2
kind: Queue
metadata:
  name: courses
  namespace: kai-scheduler
spec:
  parentQueue: default-parent-queue
  resources:
    gpu:
      quota: 4          # guaranteed GPUs for this queue
      limit: -1         # -1 = can burst above quota into idle capacity
      overQuotaWeight: 1
    cpu:
      quota: -1
      limit: -1
      overQuotaWeight: 1
    memory:
      quota: -1
      limit: -1
      overQuotaWeight: 1
---
apiVersion: scheduling.run.ai/v2
kind: Queue
metadata:
  name: batch
  namespace: kai-scheduler
spec:
  parentQueue: default-parent-queue
  resources:
    gpu:
      quota: 0          # no guaranteed quota -- burst-only into idle capacity
      limit: -1
      overQuotaWeight: 10
    cpu:
      quota: 0
      limit: -1
      overQuotaWeight: 10
    memory:
      quota: 0
      limit: -1
      overQuotaWeight: 10
```

**Every resource dimension needs an explicit block.** KAI defaults any
dimension you don't specify to `quota: 0, limit: 0` — a hard cap, not just
an unguaranteed 0. A real job was rejected with `OverLimit: batch quota has
reached the allowable limit of CPU cores. Limit is 0 cores` despite the
`gpu` dimension being correctly configured to burst via `limit: -1`,
because `cpu`/`memory` blocks were simply missing. If you only care about
GPU quota, you still need to give `cpu`/`memory` a real (even if
unlimited, `-1`) value.

## PriorityClasses and the preemption threshold gotcha

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 90
globalDefault: false
description: "Highest priority, can reclaim resources from others."
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 10
globalDefault: false
description: "Lowest priority, preemptible by higher tiers."
```

**KAI treats any PriorityClass with `value >= 100` as non-preemptible.**
A non-preemptible workload is hard-capped at its queue's guaranteed quota —
it can **never** burst over quota, regardless of `overQuotaWeight`. If your
lowest-priority "burst into idle capacity" queue design depends on
preemptibility (quota `0`, relying entirely on bursting), and you set its
PriorityClass to a value >= 100 (a natural instinct — e.g. `1000` "so it's
clearly the lowest"), the workload becomes **permanently unschedulable**:
KAI will reject it with something like
`NonPreemptibleOverQuota: Workload requested 0.05 GPUs, but queue quota is
0 GPUs`, and it will never burst no matter how much idle capacity exists.

This was verified live: a queue's entire burst-only design was silently
broken from day one by using PriorityClass values of 1000/5000/10000 (a
natural, seemingly-harmless choice for clear separation). The fix was
rescaling to values that keep the same relative order but all stay under
100 (e.g. 10/50/90).

**Rule of thumb**: keep every PriorityClass used with KAI Scheduler under
100 unless you specifically want that tier to be non-preemptible and
hard-capped at quota — and if you do want that, make sure that tier's
queue actually has a non-zero quota, or it will never schedule at all.

## Submitting into a queue with the right priority

```yaml
metadata:
  labels:
    kai.scheduler/queue: batch
spec:
  schedulerName: kai-scheduler
  priorityClassName: low-priority
```

Both the queue label and `priorityClassName` need to be set — a pod
without an explicit queue label falls into whatever your cluster's default
queue is, which may not have the quota/priority behavior you expect.
