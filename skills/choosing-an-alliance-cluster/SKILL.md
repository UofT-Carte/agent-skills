---
name: choosing-an-alliance-cluster
description: Use when deciding which Digital Research Alliance of Canada cluster to run a job on or request an allocation for — comparing Fir, Narval, Nibi, Rorqual, Trillium, or the AI-dedicated PAICE clusters (TamIA, Killarney, Vulcan) by GPU model (H100/H200/A100/L40S/MI300A), GPU memory, MIG availability, FP8 support, interconnect, compute-node internet, allocation/account, data locality, or queue wait.
---

# Choosing an Alliance Canada cluster

## Overview

The Alliance runs several production clusters that share a software stack but differ in **GPU hardware, interconnect, scheduling model, and local policy**. Picking the right one before you submit (or before you write a RAC request) saves queue time and avoids cross-cluster data moves. This skill is the *decision*; **`running-on-alliance-hpc` is the execution** once you've chosen.

**The fleet changes** — clusters are commissioned and retired and specs drift. Confirm current specs and your access via the **`alliance-docs` MCP** (pages `National_systems`, `Using_GPUs_with_Slurm`, the per-cluster page, and `Using_a_resource_allocation`) before relying on the numbers here. (If that MCP isn't connected, see `running-on-alliance-hpc/SKILL.md` for the offer-to-install step.) Don't target an **end-of-life** system — Béluga, Cedar, Graham, and Niagara are retired (Cedar→Fir, Béluga→Rorqual, Niagara→Trillium). Full per-cluster detail is in **`cluster-spec-sheet.md`**.

## When to use / not

- **Use** when you can run on more than one cluster, or you're writing a RAC/allocation request and choosing a target system, or a job won't fit/won't schedule where you first tried.
- **Skip** if you hold one allocation on one cluster and the job fits — just use it (read `running-on-alliance-hpc`).

## Step 0 — Can you even run there? (access + allocation)

Two separate gates; both must be true:

1. **Per-cluster access.** Even with an allocation you must request access to each cluster in **CCDB → Resources → Access Systems → select cluster → "I request access"** (takes up to ~1 h). No exceptions — a `def-` allocation does *not* auto-grant access to every cluster.
2. **An allocation (account) valid there.** You pass it as `--account=`:
   - **`def-<pi>`** — the *default* RAP, auto-created per PI, **lowest (opportunistic) priority**. All sponsored users are members. Works on the general-purpose clusters. Use for general/unallocated research.
   - **`rrg-<pi>` / `rpp-<pi>`** — a **RAC award** (Resource Allocation Competition; RRG = research groups, RPP = platforms). **Higher priority**, but **granted on specific resources** (e.g. `nibi-gpu`, `fir-cpu`) and only on the clusters you were awarded. RAC year starts ~first week of April.
   - **`aip-<pi>`** — a **PAICE** AIP-type RAP, required for the AI-dedicated clusters (TamIA, Killarney, Vulcan). Restricted to AI researchers; those clusters are **geo-blocked to Canada**.

   Find what you hold (RAPIs, group names, allocations) on the CCDB portal. If you have multiple groups, you **must** set `--account` explicitly.

If only one cluster passes both gates, you're done — go run.

## Step 1 — Match the GPU to the job

Pick the capability the job *needs*, then the clusters that have it (memory and FP8 are the usual deciders):

| Need | GPU | Clusters |
|---|---|---|
| FP8 + biggest memory (large LLM training/inference) | **H100-80GB** | Fir, Nibi, Rorqual, Trillium; PAICE: Killarney (perf), TamIA |
| Largest single-GPU memory (huge models) | **H200 ~141GB** / **MI300A ~128GB (unified, ROCm-only)** | TamIA (H200); Nibi (MI300A) |
| ≤40 GB, FP16/BF16, no FP8 needed | **A100-40GB** | Narval (the only big A100 fleet) |
| Inference / smaller models, cost-efficient | **L40S-48GB** | PAICE: Vulcan, Killarney (standard) |

Notes that bite:
- **A100 (Narval) has no FP8** and a hard 40 GB ceiling — don't send an FP8 or >40 GB job there.
- **MI300A (Nibi) is AMD/ROCm** — CUDA-only code will not run; whole-node scheduling; software stack is nascent.
- **L40S** has no NVLink — great for single-GPU inference, poor for multi-GPU tensor-parallel.

## Step 2 — Match the job *shape* to scheduling + interconnect

- **Small job that fits a MIG slice (≤10/20/40 GB)?** Prefer a **MIG-capable general-purpose cluster — Fir, Narval, Nibi, Rorqual.** Slices cost fewer RGU and backfill far sooner. Trillium and the PAICE clusters allocate **whole GPUs only** (TamIA *forces* all 4 GPUs per node).
- **Tightly-coupled multi-node (large MPI / multi-node training)?** Interconnect quality matters most on **Trillium** (NDR InfiniBand, fully non-blocking, 800 Gb/s GPU) — it's the *large-parallel* system built for this. Narval/Rorqual/Fir are InfiniBand with blocking factors; **Nibi is Ethernet** (200/400 G) — add `#SBATCH --switches=1` to keep a job on one switch.
- **Job longer than 24 h?** Most clusters allow 7 days; **TamIA caps jobs at 24 h.**
- **Many small jobs?** QC clusters (Narval, Rorqual, TamIA) cap you at **1000 queued+running** jobs.

## Step 3 — Compute-node internet (affects prefetch)

It's **per-cluster**, not uniform:

| Internet on compute nodes | Clusters |
|---|---|
| **Yes** (can download in-job) | Fir, Nibi |
| **No** (air-gapped — prefetch on a login node) | Narval, Rorqual, TamIA |
| **Proxy/whitelist only** | Vulcan (Squid proxy) |

Code for the **air-gapped** case regardless (prefetch weights/packages on login) — it's portable across all of them and faster/reproducible even where internet is available.

## Step 4 — Data locality, then queue, break ties

- **Run where your data already lives.** Moving TBs between clusters (Globus) costs hours + duplicate storage and quota. Locality often outweighs a marginally better GPU — *unless* the GPU is a hard requirement (e.g. FP8 forces you off Narval). If you must move, budget the Globus transfer explicitly and keep a copy off scratch.
- **Mind storage policy** (see spec sheet): **Nibi scratch** doesn't age-purge under its 1 TB soft limit; **every other cluster age-purges** scratch. Never keep the only copy of anything on scratch anywhere.
- **Break remaining ties on queue depth** — among clusters that clear Steps 0–3, pick the shorter wait. PAICE clusters (if you're eligible) often have shorter GPU queues than the general-purpose fleet.

## Quick comparison

Verify via the MCP — specs and status drift. Detail + endpoints in `cluster-spec-sheet.md`.
**FP8 note:** every cluster here supports FP8 **except Narval** (A100 is FP16/BF16 only), so FP8 is not a column below — it's "anything but Narval."

| Cluster | Class | GPU(s) | GPU mem | MIG | Compute-node internet | Max job |
|---|---|---|---|---|---|---|
| **Fir** | general | H100 ×4/node | 80 GB | yes | yes | 7 d |
| **Narval** | general | A100 ×4/node | 40 GB | yes (4 sizes) | no (air-gapped) | 7 d |
| **Nibi** | general | H100 ×8/node (+MI300A) | 80 GB (128 GB) | yes | yes | — |
| **Rorqual** | general | H100 ×4/node | 80 GB | yes | no (air-gapped) | 7 d |
| **Trillium** | large-parallel | H100 ×4/node | 80 GB | no | restricted | — |
| **TamIA** | PAICE (Mila) | H100 ×4 / H200 ×8 per node | 80 / ~141 GB | no | no (air-gapped) | 24 h |
| **Killarney** | PAICE (Vector) | L40S ×4 / H100 ×8 per node | 48 / 80 GB | no | — | — |
| **Vulcan** | PAICE (Amii) | L40S ×4/node | 48 GB | no | proxy/whitelist | 7 d |

## Worked examples

- **8B model, MIG-friendly inference, you hold `def-` on Nibi+Narval+Rorqual:** any MIG cluster works; pick the smallest slice (`h100_2g.20gb` / `a100_3g.20gb`) on the shortest queue. Locality wins the tie.
- **70B model in FP8:** H100 required → Fir/Nibi/Rorqual/Trillium — **not Narval** (A100, no FP8), even if the data is staged there. Move the data; budget the Globus transfer.
- **Multi-node tightly-coupled training, 32+ GPUs:** **Trillium** (non-blocking NDR, whole-node culture) over the general-purpose clusters.
- **AI researcher, lots of cheap single-GPU inference, has an `aip-` RAP:** **Vulcan** or **Killarney** (L40S) — PAICE queues, inference-class GPUs.

## After choosing

Hand off to **`running-on-alliance-hpc`** for environment setup, sbatch, GPU/MIG sizing, staging, and job ops on the cluster you picked.
