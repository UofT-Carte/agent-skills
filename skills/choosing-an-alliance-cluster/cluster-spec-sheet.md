# Alliance cluster spec sheet

Per-cluster detail backing `SKILL.md`. **Specs drift and clusters change** — confirm via the `alliance-docs` MCP (the per-cluster page, `Using_GPUs_with_Slurm`, `National_systems`). Availability dates are included so you can gauge staleness. All clusters share Slurm + Lmod + `StdEnv/2023` + the CVMFS wheelhouse, and all enforce **MFA on interactive SSH** (see `running-on-alliance-hpc`).

## Allocation / account model (applies everywhere)

- **`def-<pi>`** — default RAP, auto-created per PI; **lowest, opportunistic priority**; all sponsored users are members. General-purpose clusters.
- **`rrg-<pi>` (RRG)** / **`rpp-<pi>` (RPP)** — RAC awards (annual competition, year starts ~early April); **higher priority**, granted as specific allocations on specific resources (`<cluster>-cpu`/`-gpu`/`-storage`) and only on awarded clusters.
- **`aip-<pi>`** — PAICE AIP-type RAP, required for TamIA/Killarney/Vulcan; AI researchers only; those clusters are geo-blocked to Canada.
- Jobs run under your PI's allocation; with >1 group you must pass `--account=`. Per-cluster **access must still be requested in CCDB** (Resources → Access Systems), separate from holding an allocation.

---

## General-purpose clusters

### Fir — SFU, Burnaby BC (avail. 2025-08-11; replaces Cedar)
- **Endpoints:** login `fir.alliancecan.ca`; automation `robot.fir.alliancecan.ca`; Globus `alliancecan#fir-globus`; JupyterHub `jupyterhub.fir.alliancecan.ca`.
- **GPU nodes:** 160 × (1× AMD EPYC 9454 Zen4, 48 cores, 1125 GB, 7.84 TB NVMe, **4× H100 SXM5 80 GB NVLink**). **MIG** on ~half (`h100_1g.10gb`/`2g.20gb`/`3g.40gb`). Rec ≤**12 cores/GPU**.
- **CPU nodes:** 864 × (2× EPYC 9655 Zen5, 192 cores, 750 GB); 8 bigmem (6000 GB). Tune: `--cpus-per-task=8` per CCD; `--ntasks-per-node=24`.
- **Internet on compute nodes: YES** (full access).
- **Storage:** 51 PB DDN Lustre. HOME small + daily backup (cannot grow → use /project); SCRATCH large, no backup, auto-purged; PROJECT large/adjustable + daily backup. Node-local 7.84 TB NVMe.
- **Interconnect:** InfiniBand NDR; CPU islands 27:5 blocking over 216×192-core nodes; GPU 2:1.
- **Policy:** crontab unsupported; jobs 1 h–7 d (5 min test min). #78 on TOP500 (Jun 2025).

### Narval — ÉTS, Montreal QC (avail. since 2021-10)
- **Endpoints:** login/DTN `narval.alliancecan.ca`; Globus "Compute Canada - Narval"; portal `portail.narval.calculquebec.ca`.
- **GPU nodes:** 159 × (2× AMD EPYC 7413 Zen3, 48 cores, 498 GB, 3.84 TB SSD, **4× A100 SXM4 40 GB NVLink**). **MIG** 4 sizes: `a100_1g.5gb`/`2g.10gb`/`3g.20gb`/`4g.20gb`. Rec ≤**12 cores/GPU**.
- **CPU nodes:** 1145 × (2× EPYC 7532 Zen2, 64 cores, 250 GB); 33 × 2009 GB; 3 × 4000 GB.
- **Internet on compute nodes: NO** (air-gapped; exceptions by request).
- **A100 caveats:** **no FP8**; 40 GB ceiling. **CPU: AVX2 only, no AVX512** — compile with `-march=core-avx2`; avoid Intel `-xHOST`/`-march=native` (degrade to Pentium on AMD). FlexiBLAS preferred over MKL.
- **Storage:** Lustre. HOME 64 TB (small quota, daily backup); SCRATCH 5.7 PB (no backup, **auto-purged**); PROJECT 35 PB (adjustable, daily backup).
- **Interconnect:** InfiniBand HDR (Mellanox); islands of 48/56 nodes non-blocking (up to 3584 cores), else 4.7:1.
- **Policy:** crontab no; jobs 1 h–7 d; ≤**1000** jobs queued+running.

