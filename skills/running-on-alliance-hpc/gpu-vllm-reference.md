# GPU selection, MIG, and vLLM on Alliance HPC

Read this before any GPU or vLLM job. These are the things that cost real debugging time and GPU allocations. **Look up the numeric values (RGU, memory, core counts, MIG profile names) via the `alliance-docs` MCP** — pages `Using_GPUs_with_Slurm` and `Nibi` — because they change per cluster and over time. (The official wiki blocks automated web access; the MCP is the machine-readable path. If it's not connected, see `SKILL.md` for the offer-to-install step.) The values below were correct for Nibi's H100-80GB nodes as of this writing — treat them as possibly stale.

## GPU models and specifiers per cluster

**Always pass an explicit model specifier** — a bare `--gpus=1` may be rejected or land on an arbitrary GPU. Specifiers differ by cluster (confirm via the MCP, page `Using_GPUs_with_Slurm`, or on the cluster with `avail` for GRES):

| Cluster | GPU model | Whole-GPU specifier | MIG slice specifiers | ~max cores/GPU |
|---|---|---|---|---|
| Fir | H100-80GB | `h100` | `h100_1g.10gb` / `_2g.20gb` / `_3g.40gb` | 12 |
| Narval | A100-40GB | `a100` | `a100_1g.5gb` / `_2g.10gb` / `_3g.20gb` / `_4g.20gb` | 12 |
| Nibi | H100-80GB (+ MI300A) | `h100` (`mi300a`) | `h100_1g.10gb` / `_2g.20gb` / `_3g.40gb` | 14 |
| Rorqual | H100-80GB | `h100` | `h100_1g.10gb` / `_2g.20gb` / `_3g.40gb` | 16 |
| Trillium | H100-80GB | `h100` | — (whole GPU only) | whole-node |
| TamIA | H100-80GB, H200 | `h100`, `h200` | — | — |
| Killarney | H100-80GB, L40S | `h100`, `l40s` | — | — |
| Vulcan | L40S-48GB | `l40s` | — | — |

(The long form `nvidia_h100_80gb_hbm3_2g.20gb` is equivalent to the `h100_2g.20gb` synonym on H100 clusters.) Match CPU/mem to the GPU using the **bundle characteristics** table (via the MCP) — over-asking delays scheduling. Use `choosing-an-alliance-cluster` to pick which of these to target.

## Choosing a slice vs whole GPU — worked example (Nibi H100)

Nibi GPU nodes are **8× H100 SXM (80 GB), NVLink**, 112 cores, ~2000 GB RAM → a per-GPU "bundle" of ≈ **14 cores / 250 GB**. About half the GPU nodes are MIG-partitioned. The *method* below is identical on Fir/Rorqual (H100) and Narval (A100); only the specifiers and RGU values change.

**Rule: request the smallest MIG slice the model fits in.** Slices cost proportionally fewer RGU *and* schedule far sooner because they backfill into scheduler gaps. A 30-min `2g.20gb` job can start almost immediately; a `--gpus=h100:4` job waits for four free GPUs on one node.

| GPU request (Nibi H100) | RGU/billing | recommended cores / mem | use for |
|---|---|---|---|
| `--gpus=h100_1g.10gb:1` | ~1.74 | 2 cores, ~31 GB | ≤ ~10 GB weights |
| `--gpus=h100_2g.20gb:1` | ~3.48 | 4 cores, ~62 GB | a 4B/8B model at modest resolution |
| `--gpus=h100_3g.40gb:1` | ~6.1 | 6–7 cores, ~124 GB | a 40-GB-class / 8B dense model |
| `--gpus=h100:1` (whole) | ~12.2 | 14 cores, 250 GB | doesn't fit a slice; ~60 GB MoE |
| `--gpus=h100:N` `--nodes=1` | N × 12.2 | N × bundle | tensor-parallel / model > one GPU |

RGU drives fair-share priority; 8 full H100s ≈ 98 RGU, so expect some tasks to sit `PD` rather than all starting at once. (On Narval, an A100 whole GPU and its `a100_*g.*` slices carry different RGU — look them up.)

## Multi-GPU placement — two gotchas

**Pin multi-GPU jobs to one node.** `--gpus=N` is a *total TRES count* — Slurm may satisfy it across multiple nodes. A single-process tensor-parallel job (vLLM, DeepSpeed, `torchrun` on one node) then sees only its launch node's GPUs and dies:

```
ValueError: World size (4) is larger than the number of available GPUs (1) in this node.
```

Fix — co-locate:
```bash
sbatch --nodes=1 --gpus=h100:4 --cpus-per-task=32 --mem=250G --time=0-08:00 job.sbatch
# (--gpus-per-node=h100:4 also co-locates; if you use it, don't ALSO carry a --gpus= default)
```

**Don't mix GPU-request forms.** Pick **one** form per job. If the sbatch header uses `--gpus=…` and you add `--gpus-per-node=…` on the CLI, the cluster **rejects the job**. Standardize on `--gpus=` everywhere (header *and* overrides) so they compose.

## Software environment recipe

No conda. Load modules → virtualenv → `pip --no-index` from the wheelhouse.

```bash
# load StdEnv + the exact toolchain modules you need, BEFORE the venv.
# confirm versions first: module spider opencv
module load StdEnv/2023 gcc python/3.12 opencv cuda

virtualenv --no-download env        # --no-download: use the module's pip/setuptools
source env/bin/activate
pip install --no-index --upgrade pip

# wheelhouse first (CUDA-matched, no compiling), PyPI fallback only if needed
pip install --no-index vllm torchvision accelerate pillow pyyaml pytest \
  || pip install vllm torchvision accelerate pillow pyyaml pytest
# discover wheelhouse contents:  avail_wheels 'vllm*'
```

**Some deps must come from a MODULE, not pip.** vLLM depends on `opencv-python-headless`; the wheelhouse serves a *dummy* wheel for it (and for `pyarrow`) that always fails. Loading the **`opencv` module** (and `arrow`) before pip makes pip see the requirement as already satisfied and skip it. Create the venv with access to module packages (`virtualenv --no-download`, or `--system-site-packages`). General lesson: when a wheelhouse install fails on a system-ish library, find a module that provides it and load it first.

**Editable installs need build deps pre-installed** (with `--no-build-isolation`, so pip doesn't refetch torch):
```bash
pip install --no-index hatchling editables || pip install hatchling editables
pip install -e . --no-deps --no-build-isolation
```

**Build the venv ONCE on the login node.** Concurrent array tasks installing into the *same* venv path race and corrupt it. Build (+ patch + prefetch) once on login; GPU jobs only `source` it. For thousands of short jobs, build into `$SLURM_TMPDIR` per-job (node-local disk imports far faster than thousands of tiny files on the parallel FS) or pack a squashfs/`venv --copies` tarball.

**Prefetch model weights on the login node**, then run offline:
```bash
export HF_HOME="${SCRATCH:?run on a login node}/hf"   # on SCRATCH (tens of GB); NOT $HOME
hf download "$MODEL_ID"                                # snapshots + resumes partials
# in the job: HF_HUB_OFFLINE=1, TRANSFORMERS_OFFLINE=1, and HF_HOME pointing at the SAME path
```
A 54-second model-load failure is almost always `HF_HOME` pointing at the wrong/empty path — keep **one canonical `HF_HOME`** across prefetch and every job. Even though Nibi compute nodes have internet, prefetch + offline is the portable, reproducible, no-paid-GPU-download choice (and is *mandatory* on air-gapped clusters).

## vLLM on Alliance — set these in the job script

These four env vars pre-empt the most common crashes:

```bash
# MIG + forking: torch's NVML fast path can't parse a MIG UUID, inits CUDA in the parent;
# vLLM's forked EngineCore then dies "Cannot re-initialize CUDA in forked subprocess". Spawn.
export VLLM_WORKER_MULTIPROC_METHOD=spawn

# flashinfer's fused sampler JIT-compiles and needs nvcc at runtime; if you only read
# answer-token logits you don't need it — use the torch sampler.
export VLLM_USE_FLASHINFER_SAMPLER=0

# the Alliance vllm build lacks a usable deep_gemm and the warmup probe raises instead of
# skipping. Disable it (bf16 never needs it; FP8 falls back to cutlass).
export VLLM_USE_DEEP_GEMM=0

# MoE models: vLLM auto-picks a FlashInfer CUTLASS kernel that JIT-builds with nvcc, which
# compute nodes DON'T have -> "Could not find nvcc" at engine init. Force the TRITON backend
# (triton self-compiles, no CUDA toolkit). Harmless for non-MoE models.
export VLLM_USE_FLASHINFER_MOE_FP16=0
export VLLM_USE_FLASHINFER_MOE_FP8=0
```

**No `nvcc` on compute nodes** — only the driver. Any JIT/CUTLASS kernel build fails. Prefer self-compiling backends (TRITON) or `module load cuda` to provide a toolkit.

**Redirect caches onto `$SCRATCH`** (they have hundreds of thousands of small files; `$HOME`/`$PROJECT` have inode quotas):
```bash
export TRITON_CACHE_DIR="$SCRATCH/triton_cache"
export VLLM_CACHE_ROOT="$SCRATCH/vllm_cache"
export TORCH_HOME="$SCRATCH/torch"
mkdir -p "$TRITON_CACHE_DIR" "$VLLM_CACHE_ROOT" "$TORCH_HOME"
```

**vLLM can't parse MIG device UUIDs.** On a MIG slice `CUDA_VISIBLE_DEVICES` holds `MIG-<uuid>`; vLLM `int()`-parses it and crashes (`vllm#7211`, `vllm#13815`). Patch the installed copy once at setup time (in `vllm/platforms/interface.py`) to fall back to physical index 0 — a MIG job sees exactly one slice, and capability checks resolve to the parent GPU, which is correct:

```python
#   try:    return int(physical_device_id)
#   except ValueError:  return 0   # MIG slice: one visible slice, parent GPU 0
```
Make the patch **idempotent and loud**: re-running setup must not double-apply, and it must fail clearly if vLLM's source moves (so you notice on the next upgrade).

**Don't add `deep_gemm` to the install.** Its wheelhouse build pins an older torch that conflicts with vllm's, sending pip into long backtracking. Disable the feature with the env var instead.

**Other knobs:** `gpu_memory_utilization` ~0.90 with `VLLM_MEMORY_PROFILER_ESTIMATE_CUDAGRAPHS=1` for better CUDA-graph accounting; FP8 frees VRAM for a bigger KV cache (larger batches). `max_tokens` is **not** a decode-speed lever under grammar-guided JSON + `temp=0` — generation stops at the JSON close regardless; to cut decode you must shorten the *schema*, which changes semantics → validate quality.
