# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A documentation/example-manifest guide, not a deployable application. It teaches how to schedule
PyTorch/TensorFlow GPU jobs on a Kubernetes cluster running NVIDIA KAI Scheduler: single-GPU
exclusive, MPS-shared/fractional, multi-GPU gang-scheduled, multi-node NCCL-tuned, plus priority
queues and storage. There is no build/test/run step in the traditional sense — "verification" here
means the YAML examples are internally consistent with what the docs describe (correct
`schedulerName`, `kai.scheduler/queue` labels, placeholder conventions, etc.), not a CI pipeline.

## Structure

- `docs/` — concept explainers (`gpu-sharing-modes.md`, `gang-scheduling.md`,
  `multi-node-networking.md`, `priority-queues.md`, `storage.md`, `ablator.md`, `cluster-access.md`).
- `examples/` — template manifests organized by `single-gpu/`, `multi-gpu/`, `multi-node/`.
- Root `README.md` is the entry point — Quick Start table + table of contents; keep both in sync
  when adding a new doc or example.

## Load-bearing assumptions (see README's "Assumptions" section)

Every example and doc in this repo assumes, and re-derives nothing contrary to:

- KAI Scheduler v0.12.10+ (`schedulerName: kai-scheduler`, `Queue`/`PodGroup` CRDs).
- NVIDIA GPU Operator device plugin in **plain (non-MIG) mode** — `nvidia.com/gpu` is the real
  physical GPU count, not MIG-partitioned or replicated. MIG-based sharing is explicitly out of
  scope.
- NVIDIA MPS available on GPU nodes for fractional sharing via KAI's `gpu-memory` annotation.
- A CNI without a fast path for multi-node NCCL traffic by default (e.g. Flannel VXLAN) — this
  motivates the `hostNetwork` advice in the multi-node docs; skip that advice on RDMA/InfiniBand
  clusters.

If asked to edit an example or doc, don't silently drop or contradict these assumptions — if a
change requires a different assumption (e.g. MIG mode, a different scheduler), call it out
explicitly rather than blending it in as if it were universal.

## Examples are templates, not working configs

Every manifest under `examples/` intentionally uses placeholders (`your-registry/your-image`,
`<your-namespace>`, storage class names, queue/priority-class names) marked with comments. When
asked to "fix" or "customize" an example, preserve the placeholder-and-comment pattern — don't
replace a placeholder with a hardcoded value that merely looks like a real one (a real-looking
namespace or image that isn't actually valid is worse than an obvious placeholder).

## Where `ablator` fits

[`ablator`](https://github.com/bjoernellens1/ablator) is a separate sibling tool (not in this repo)
that generates and submits the kind of manifests this guide teaches, from a TOML config + JSON
spec — useful for hyperparameter sweeps/ablations or for users who don't want to hand-write YAML.
See `docs/ablator.md` for the full relationship and a worked comparison; don't re-derive it from
scratch if asked something like "add an ablator example" or "how does ablator relate to this
guide."
