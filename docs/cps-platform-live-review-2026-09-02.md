# CPS Compute Platform — live cluster correctness review

Date: 2026-09-02

This is the second correctness pass for the proposed CPS JupyterHub/KAI compute-platform evolution. It compares the design against the authoritative GitOps repository `mul-cps/cps-gpu-cluster` rather than the legacy standalone JupyterHub repository.

## Verified production baseline

### KAI

The live GitOps pin is KAI Scheduler **v0.16.3**. The current policy is:

| Queue | GPU deserved quota | over-quota weight | Current role |
|---|---:|---:|---|
| `courses` | 4 | 1 | course/student interactive, predominantly MPS-shared |
| `phd-interactive` | 2 | 10 | research/PhD interactive, including whole GPU |
| `batch` | 0 | 2 | opportunistic batch/burst |
| `ci` | 0 | 1 | lowest-weight opportunistic GPU CI |
| `default` | unlimited/fallback | 10 | fallback/unlabeled |

CPU and memory dimensions are explicitly configured because an omitted resource dimension becomes a hard zero limit in the KAI queue model used by the cluster.

Current priority classes are course 90, research interactive 50 and batch 10. These values were deliberately rescaled below the old KAI non-preemptible threshold after a real incident; treat them as validated v0.16.3 policy and re-qualify semantics on any target upgrade.

### JupyterHub

The live Hub is a Fleet-managed Zero-to-JupyterHub deployment, not the old DockerSpawner repository.

Current important behavior:

- OIDC through the shared Dex broker;
- `preferred_username` is deliberately preserved across CPS/CIT identity sources because existing usernames already key user storage continuity;
- OIDC groups are managed by JupyterHub;
- named servers enabled, limit 3/user;
- custom card/profile selector is already implemented;
- GPU profiles already set `schedulerName: kai-scheduler`;
- student profiles route to `courses` / `kai-course-high`;
- general/full-GPU research profiles route to `phd-interactive` / `kai-phd-interactive`;
- personal/project/scratch NFS options and power-user controls already exist.

Therefore the new platform should evolve the existing UI/spawn logic, not replace it wholesale.

### GPU sharing

The production architecture is a unified non-MIG A100 pool. KAI `gpu-memory` scheduling plus NVIDIA MPS is used for fractional/shared VRAM, while whole-GPU workloads use exclusive GPU allocation. Existing enforcement/reclaim tests are load-bearing and must remain part of qualification.

### Descheduler

A Kubernetes Descheduler runs every five minutes for batch-only capacity consolidation. Its `DefaultEvictor` threshold makes only the priority-10 batch tier reachable; current interactive/course sessions are structurally excluded.

This is not currently tenant-fairness policy, but it **is** a second eviction loop. Under a 'full KAI' evolution:

- KAI should own tenant fairness, quota reclaim and workload preemption;
- the Descheduler may temporarily remain as a narrowly scoped defragmentation mechanism;
- do not remove `LowNodeUtilization` until a KAI-only test proves interactive startup, whole/multi-GPU placement and cluster utilization do not regress;
- if retained, its victim eligibility must remain limited to intentionally opportunistic/checkpointable work.

## Design corrections from pass 2

### 1. Identity is not raw OIDC `sub`

The generic design originally preferred immutable OIDC subject as the compute key. On the live brokered setup, that can conflict with the explicit username/PVC continuity requirement.

Use a persistent internal `compute_user_id` and maintain aliases to current Hub username and upstream identity attributes. Existing users should initially be anchored to their current Hub username; future identity changes update aliases rather than create new fair-share histories.

### 2. Preserve the existing guarantees

Do not create new `teaching` and `research` roots with new guarantees while `courses` and `phd-interactive` remain active: that would temporarily double-count deserved capacity.

Two safe migration patterns should be evaluated in staging:

**A. Convert existing queues into parents** if the target KAI hierarchy semantics cleanly preserve their current resource policy:

```text
default-parent-queue
├── courses (preserve quota 4)
│   └── <course>
│       └── <user>
│           ├── interactive
│           └── batch
├── phd-interactive (preserve quota 2 / weight 10)
│   └── <allocation>
│       └── <user>
│           ├── interactive
│           ├── desktop
│           └── batch
├── batch (quota 0 / weight 2, opportunistic only)
└── ci
```

