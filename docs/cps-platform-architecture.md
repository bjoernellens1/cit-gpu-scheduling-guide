# CPS Compute Platform Architecture

Status: proposed
Date: 2026-09-02

## Scope

This document defines how the CPS university compute platform should use NVIDIA KAI Scheduler as the single scheduling-policy layer for research, teaching, interactive sessions, remote desktops and submitted batch workloads.

It complements the JupyterHub-side platform design by specifying scheduler-facing responsibilities and rollout constraints.

## Architecture

```text
                           University Keycloak
                                  |
                                  v
                              JupyterHub
                                  |
                       authenticated identity
                                  |
                                  v
                    CPS Compute Controller
                      /          |          \
                     /           |           \
            queue reconciler  usage/quota   job submitter
                   |              |              |
                   +--------------+--------------+
                                  |
                                  v
                       KAI queue hierarchy
                                  |
                      allocation -> user
                                  |
            +---------------------+---------------------+
            |                     |                     |
       interactive              desktop                batch
       Jupyter pods         VNC / Xpra pods       Jobs / PodGroups
            |                     |                     |
            +---------------------+---------------------+
                                  |
                                  v
                           KAI Scheduler
                 hierarchical quota / fair share
                    time-based fair share
                      borrowing / reclaim
                    gang / topology / GPU
                                  |
                                  v
                        NVIDIA GPU Operator
                                  |
                       GPU / MPS / CPU nodes

            Kubernetes + KAI + accounting events
                                  |
                                  v
                            cps-notifier
                 /          |          |          \
             Hub/Lab      Gotify      SMTP       Teams
```

## Scheduler ownership

KAI is the sole production owner of:

- queue hierarchy
- resource guarantees
- over-quota borrowing
- queue fairness
- time-based fair share
- workload priority
- reclaim/preemption policy
- gang scheduling
- topology-aware scheduling
- GPU placement/sharing policy

Do not introduce Kueue as a second admission/fairness layer unless there is a future, explicitly validated integration where one component is intentionally policy-neutral.

## Identity-to-fairness model

KAI currently schedules queues, not authenticated JupyterHub identities directly. Therefore CPS should materialize authenticated users into the queue tree.

Target hierarchy:

```text
cps
├── research
│   ├── <project>
│   │   ├── <stable-user-id>
│   │   │   ├── interactive
│   │   │   ├── desktop
│   │   │   ├── batch
│   │   │   └── opportunistic
│   │   └── ...
│   └── ...
├── teaching
│   ├── <course>
│   │   ├── <stable-user-id>
│   │   │   ├── interactive
│   │   │   └── batch
│   │   └── ...
│   └── ...
└── services
```

This ensures all workloads for one user share the same fair-share ancestry. It prevents a user from receiving multiple independent shares by launching Jupyter, a desktop and several batch jobs simultaneously.

User-level fairness is an upstream KAI roadmap item, so this queue materialization is an intentional compatibility layer that can be simplified later if KAI grows a mature native user accounting primitive.

## Queue reconciliation

A `cps-compute-controller` should reconcile queues from declarative allocation state.

Inputs:

- stable authenticated user ID
- JupyterHub/Keycloak groups
- project/course allocation mapping
- workload classes allowed for each allocation
- quota/priority/over-quota policy
- course lifecycle and scheduled reservations

Properties:

- idempotent
- lazy user-queue creation is allowed
- deterministic queue names
- no user-controlled parent/priority selection
- safe garbage collection for inactive users/expired courses

## Workload metadata contract

Every CPS-managed pod/job must include:

```yaml
metadata:
  labels:
    cps.unileoben.ac.at/user: u-<stable-id>
    cps.unileoben.ac.at/allocation: <project-or-course>
    cps.unileoben.ac.at/workload-class: <class>
    cps.unileoben.ac.at/source: <source>
    kai.scheduler/queue: <resolved-queue>
spec:
  schedulerName: kai-scheduler
```

The readable username/email may be stored as annotations or in the accounting database, but should not be the canonical scheduler identity.

## Workload classes

### Interactive

Jupyter sessions. Queueing is allowed; active sessions should not normally be preempted.

### Desktop

Remote desktop sessions. CPU and GPU desktops are scheduler-managed workloads and consume the same user allocation as notebooks/jobs.

### Batch

Normal training/compute jobs. May be made checkpointable/preemptible according to project policy.

### Opportunistic

Sweeps/backfill. No or minimal guaranteed quota; intended to consume idle capacity and be reclaimed first.

### Services

Persistent infrastructure/inference workloads with explicit reserved policy.

## Remote desktops

### CPU path

Use a lightweight desktop plus TigerVNC/noVNC (or an equivalent thin VNC stack). No GPU resource is requested.

### GPU path

Use Xpra with VirtualGL and an NVIDIA GPU allocated through KAI. Hardware acceleration must be verified with a conformance test rather than inferred from successful desktop startup.

The GPU desktop must:

- request/receive the GPU through the scheduler
- expose the same user/project storage as other profiles
- count against user concurrency and GPU-hour accounting
- be non-preemptible by default once active
- expose authenticated browser access only

