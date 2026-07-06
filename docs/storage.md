# Storage patterns for training jobs

## Split input data from checkpoints — always

Mount your read-mostly training dataset and your writable checkpoint
directory as **separate PVCs**, not subpaths of one volume:

```yaml
volumeMounts:
  - name: training-data
    mountPath: /data
    readOnly: true
  - name: checkpoints
    mountPath: /checkpoints
volumes:
  - name: training-data
    persistentVolumeClaim:
      claimName: training-data-pvc
  - name: checkpoints
    persistentVolumeClaim:
      claimName: training-checkpoints-pvc
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: training-data-pvc
spec:
  accessModes:
    - ReadOnlyMany            # RWX for concurrent multi-pod reads
  storageClassName: <your-rwx-storage-class>
  resources:
    requests:
      storage: 100Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: training-checkpoints-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: <your-rwx-storage-class>
  resources:
    requests:
      storage: 50Gi
```

Why: training code should never be able to accidentally write into the
dataset path, and a `ReadOnlyMany` dataset claim only makes sense pointing
at storage that's already populated out-of-band — a fresh, empty
dynamically-provisioned `ReadOnlyMany` volume is useless, since nothing
could ever write the dataset into it.

## Never use node-local storage for checkpoints

If your cluster offers a node-local fast-storage option (e.g. local NVMe
via a `local-path`-style StorageClass), it is genuinely faster — but it is
**not safe for checkpoints**. If your job gets preempted, evicted (by a
descheduler or priority preemption — see
[priority-queues.md](priority-queues.md)), or simply restarted, the
scheduler may place the replacement pod on a **different physical node**,
where a node-local checkpoint written by the old pod is completely
unreachable. Your job will appear to "lose" its checkpoint with no error —
it's just on a disk the new pod can't see.

**Rule**: only use shared, node-independent storage (a real network/RWX
StorageClass) for anything that must survive rescheduling. Reserve
node-local fast storage for genuinely ephemeral scratch data your job can
regenerate from scratch if lost.

## Overcommitted/thin-provisioned storage classes

Some storage backends (e.g. Longhorn with an overcommit-oriented
StorageClass) let you request a large *logical* capacity backed by much
less *real* disk, useful for capacity you don't expect to fully use.
Understand your specific storage backend's behavior here — an
overcommitted class can, if enough tenants overcommit simultaneously, run
out of real scheduling headroom even though your own PVC's requested size
looks fine. If a PVC gets stuck `Pending` with no obvious quota issue,
check whether the underlying storage backend has hit a real capacity
ceiling despite "available" logical space, not just your own request size.

## Recovery from eviction — resume, don't restart from scratch

Whatever training framework you use, implement checkpoint-resume so that
whichever pod picks the work back up — the original node or a new one
after rescheduling — finds its progress on shared storage:

```
python train.py --checkpoint-dir=/checkpoints --resume-from-latest
```

For a plain Kubernetes `Job`, if a pod is evicted mid-run (e.g. by a
descheduler consolidating nodes), Kubernetes itself — not your training
framework — recreates the pod under the Job controller; you do not need
to manually resubmit for a mid-run eviction. You **would** need to
manually resubmit if the Job exhausts its `backoffLimit` and is marked
`Failed`, or after a successful completion if you want to run it again —
Kubernetes does not auto-resubmit a finished or failed Job.

For gang-scheduled multi-pod jobs specifically: if one pod of the group is
evicted, the Indexed Job controller recreates it at the same ordinal, but
the scheduler's gang-scheduling semantics mean it (and possibly its
siblings, if resources are tight) will wait to be re-admitted together
rather than partially running — see [gang-scheduling.md](gang-scheduling.md).

## Multi-writer checkpoint races (DDP/multi-worker specifically)

For distributed data-parallel training, all ranks hold replicated model
state after an all-reduce — writing the checkpoint from every rank would
race on the same file(s). Gate your checkpoint-writing call on rank 0
only:

```python
if torch.distributed.get_rank() == 0:
    save_checkpoint(...)
```

All ranks can safely read the (read-only) dataset mount concurrently —
only the write path needs this guard.
