# Slurm jobs, data movement, resumability, and forensics

Companion to `SKILL.md`. GPU/vLLM specifics live in `gpu-vllm-reference.md`.

## Access points

Hostnames follow `<cluster>.alliancecan.ca`; the Nibi values below are the example — swap in `fir`, `narval`, `rorqual`, etc. (confirm a cluster's exact endpoints via the `alliance-docs` MCP page for that cluster).

- SSH / rsync / scp / sftp: `nibi.alliancecan.ca` (login nodes double as transfer nodes).
- Non-interactive automation, where offered (separate key setup): `robot.nibi.alliancecan.ca`.
- Globus collection (very large or inter-cluster): `alliancecan#nibi`.
- Web shell / desktop / JupyterLab: Open OnDemand (Nibi: `ondemand.sharcnet.ca`; other clusters have their own portal).

Login nodes: editing, git, short data prep, job submission — **no GPU, no heavy/long processes** (they get killed). All real compute goes through Slurm to compute nodes.

## Moving code and data

**Code = git; gitignored data/secrets = rsync.** The cluster holds an authed git clone. Deliver tracked changes with `git pull --ff-only` (fails loudly instead of merging over cluster-local edits); rsync only an allowlist of gitignored files git can't carry.

```bash
# CODE / CONFIG (tracked)
git commit -am "..."; git push          # local
ssh <user>@nibi.alliancecan.ca           # (MFA — a human does this)
cd ~/scratch/<proj> && git pull

# DATA + RESULTS (gitignored): rsync only — git will NOT carry these
rsync -av --exclude '.venv' --exclude env --exclude __pycache__ \
  --exclude .pytest_cache --exclude results \
  ./ <user>@nibi.alliancecan.ca:~/scratch/<proj>/        # push data up
rsync -av <user>@nibi.alliancecan.ca:~/scratch/<proj>/results/ results/   # pull results down
```

**The stale-data trap:** `git pull` updates code but cannot touch gitignored `data/`, so the cluster runs new configs against an **old dataset** and dies (e.g. `KeyError` on a filename that only exists in the new data). When data changes you must rsync it.

**Resume-safe rsync of one huge file** (e.g. a 133 GB parquet); run under `tmux`/`screen`:
```bash
rsync -ah --partial --append-verify --mkpath --no-inc-recursive --info=progress2 \
  /local/path/big.parquet  <user>@nibi.alliancecan.ca:/scratch/<user>/<proj>/data/big.parquet
```
`--partial`+`--append-verify` resume then checksum the whole file; `--mkpath` makes missing dirs (rsync ≥ 3.2.3); `--no-inc-recursive --info=progress2` give one aggregate progress line. **Always verify after transfer** — size *and* semantics (e.g. `pq.read_metadata(f).num_rows`).

**Recover a deleted `$HOME`/`$PROJECT` file** from 30-min snapshots: `oops [dir]`, then `cp` the returned (read-only) path. Copying scratch→project with symlinks leaves links pointing at scratch — use `tar -cf - ./* | tar -C /project/.../dest -xf -`.

**exFAT local quirk:** a local working copy on exFAT commits new `.sh` files non-executable. After `git add`: `git update-index --chmod=+x path/to/script.sh`. (`.sbatch` run via `sbatch` don't need the bit; scripts run as `./x.sh` do.)

## sbatch template

Defaults sized for the smallest tier; override per run on the CLI (CLI flags beat `#SBATCH`).

```bash
#!/bin/bash
#SBATCH --account=def-<pi>
#SBATCH --gpus=h100_2g.20gb:1      # a 20 GB MIG slice by default
#SBATCH --cpus-per-task=4          # match the slice (see gpu-vllm-reference.md table)
#SBATCH --mem=62G
#SBATCH --time=0-06:00             # right-size: shorter backfills sooner
#SBATCH --output=slurm-%j.out      # (or %x_%j.out / .err to split by job name)
set -euo pipefail

module load StdEnv/2023 gcc python/3.12 opencv cuda   # SAME modules as setup
source env/bin/activate
export HF_HOME="${HF_HOME:-$SCRATCH/hf}"
# ... vLLM + cache env vars from gpu-vllm-reference.md ...
srun python scripts/run_eval.py --config "$1"          # srun so Slurm tracks the step
```

```bash
# smoke test: few items, short time -> backfills fast
sbatch --time=0-00:30 scripts/eval.sbatch configs/qwen3vl-4b.yaml 5
# a bigger tier
sbatch --gpus=h100_3g.40gb:1 --cpus-per-task=6 --mem=124G --time=0-12:00 scripts/eval.sbatch configs/qwen3vl-8b.yaml
# pass params without editing the template
sbatch --export=ALL,CONFIG_FILE=$CFG,VARIANT=fp8 scripts/eval.sbatch
```

## Right-sizing `--time`

```
wall-clock ≈ (n_items × work_per_item) / throughput + model-load overhead
e.g. 2154 images × 2 perms × 4.4 s/call ≈ 5.3 h  →  request --time=0-07:00
```
Pad ~20–30% for per-call variance, **not** 50% — a 5.3 h job at 8 h backfills later than at 7 h. Because resumable jobs just continue on resubmit, under-budgeting costs one resubmit, not lost work; err short. Calibrate throughput on a few-thousand-item subsample, not a tiny one (ramp-up/drain skews small samples). If GPU utilization isn't pinned high during calibration, the GPU is CPU/IO-starved (e.g. JPEG decode) — add `--cpus-per-task`, don't add GPUs.

## Resumability + output hygiene

Append one row per unit and skip already-done units, so `TIMEOUT`/preemption loses nothing — resubmit the *same* command to continue:

```python
done = read_completed(results_path)                   # {(item, perm), ...}
todo = [p for p in range(cfg.n_perm) if (item, p) not in done]
for perm in todo:
    append_row(results_path, score(...))              # one JSON line per unit
```

Guard the output file against silent corruption — write a header hashing config + input set, and **refuse to append on mismatch**, before paying model-load cost:
```python
def ensure_header(path, cfg, input_hash):
    if path.exists():
        h = read_header(path)
        if h["config_hash"] != config_hash(cfg) or h["input_hash"] != input_hash:
            raise ValueError(f"{path}: produced by different config/inputs; delete or rename first")
```
Design results filenames around the *experiment* (distinct config name → distinct output file) so A/B variants never collide; make experiment knobs part of the config so the hash distinguishes them. Hash only **non-default** fields so adding a new optional knob doesn't invalidate existing files. **Clear stale checkpoints/parts before a fresh run** — leftover parts get merged into the new output. Because `rsync` excludes `results/`, the cluster's `results/` is *not* refreshed by a code sync; a stale same-named file there will either trip the guard (good) or, if you `rsync … results/ results/`, get pulled down and overwrite local expectations (sanity-check what you sync).

## Parameter sweeps

For N independent configs, submit N resumable jobs (each right-sized for its tier) via a thin submitter; re-running it after a failure just resubmits and skips completed units. For homogeneous sweeps, Slurm **job arrays** are idiomatic; throttle concurrency with `%K`:
```bash
sbatch --array=0-15%8 ...   # 16 shards, ≤8 running at once; inside: SHARD=$SLURM_ARRAY_TASK_ID
```
One-job-per-config is easier when tiers need *different* resources; arrays are cleaner when they don't.

## Monitoring and forensics

```bash
squeue -u $USER --format="%.10i %.30j %.8T %.10M %R"   # id, name, state, time, reason
squeue -u $USER --start                                # estimated start for PENDING jobs
sacct -u $USER -X --format=JobID,JobName%18,State,Elapsed,ReqTRES%45,ExitCode,Start,End -S today
seff <jobid>                                           # CPU/mem efficiency after completion
tail -f slurm-<jobid>.out                              # live log; tracebacks land here
scancel <jobid>                                        # scancel -u $USER kills ALL — careful
```
- `-X` = one row per job (hides `.batch`/`.extern` steps). `ReqTRES` distinguishes a MIG slice (`gres/gpu:nvidia_h100_80gb…`) from whole GPUs (`gres/gpu:h100=N`).
- **States:** `COMPLETED` (0:0), `FAILED` (non-zero), `TIMEOUT`, `OUT_OF_MEMORY`, `CANCELLED`, `PENDING`/`RUNNING`.

**Failure-signature heuristics:**
- Dies in ~10–60 s, `FAILED 1:0` → startup error (import, config parse, missing input, refused stale output, bad `HF_HOME`). **Read the log for the traceback — don't guess.**
- `Start=None`, `Elapsed=00:00:00`, `CANCELLED` → never ran (killed while `PENDING`). Several jobs sharing the *exact* cancel timestamp = a bulk `scancel`/maintenance flush, not a per-job fault.
- Ran for hours then `TIMEOUT` → under-budgeted; raise `--time` and resume.
- `OUT_OF_MEMORY` → smaller batch/model or bigger slice.
- Completed but output looks wrong → reconcile job `End` time with the output-file mtime; a stale file is the usual culprit.

## Troubleshooting table (real incidents)

| Symptom | Root cause | Fix |
|---|---|---|
| `FAILED` in ~1 min, log "Could not find nvcc" at vLLM init | MoE FlashInfer CUTLASS kernel JIT needs `nvcc`; compute nodes have only the driver | Force TRITON MoE: `VLLM_USE_FLASHINFER_MOE_FP16/FP8=0` |
| `FAILED` in ~54 s at model load | `HF_HOME` pointed at an empty path → cache miss | Point `HF_HOME` at the real weights cache + `HF_HUB_OFFLINE=1` |
| `FAILED` fast: "input not found: …parquet" | large input absent from scratch (missing migration / manual removal) | Re-upload from the off-cluster source of truth; add a prereq `ls`/row-count before submitting |
| New run mixes in stale rows | leftover parts/checkpoint from a prior run merged in | Clear parts + checkpoint + old output before a fresh run |
| Concurrent array tasks corrupt the venv | all tasks installed into the same venv path at once | Build the venv once on login; tasks only `source` it |
| `.sh` won't execute after sync | exFAT local FS dropped the exec bit at commit | `git update-index --chmod=+x <script>` |
| Jobs stuck `PD` for ages | over-requested `--time` and/or many full-H100s competing | right-size `--time` for backfill; throttle the array with `%K` |