## Submit-as-Job contract

Interactive Jupyter servers should not run long training merely because they happen to own a GPU.

The platform should provide a JupyterHub OAuth-authenticated job submission service that generates trusted KAI workloads.

Supported initial intents:

- CPU job
- shared/MPS GPU job
- exclusive full-GPU job
- multi-GPU gang-scheduled training
- opportunistic sweep/ablation

The submitter owns:

- queue assignment
- priority selection
- resource validation
- identity labels
- project/storage mounts
- immutable image reference
- provenance metadata
- PodGroup/gang semantics where needed

Notebook pods should not receive broad RBAC allowing arbitrary queue/priority/job creation.

## Fair share vs quota

KAI queue fairness controls contested allocation. CPS usage accounting controls hard user/project budgets.

Examples of hard policy outside KAI fair-share:

- maximum concurrent full GPUs per user
- maximum interactive GPU sessions
- rolling GPU-hour budget
- course-specific shared-GPU budget
- storage limits

The submitter/spawner checks hard policy before creating work; admitted work then competes through KAI.

## Metrics and dashboard integration

Use KAI's queue fair-share and usage metrics, Kubernetes state, DCGM/Prometheus and storage metrics to build a normalized per-user dashboard.

Admin view remains Grafana. User view should expose only their own allocation:

- active sessions
- queued/running/completed jobs
- waiting reason
- queue/fair-share status
- GPU-hour consumption
- concurrent GPU use
- storage usage
- notifications

## Notification architecture

Add `cps-notifier` as a separate event/state service.

Inputs:

- Kubernetes Job/Pod state
- KAI PodGroup/events and scheduler explanations
- JupyterHub session lifecycle
- quota/accounting events
- storage threshold events

The notifier reduces raw events into durable domain events and stores them before fan-out.

Initial channels:

1. Hub/dashboard inbox
2. JupyterLab notification extension
3. Gotify
4. SMTP email
5. Microsoft Teams

Gotify is the preferred self-hosted push transport. Use an application token owned by `cps-notifier`; do not put Gotify application tokens inside user jobs. If per-user Gotify routing is needed, store destinations/token references in platform secrets/preferences rather than job manifests.

Teams should use a currently supported mechanism such as Workflows/webhooks or a bot, not deprecated legacy connectors.

Suggested event reduction:

```text
raw pod/KAI events
       |
       v
workload state reducer
       |
       +-- submitted
       +-- queued
       +-- started
       +-- completed
       +-- failed
       +-- preempted
       +-- cancelled
       +-- waiting reason changed (rate limited)
       |
       v
notification outbox
       |
       +-- hub/lab
       +-- gotify
       +-- smtp
       +-- teams
```

Requirements:

- deduplication
- idempotent transport delivery
- rate limiting for repeated waiting reasons
- retry with bounded backoff
- durable unread history
- per-event/per-channel user preferences
- no scheduler/job dependency on notifier availability

## KAI upgrade/evolution work

Before enabling new policy, audit the live cluster against the current stable KAI line rather than assuming this guide's earlier v0.12.10 behavior remains exact.

Validate at least:

- current KAI version and CRDs
- queue schema and hierarchy migration
- current PriorityClass/preemptibility behavior
- time-based fair-share configuration and metrics
- whole-GPU allocation
- MPS/fractional GPU allocation
- exclusive workload after MPS workload
- multi-GPU PodGroup/gang scheduling
- scheduler explainability events
- Prometheus queue fair-share/usage metrics

Keep the existing field-tested MPS enforcement checks. A shared-GPU request is not considered enforced until an intentional over-allocation test fails near the configured memory limit.

## Rollout order

1. Inventory current production KAI/GPU Operator/JupyterHub configuration.
2. Add scheduler conformance tests and capture a known-good baseline.
3. Upgrade KAI in staging/test path and re-run conformance.
4. Introduce declarative allocation policy and user queue controller.
5. Route existing Jupyter profiles through the new identity/queue hierarchy.
6. Add CPU VNC and GPU Xpra/VirtualGL desktop profiles.
7. Add job submitter for CPU/single-GPU first.
8. Add multi-GPU/gang and sweep templates.
9. Add usage/accounting dashboard.
10. Add notifier with Hub/Lab + Gotify first, then SMTP and Teams.
11. Enable time-based fair share only after simulation/shadow evaluation against representative teaching/research demand.
12. Add course calendar reservations/pre-pull/lifecycle automation.

## Acceptance criteria

- All managed workloads resolve to a stable user and allocation.
- Jupyter, desktop and batch usage for one user share one fair-share parent.
- Queue/priority escalation cannot be performed from an ordinary notebook.
- Active interactive/desktop sessions are protected according to policy.
- Batch workloads continue when Jupyter servers stop.
- KAI waiting reasons are visible to users.
- Job completion/failure produces one durable notification and independent channel deliveries.
- Gotify, email or Teams outage cannot affect scheduling/job execution.
- Hardware-accelerated Xpra is validated on GPU nodes; CPU desktop remains lightweight.
- Time-based fair-share rollout has reproducible simulation/shadow evidence before affecting production users.
