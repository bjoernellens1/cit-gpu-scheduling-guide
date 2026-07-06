# Multi-node NCCL tuning (non-RDMA Ethernet fabrics)

If your distributed training job spans multiple physical nodes and your
cluster doesn't have InfiniBand/RDMA, the defaults leave real performance
on the table — in one real, measured case, over 60x.

## The default-path problem: CNI overlay overhead

By default, pod traffic on many Kubernetes CNIs (e.g. Flannel with a VXLAN
backend) transits an encapsulated overlay network rather than the physical
NIC directly. On a real cluster with a 10GbE fabric, this measured only
**~0.39 GB/s** inter-node point-to-point bandwidth — about 30% of the
~1.1-1.2 GB/s a single 10GbE NIC should sustain at line rate.

**Fix: `hostNetwork: true`** on training pods, bypassing the CNI overlay
entirely:

```yaml
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet   # required so cluster DNS still resolves with hostNetwork
```

This is the standard fix used by NCCL-on-Kubernetes reference
architectures (e.g. Kubeflow Training Operator deployments, cloud GPU
node-pool guides) for exactly this class of problem.

**Trade-off**: `hostNetwork` gives up per-pod network isolation — the pod
uses the node's network namespace and IP directly, and its ports become
node-level ports (watch for port conflicts if running multiple such jobs
on the same node). This is a reasonable trade for a trusted internal batch
queue; reconsider for less-trusted/multi-tenant workloads where network
isolation matters more.

Pin NCCL to the real physical interface once you're on `hostNetwork`
(auto-detection can otherwise still pick a virtual/loopback interface):

```yaml
env:
  - name: NCCL_SOCKET_IFNAME
    value: "eth0"   # confirm your node's real interface name first
```

## Socket transport parallelism

NCCL's default socket transport on a non-RDMA fabric uses too few
threads/sockets to saturate a typical 10GbE (or faster) NIC. Increase
parallelism:

```yaml
env:
  - name: NCCL_SOCKET_NTHREADS
    value: "4"
  - name: NCCL_NSOCKS_PERTHREAD
    value: "4"
```

These specific values (4x4) are a reasonable **starting point**, not
verified-optimal for every fabric — re-benchmark on your own hardware (see
the discriminator test below) before trusting a specific value in
production. The product of `NCCL_SOCKET_NTHREADS × NCCL_NSOCKS_PERTHREAD`
is capped at 64 by NCCL itself — don't push both values too high.

## `OMP_NUM_THREADS` — a silent, easy-to-miss default

`torchrun` defaults `OMP_NUM_THREADS` to `1` per process when it can't see
your real CPU request, and prints a warning
(`Setting OMP_NUM_THREADS environment variable for each process to be 1`)
that's easy to miss in noisy logs. With, say, 8 CPUs requested for 2 GPU
processes per pod, a default of 1 thread leaves 7 of your requested cores
idle per process for data-loading/preprocessing — a real, silent
performance cliff.

```yaml
env:
  - name: OMP_NUM_THREADS
    value: "4"   # set explicitly to requests.cpu / nproc_per_node
```

## Benchmark before and after — don't assume, measure

Before tuning anything, establish a real baseline with a point-to-point
NCCL send/recv test between two nodes, and re-run it after each change to
confirm actual improvement on your specific hardware:

```python
import os, time, torch, torch.distributed as dist

rank = int(os.environ["RANK"])
dist.init_process_group(backend="nccl", rank=rank, world_size=2)
torch.cuda.set_device(0)
dev = torch.device("cuda:0")

for mb in [16, 256]:
    n = (mb * 1024 * 1024) // 4
    t = torch.randn(n, device=dev)
    iters = 10
    if rank == 0:
        dist.barrier()
        start = time.time()
        for _ in range(iters):
            dist.send(t, dst=1)
        torch.cuda.synchronize()
        elapsed = (time.time() - start) / iters
        print(f"{mb} MB: {elapsed*1000:.2f} ms  {(mb/1024)/elapsed:.2f} GB/s")
    elif rank == 1:
        dist.barrier()
        for _ in range(iters):
            dist.recv(t, src=0)
        torch.cuda.synchronize()
```

Run this with and without `hostNetwork`/socket tuning and compare. On the
real cluster this guide is based on, applying `hostNetwork` +
`NCCL_SOCKET_IFNAME` alone (before further socket-thread tuning) was the
single highest-impact change — confirm the same holds for your fabric
before assuming socket-thread tuning matters more.

## If you're on a hypervisor/virtualized network fabric

If your GPU nodes are themselves VMs (e.g. on Proxmox, VMware, or a cloud
hypervisor), be aware that the "10GbE" reported inside the guest may be a
**virtio-net (or equivalent) virtual NIC**, not a real dedicated physical
link. Virtio-net defaults to a **single queue**, meaning all of a VM's
network traffic — regardless of how many parallel NCCL sockets/threads you
configure inside the guest — gets serialized through a single backend
worker thread on the hypervisor host. If you've applied all the tuning
above and still see a hard bandwidth ceiling well under your fabric's
nominal speed, check with whoever manages the hypervisor whether
multi-queue virtio-net is enabled (both on the host VM config **and**
activated inside the guest, e.g. via `ethtool -L <iface> combined N`) —
this is a real, documented bottleneck class for VM-based GPU clusters, not
purely an NCCL-level problem.
