# Using `ablator` instead of hand-writing manifests

Everything else in this guide teaches the underlying Kubernetes/KAI
mechanics — what a `Job`/`PodGroup` manifest needs to contain and why.
[`ablator`](https://github.com/bjoernellens1/ablator) is a separate,
sibling tool that *generates and submits* those manifests for you from a
small TOML config plus a JSON "spec" describing one or more runs. It's
worth reaching for instead of copying one of this repo's YAML examples
when either of these is true:

- **You want to run many related jobs, not one.** A hyperparameter sweep
  or ablation (e.g. "same training script, five learning rates") is one
  `ablator` spec with multiple `arms`, not five hand-edited copies of a
  `Job` manifest. `ablator` also gives you a shared job queue, retry-once
  semantics, and `ablator collect` to gather result files across all runs
  of a sweep in one command — none of which exist if you're just
  `kubectl apply`-ing manifests one at a time.
- **You don't want to hand-write YAML at all.** `ablator`'s optional TUI
  (`pip install ablator[tui]`) walks you through configuration
  interactively and gives you a k9s-style live view of the queue and
  running jobs — see below.

`ablator` isn't limited to Kubernetes — it also dispatches to bare-metal
hosts over SSH with the same job queue — but this section only covers its
Kubernetes dispatch path against this cluster, since that's what overlaps
with this guide.

For the full setup walkthrough — getting Rancher/Authentik access,
downloading a kubeconfig, installing `kubectl`, installing `ablator`
itself, and running a first real job — see ablator's own
[`docs/cluster-setup.md`](https://github.com/bjoernellens1/ablator/blob/main/docs/cluster-setup.md).
That doc already covers this in detail against this exact cluster; it
isn't duplicated here.

## Why this matters for KAI specifically

`ablator`'s Kubernetes backend isn't a generic "submit some YAML to
whatever cluster" tool bolted on after the fact — it was built with this
cluster's KAI Scheduler conventions in mind, the same ones this guide
documents elsewhere. A `[machines.<name>]` block with `backend = "k8s"`
generates a `Job` manifest that sets, from a handful of plain config
fields:

| ablator config field | KAI manifest field it produces |
|---|---|
| `namespace` | the Job's `metadata.namespace` |
| `kai_queue` | `kai.scheduler/queue` label (on the Job template *and* pod template — see [docs/priority-queues.md](priority-queues.md)) |
| `priority_class` | `priorityClassName` |
| `scheduler_name` (defaults to `kai-scheduler`) | `schedulerName` |
| `gpu_count` | `resources.limits."nvidia.com/gpu"` |
| `image` / `image_pull_secret` | container image / `imagePullSecrets` |
| `extra_volumes` | `volumes` + `volumeMounts` |

In other words: the exact fields this guide's raw examples ask you to
fill in by hand (`kai.scheduler/queue`, `schedulerName: kai-scheduler`,
`priorityClassName`, `nvidia.com/gpu`) are exactly what `ablator` derives
from config instead. If you already understand *why* those fields matter
from reading this guide's other docs, `ablator` just saves you from
retyping them correctly on every job.

One cluster-specific gotcha carried over faithfully from this guide: jobs
dispatched to the cluster's default `batch` queue/`kai-batch-low`
priority class have zero guaranteed quota and can be preempted at any
time (see [docs/priority-queues.md](priority-queues.md)). `ablator`
treats a preempted/evicted pod as a job failure (one automatic retry,
then quarantine) rather than a live migration — so, same as any raw
manifest on this queue, **your training script still needs to checkpoint
and resume on its own.** `ablator` doesn't change that requirement, it
just doesn't hide it either.

## Worked example: the same job, two ways

This guide's [`examples/single-gpu/pytorch-exclusive.yaml`](https://github.com/bjoernellens1/cit-gpu-scheduling-guide/blob/main/examples/single-gpu/pytorch-exclusive.yaml)
is a whole-GPU, single-pod PyTorch training `Job` on the `batch` queue.
Here's the equivalent expressed as an `ablator` config + spec instead of
a raw manifest (based on ablator's actual
[`examples/pytorch-generic.toml`](https://github.com/bjoernellens1/ablator/blob/main/examples/pytorch-generic.toml)).

**The manifest way** (this guide): edit the YAML directly — namespace,
image, `kai.scheduler/queue`, `priorityClassName`, `nvidia.com/gpu`,
volumes — and `kubectl apply -f pytorch-exclusive.yaml`.

**The ablator way**: config once, then describe runs as data.

`~/.config/ablator/config.toml` (trimmed from `pytorch-generic.toml`):

```toml
[queue]
path = "/home/YOURUSER/ablator/queue/queue.jsonl"
model_path_template = "output/{name}_{arm}"
result_glob = "{model_path}/report.json"

[machines.local]
hostname_patterns = ["*"]

[machines.a100cluster]
backend = "k8s"
namespace = "<your-namespace>"   # your Rancher-assigned namespace, see docs/cluster-access.md
kai_queue = "batch"
priority_class = "kai-batch-low"
gpu_count = 1
image = "your-registry/your-image:latest"

[types.train]
cwd = "/workspace"
command = ["python", "train.py", "--checkpoint-dir=/checkpoints",
           "--data-dir=/data", "--resume-from-latest"]
```

`myjob.json` (one run — no sweep yet, just the single-job equivalent):

```json
{
  "name": "myjob",
  "parallel": true,
  "base": {"type": "train", "machine": "a100cluster"},
  "arms": [
    {"id": "run1", "extra_args": ""}
  ]
}
```

Enqueue and run it:

```bash
ablator plan myjob.json
ablator run
ablator status myjob
ablator collect myjob
```

The payoff shows up once you want more than one run: turning this into a
sweep is just adding `arms` (e.g. different `extra_args` per learning
rate) to the same spec — no copy-pasting manifests, and `ablator status`/
`ablator collect` then work across the whole sweep at once.

## The friendliest on-ramp: the TUI

If you've read this far and understand *why* the fields above matter but
don't want to write TOML or a spec JSON by hand yet, run `ablator` with
no arguments from a terminal (install with `pip install ablator[tui]`).
On first run it interactively prompts for namespace, KAI queue, priority
class, training image, GPU count, and an optional shared PVC, then drops
you into a k9s-style full-screen view of the job queue and running jobs
(including live pod status/log tail for Kubernetes-dispatched jobs), with
a screen to list/switch kubeconfig contexts if you work against more than
one cluster. See ablator's
[`docs/cluster-setup.md`](https://github.com/bjoernellens1/ablator/blob/main/docs/cluster-setup.md#recommended-guided-setup-via-the-tui)
for the full walkthrough.