**B. Create renamed `teaching`/`research` roots and atomically transfer policy/workloads**. Never leave both old and new guaranteed roots simultaneously active with full deserved quota.

Pattern A has less policy churn and is the preferred first experiment.

### 3. Split normal research batch from opportunistic batch

The current top-level `batch` queue is intentionally quota-zero opportunistic filler. It should remain that semantic class.

The new Submit-as-Job service should distinguish:

- **normal research batch**: child of the same user/research allocation as the user's notebooks/desktops, so it shares that user's fair-share ancestry;
- **opportunistic sweep/backfill**: current/top-level zero-guarantee `batch` branch or its future renamed equivalent.

Otherwise a user could gain one research-interactive share plus a separate independent batch share.

### 4. Time-based fair share is opt-in and needs evidence

No time-based fair-share configuration was found in the current cluster repo. Introduce it only after:

- target KAI upgrade qualification;
- per-user queue hierarchy exists;
- simulator/shadow scenarios exercise research users, lecture spikes and opportunistic workloads;
- selected historical window/decay/sensitivity is documented;
- rollback to classic DRF is trivial.

KAI's time-based mechanism affects over-quota distribution; deserved quota and queue priority still take precedence. This is desirable for the CPS mix because course/reliable interactive guarantees can remain stable while surplus/batch access becomes fair over time.

### 5. Existing scheduler architecture documentation has drift

`docs/gpu/gpu-scheduling-architecture.md` in the live cluster repo still contains an older summary table showing `phd-interactive` GPU quota 1.5 / overQuotaWeight 2, while the current `kai-policy/queues.yaml` uses quota 2 / weight 10. The manifest is the current policy source and the architecture document should be corrected.

### 6. Notification routing needs a durable service

No Gotify integration exists in the live repo. Add a separate `cps-notifier`, not ad-hoc send logic in JupyterHub/jobs.

Initial channels:

- Hub/dashboard inbox;
- JupyterLab notification extension;
- Gotify;
- SMTP;
- Teams.

Gotify personal routing requires a per-user Gotify application/token (or an explicitly shared project/course application), because a Gotify application belongs to one Gotify user. One notifier application token is not arbitrary per-recipient routing.

### 7. Remote desktop must stay under KAI

CPU desktop: XFCE + TigerVNC/noVNC, no GPU.

GPU desktop: Xpra + VirtualGL, KAI GPU allocation, same user/allocation queue ancestry as Jupyter, and an accelerated OpenGL conformance test.

## Recommended target workload tree

The long-term semantic model is:

```text
allocation guarantee
    -> project/course
        -> user
            -> interactive
            -> desktop
            -> normal-batch

opportunistic
    -> user
        -> sweep/backfill
```

For the first rollout, preserve current `courses` and `phd-interactive` objects as the guaranteed roots if KAI's child-queue behavior validates cleanly.

## Upgrade boundary

The production cluster is v0.16.3, so the immediate qualification target is the current stable line (currently v0.17.0), not a large v0.12-to-current leap.

Qualification must cover:

- queue hierarchy/resource inheritance;
- priority/preemptibility semantics;
- whole GPU;
- MPS fractional memory enforcement;
- exclusive-after-MPS reuse;
- explicit PodGroup/gang scheduling;
- reclaim across current workload types;
- fair-share/queue metrics;
- waiting/explanation events;
- Descheduler interaction;
- preemption delay where useful.

## Pass-2 conclusion

The proposed platform architecture remains valid, but implementation should be **evolutionary**:

1. qualify v0.16.3 -> current stable;
2. preserve current guarantee values;
3. introduce persistent compute identity;
4. turn existing guaranteed queues into hierarchical user-aware policy rather than replacing the whole scheduler layout at once;
5. distinguish normal user batch from opportunistic batch;
6. keep the existing batch-only Descheduler until a KAI-only replacement is proven;
7. build job submission, desktop, usage/quota and notifier services around the already-working Hub/KAI integration.
