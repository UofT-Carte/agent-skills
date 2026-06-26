---
name: running-on-alliance-hpc
description: Use when running experiments, jobs, or code on Digital Research Alliance of Canada HPC clusters (Nibi, Fir, Narval, Rorqual, Trillium) — writing Slurm sbatch jobs, requesting GPUs/H100s/MIG slices, setting up module+virtualenv+wheelhouse environments, running vLLM or PyTorch on cluster GPUs, staging code and data to the cluster, or diagnosing failed/pending Slurm jobs.
---

# Running on Alliance Canada HPC clusters

## Overview

The Digital Research Alliance of Canada (formerly Compute Canada) clusters — Fir, Narval, Nibi, Rorqual, Trillium, and the AI-dedicated PAICE clusters — share a **Slurm** scheduler, **Lmod** module system, `StdEnv` environments, and a prebuilt **wheelhouse**. This skill is the hard-won workflow for getting compute (especially GPU/vLLM) jobs to run reliably. The workflow is **cluster-general**; the things that vary between clusters — GPU model, MIG availability, scratch-purge policy, whether compute nodes have internet — are flagged throughout. Command examples use H100/MIG specifiers (Fir/Nibi/Rorqual); A100 clusters (Narval) and L40S/H200 clusters differ in the *specifiers*, not the *method*.

**Pick the cluster first.** If which cluster to run on isn't already settled, that's its own decision — **RELATED SKILL: `choosing-an-alliance-cluster`** (GPU fit, allocation, MIG, data locality, queue). Everything below assumes you've chosen one.

**Core operating principle — you cannot drive the cluster directly.** Alliance enforces **MFA on interactive SSH**, so an assistant/agent/CI **cannot `ssh`, `rsync`, or `sbatch` non-interactively**. Your job is to:
1. Do everything that *can* be done off-cluster yourself (write code, sbatch scripts, configs, analysis, reports).
2. Emit the **exact commands** the human runs on the cluster.
3. **Never assume a cluster command ran** — wait for the human to paste back `squeue`/`sacct`/log output before reasoning about results.