### Nibi — SHARCNET, U Waterloo ON (avail. 2025-07-31)
- **Endpoints:** login `nibi.alliancecan.ca`; automation `robot.nibi.alliancecan.ca`; OOD `ondemand.sharcnet.ca`; Globus `alliancecan#nibi`; portal `portal.nibi.sharcnet.ca`. 134,400 cores, 288 H100.
- **GPU nodes:** 36 × (2× Intel 8570, 112 cores, 2000 GB, **11 TB** node-local, **8× H100 SXM 80 GB NVLink**) → bundle ≈ **14 cores / 250 GB per GPU**. **MIG** ~half (`h100_1g.10gb`/`2g.20gb`/`3g.40gb`). Plus **6 × MI300A nodes** (4× AMD MI300A APU, 96 cores, 495 GB, **unified CPU+GPU memory**).
- **CPU nodes:** 700 × (2× Intel 6972P, 192 cores, 748 GB, 3 TB local); 10 bigmem (6000 GB).
- **Internet on compute nodes: YES** (all nodes; no proxy needed).
- **MI300A:** **ROCm only** (CUDA won't run); schedule **whole nodes**; software stack nascent (as of 2026-05, no ROCm modules — build against `/opt/rocm`).
- **Storage:** 25 PB all-SSD **VAST**. Charged for **apparent file size** (no compression credit). **SCRATCH: 1 TB soft / 60-day grace, then writes blocked — NOT an age-purge** (experimental mechanism). `/project` & `/nearline` dirs are self-created (`mkdir`). `oops` recovers HOME/PROJECT from 30-min snapshots (kept 2 weeks).
- **Interconnect:** Nokia 200/400 G **Ethernet** (non-blocking among GPU nodes); 2:1 blocking at 400 G uplinks. Use `#SBATCH --switches=1` for tight multinode.

### Rorqual — ÉTS, Montreal QC (avail. 2025-06-19; replaces Béluga)
- **Endpoints:** login/DTN `rorqual.alliancecan.ca`; automation `robot.rorqual.alliancecan.ca`; Globus `alliancecan#rorqual`; JupyterHub `jupyterhub.rorqual.alliancecan.ca`; portal `metrix.rorqual.alliancecan.ca`.
- **GPU nodes:** 81 × (2× Intel Xeon Gold 6448Y, 64 cores, 498 GB, 3.84 TB NVMe, **4× H100 SXM5 80 GB**). **MIG** ~half (`h100_1g.10gb`/`2g.20gb`/`3g.40gb`). Rec ≤**16 cores/GPU** (highest of the fleet).
- **CPU nodes:** 670 × (2× EPYC 9654 Zen4, 192 cores, 750 GB); 8 × 3013 GB. `$SLURM_TMPDIR` enlargeable via `--tmp=xG` (370–3360).
- **Internet on compute nodes: NO** (air-gapped; exceptions by request).
- **Storage:** Lustre. HOME 116 TB (small quota, daily backup); SCRATCH 6.5 PB via `$HOME/links/scratch` (no backup, **auto-purged**); PROJECT 62 PB via `$HOME/links/projects/<name>` (adjustable, daily backup).
- **Interconnect:** InfiniBand HDR 200 Gb/s; max blocking 5.667:1; CPU islands ≤31×192-core non-blocking.
- **Policy:** crontab no; jobs 1 h–7 d; ≤**1000** jobs queued+running.

---

## Large-parallel cluster

### Trillium — SciNet, U Toronto ON (avail. 2025-08-07; replaces Niagara)
- **Endpoints:** login `trillium.alliancecan.ca` / `trillium-gpu.alliancecan.ca`; DTN `tri-dm{2,3,4}.scinet.utoronto.ca`; OOD `ondemand.scinet.utoronto.ca`; Globus `alliancecan#trillium` (+ `alliancecan#hpss` nearline); portal `my.scinet.utoronto.ca`.
- **GPU nodes:** 63 × (1× EPYC 9654 Zen4, 96 cores, 749 GB, **4× H100 SXM 80 GB NVLink**). **No MIG.**
- **CPU nodes:** 1224 × (2× EPYC 9655 Zen5, 192 cores, 749 GB).
- **Scheduling culture:** *large-parallel* — favors **whole-node** jobs that fill nodes; transitioning users from Niagara. (Interim power/cooling constraints during Niagara shutdown.)
- **Storage:** 29 PB **VAST** NVMe (714 GB/s read), POSIX + S3; **114 PB HPSS tape** nearline (dual-copy archive).
- **Interconnect:** Nvidia **NDR InfiniBand, fully non-blocking** — 400 Gb/s CPU, **800 Gb/s GPU**. Best in the fleet for tightly-coupled multinode.
- Liquid-cooled, PUE < 1.03.

---

## PAICE — AI-dedicated clusters (require an `aip-` RAP; Canada-only access)

### TamIA — U Laval QC, with Mila + Calcul Québec (avail. 2025-03-31)
- **Endpoints:** login/DTN `tamia.alliancecan.ca`; automation `robot.tamia.ecpia.ca`; portal `portail.tamia.ecpia.ca`.
- **GPU nodes:** 42 × (2× Intel Xeon Gold 6442Y, 48 cores, 512 GB, 7.68 TB SSD, **4× H100 SXM 80 GB NVLink**); plus **H200** nodes (`--gpus=h200:8`). 4 CPU-only nodes.
- **Scheduling:** **whole-node only — every job must use all 4 GPUs** of each node (`--gpus=h100:4` / `h200:8`). **Max job 24 h.** ≤1000 jobs. VSCode banned on login nodes.
- **Internet on compute nodes: NO** (air-gapped).
- **Storage:** Lustre; HOME/SCRATCH/PROJECT (HOME/PROJECT backup ETA summer 2025; SCRATCH purged).
- **Interconnect:** InfiniBand NDR, non-blocking fat-tree; each H100 on an NDR200 port.

### Killarney — U Toronto ON, with Vector Institute + SciNet (avail. 2025-06-09)
- **Endpoints:** login `killarney.alliancecan.ca`; status `status.alliancecan.ca/system/Killarney`.
- **Standard tier:** 168 × (2× Intel Xeon Gold 6338, 64 cores, 512 GB, 350 GB SSD, **4× L40S 48 GB**) → 672 L40S.
- **Performance tier:** 10 × (2× Intel Xeon Gold 6442Y, 48 cores, 2048 GB, 800 GB NVMe, **8× H100 SXM 80 GB**) → 80 H100.
- **Eligibility:** Vector-affiliated PIs w/ CCAI Chairs, or AI researchers; **geo-blocked** to safe countries.
- **Storage:** 1.7 PB all-NVMe VAST; HOME small + daily backup; SCRATCH purged; PROJECT adjustable + daily backup.
- **Interconnect:** Standard IB HDR100 (100 Gb/s); Performance 2× HDR200 (400 Gb/s).

### Vulcan — U Alberta, with Amii (avail. 2025-04-15)
- **Endpoints:** login `vulcan.alliancecan.ca`; portal `portal.vulcan.alliancecan.ca`; status page available.
- **GPU nodes:** 252 × (2× Intel Xeon Gold 6448Y, 64 cores, 512 GB, **4× L40S 48 GB**) → 1008 L40S.
- **Internet on compute nodes:** **Squid proxy with domain whitelist** (not open, not fully air-gapped) — ask support to whitelist a domain.
- **Eligibility:** any AI / applied-AI researcher. **Max job 7 d.**
- **Storage:** ~5 PB Dell PowerScale (NVMe+HDD); HOME small + daily backup; SCRATCH purged; PROJECT adjustable + daily backup.
- **Interconnect:** 100 Gb/s Ethernet with RoCE.

---

## End-of-life — do not target
**Béluga, Cedar, Graham, Niagara** are retired. Migration: Cedar→Fir, Béluga→Rorqual, Niagara→Trillium (Graham workloads spread across the new general-purpose fleet). Cloud (Arbutus etc.) is separate and not Slurm-scheduled.