(Some clusters offer a headless automation node — e.g. `robot.nibi.alliancecan.ca` — for genuinely non-interactive jobs, but it needs separate key setup; don't assume it.)

**Authoritative facts change — look them up via the `alliance-docs` MCP.** Quotas, RGU/billing values, MIG profile names, and module versions drift. The official wiki (`docs.alliancecan.ca`) is the source of truth but **blocks automated web access**, so the **`alliance-docs` MCP** is the machine-readable way to query it: `search_docs`, then `get_page_content` (key pages: your cluster's own page — e.g. `Nibi`, `Fir`, `Narval`, `Rorqual` — plus `Using_GPUs_with_Slurm`, `Standard_software_environments`, `National_systems`). Trust it over any hardcoded number in this skill.

**If the `alliance-docs` MCP is not connected, _offer_ to install it — only offer; do not install it without the user's go-ahead.** It's a public, no-auth streamable-HTTP server ("Alliance Docs"). In **Claude Code**:

```
claude mcp add --transport http alliance-docs https://alliance-docs-mcp.fly.dev/mcp/
```

(Add `-s user` to make it available in every project, not just this one.) **Other runtimes:** register an HTTP/streamable MCP server at `https://alliance-docs-mcp.fly.dev/mcp/`. If the user declines (or you're mid-task), fall back to the values in this skill and flag that they may be stale.

## When to use

- Writing or editing an sbatch script, choosing GPU/CPU/memory/time for a job.
- Setting up a Python environment (modules + virtualenv + wheelhouse) for the cluster.
- Running vLLM, PyTorch, or other GPU workloads — **read `gpu-vllm-reference.md`**, the gotchas there cost the most time.
- Staging code/data onto the cluster or pulling results back.
- A job `FAILED`/`TIMEOUT`/sits `PENDING` and you're diagnosing why — **see the troubleshooting table in `slurm-ops-reference.md`**.

## The campaign workflow

Every run, in order. Each step exists to surface a failure *before* it costs GPU time.

1. **Deliver code with git, data with rsync.** *First time:* on a login node, **`git clone` your repo** into `/project` or `/scratch` and authenticate git once (HTTPS + a GitHub token, or an SSH key generated on the cluster) — login nodes always have internet. *Every time after:* **tracked file changed → `git pull --ff-only`; gitignored file (data, `.env`) changed → `rsync`.** `git pull` cannot carry gitignored paths — confusing the two causes the classic stale-data crash (new code against old data, dies on a missing key/file). First-time git auth + resume-safe rsync of huge files are in `slurm-ops-reference.md`.

2. **Stage everything heavy on a *login node*, before any GPU allocation.** Build the venv, run model-free tests, and pre-download weights on the login node. A broken wheel or a 40-minute checkpoint download should surface for free — never 10 minutes into a 4×H100 allocation. Make this step **idempotent** (reuse the env if present; `hf download` resumes).

3. **Build the environment the Alliance way** (no conda): `module load` first, then `virtualenv`, then `pip --no-index` from the wheelhouse. Some system-ish deps (**opencv, pyarrow, mpi4py**) must come from a **module**, not pip — the wheelhouse serves a dummy wheel that always fails. Full recipe and traps in `gpu-vllm-reference.md`.

4. **Right-size the request** (three levers, highest-leverage first):
   - **(a) Cut the *work* before the wall-clock** — fewer permutations/samples often reproduces the result in half the time.
   - **(b) Estimate `--time` from a measured rate + a modest 20–30% buffer.** Do **not** over-pad: a 5 h job requested at 8 h backfills *later* than at 7 h (long limits don't fit short gaps). Resumable jobs (step 6) make guessing low cheap — err short.
   - **(c) Match CPU/mem to the GPU.** See the per-tier table in `gpu-vllm-reference.md`. Over-asking only delays scheduling.

5. **Choose the GPU: smallest slice that fits.** Where **MIG** is available (Fir, Narval, Nibi, Rorqual — *not* Trillium or the PAICE clusters), a model that fits a MIG slice should take one — slices cost fewer RGU and **backfill far sooner** than a whole GPU. Slice sizes are GPU-specific (H100: ~10/20/40 GB; A100 on Narval differs). Whole/multi-GPU only when the model needs it, and then pin **`--nodes=1`** (a bare `--gpus=N` may scatter across nodes and break tensor-parallel jobs). Per-cluster GPU specifiers, sizing table, and the `--gpus`-mixing rejection are in `gpu-vllm-reference.md`.

6. **Make the job resumable + guard its output.** Append results incrementally and skip done units, so a `TIMEOUT` loses nothing (just resubmit the same command). Write a header hashing the config + input set and **refuse to append on mismatch** — this catches stale output files in 10 s instead of producing garbage. Detail in `slurm-ops-reference.md`.

7. **Smoke-test, then launch.** A 5-item, 30-minute job catches ~90% of failures (bad path, stale data, env var, OOM, MIG/spawn issues) for almost nothing and backfills immediately. Only after it's green do you submit the multi-hour run.

8. **Monitor and, on failure, diagnose by signal.** `sacct` for the *state*, the `slurm-<jobid>.out` log for the *traceback*. Died in ~1 min → startup error (read the log, don't guess). Ran for hours then `TIMEOUT` → under-budgeted, just resume. Commands and failure-signature heuristics in `slurm-ops-reference.md`.

## Quick reference

```bash
# storage / quota / recover deleted home|project file (30-min snapshots)
diskusage_report ;  oops [dir]
# modules: find one + its prereqs, then load StdEnv + toolchain BEFORE the venv
module spider <name>
module load StdEnv/2023 gcc python/3.12 opencv cuda
avail_wheels 'vllm*'                       # what the wheelhouse has
# submit / override resources on the CLI (CLI flags beat #SBATCH defaults)
sbatch --account=def-<pi> --gpus=h100_2g.20gb:1 --cpus-per-task=4 --mem=62G --time=0-06:00 job.sbatch  # H100 MIG slice; on Narval use a100/a100_*g.* specifiers
sbatch --nodes=1 --gpus=h100:4 ...         # multi-GPU: pin to one node
sbatch --array=0-15%8 ...                  # 16 shards, ≤8 concurrent
# monitor / forensics
squeue -u $USER --format="%.10i %.30j %.8T %.10M %R"
sacct -u $USER -X --format=JobID,JobName%18,State,Elapsed,ReqTRES%45,ExitCode,Start,End -S today
seff <jobid> ;  tail -f slurm-<jobid>.out ;  scancel <jobid>   # scancel -u $USER kills ALL
```

Storage tiers (look up current numbers via the `alliance-docs` MCP): `$HOME` small+backed-up (code); `$SCRATCH` large, **not** backed up (data, caches, outputs — Nibi: no age-purge under 1 TB soft / 20 TB hard; **other clusters age-purge at ~60 days**); `/project/def-<pi>` group quota, backed up (durable shared data); `$SLURM_TMPDIR` node-local, per-job. Keep many-small-file venvs/caches **off** `$HOME`/`$PROJECT` inode quotas — redirect onto `$SCRATCH`.

## Common mistakes

| Mistake | Reality |
|---|---|
| Trying to `ssh`/`sbatch` for the user, or assuming a job ran | MFA blocks non-interactive SSH. Emit commands; wait for pasted-back output. |
| Assuming all compute nodes do (or don't) have internet | It's **cluster-specific**: Nibi compute nodes have internet; most others are air-gapped. Prefetch on a login node either way (portable, faster, reproducible). |
| Hardcoding one cluster's GPU specifier everywhere | Specifiers differ per cluster (H100 vs `a100` vs `l40s`…). Use `choosing-an-alliance-cluster` + the per-cluster table; confirm with `avail_wheels`/the MCP. |
| `--gres=gpu:h100:1` | Documented form is `--gpus=h100:1`. `--gres=...` still works but isn't the current syntax. |
| Whole H100 for a small model | Use the smallest **MIG slice** that fits — cheaper and schedules sooner. |
| `pip install vllm` (no `--no-index`); pip-installing opencv/pyarrow | Use the wheelhouse (`--no-index`); load opencv/pyarrow as **modules** or pip's dummy wheel breaks the build. |
| Over-padding `--time` for safety | Shorter limits backfill sooner. Pad ~20–30%; lean on resumability. |
| Hardcoding quotas / RGU / GPU memory from memory | Look it up via the `alliance-docs` MCP — these change. Nibi H100 = 80 GB, not 94. |

## Related & reference

- **`choosing-an-alliance-cluster`** (sibling skill) — pick the cluster before you start: GPU model fit, MIG availability, allocation, data locality, queue.
- **`gpu-vllm-reference.md`** — GPU/MIG sizing table, multi-GPU placement, the `--gpus` mixing gotcha, and the full vLLM-on-Alliance gotcha set (spawn, flashinfer/deep_gemm, no-`nvcc` on compute nodes, the MIG-UUID patch, cache redirection). **Read before any GPU/vLLM job.**
- **`slurm-ops-reference.md`** — sbatch templates, resume-safe rsync, resumability + hashed-header hygiene, monitoring commands, failure-signature heuristics, and a real-incident troubleshooting table.
