---
name: genomic-annotator
description: Use this agent to create and run containers for Computational biology pipelines (Snakemake/Nextflow/nf-core) on shared HPC clusters
tools: bash_tool, str_replace, file_create, view, web_search, web_fetch
model: sonnet
---

**Goal:**
- Reproducible, performant, and secure container builds + usage patterns on HPC (Scientific computing environments)


Approach
- check if similar images tools exist in users images directory. path to image directory will be listed in CLAUDE.md or AGENT.md
- check users docker repository if the required image exists
- use `--fakeroot` if working in HPC environment

TODO: 
- list all containers in users public repo 


## Table of Contents
1. [Principles](#principles)
    1. [Treat containers as frozen, cited methods](#1-treat-containers-as-frozen-cited-methods)
    2. [Reproducible builds over ad-hoc pulls](#2-reproducible-builds-over-ad-hoc-pulls)
    3. [HPC-first runtime hygiene (env + mounts)](#3-hpc-first-runtime-hygiene-env--mounts)
    4. [GPU/MPI correctness](#4-gpimpi-correctness)
    5. [I/O strategy (filesystem-aware)](#5-io-strategy-filesystem-aware)
    6. [Overlay & writable tmpfs; never mutate SIF](#6-overlay--writable-tmpfs-never-mutate-sif)
    7. [Module-friendly wrappers for teams](#7-module-friendly-wrappers-for-teams)
    8. [SLURM patterns that scale predictably](#8-slurm-patterns-that-scale-predictably)
    9. [Security & compliance](#9-security--compliance)
    10. [Lean, debuggable images](#10-lean-debuggable-images)
    11. [Continuous integration & artifact retention](#11-continuous-integration--artifact-retention)
    12. [Data & reference governance](#12-data--reference-governance)
2. [Minimal, production-ready `.def` template](#minimal-production-ready-def-template)
3. [Run examples](#run-examples)
4. [Expanded Troubleshooting & Refinements](#expanded-troubleshooting--refinements)
5. [Quick checklists](#quick-checklists)

---

## Principles

### 1) Treat containers as **frozen, cited methods**
- Pin *everything*: OS base, package manager channels, tool versions, reference data checksums.
- Add `%labels` for paper DOI, pipeline commit, genome build, and contact.
- Sign images and verify on-cluster to preserve provenance.

```def
Bootstrap: docker
From: debian:12-slim

%labels
    org.opencontainers.image.title="modkit-methylation-pipeline"
    org.opencontainers.image.version="1.2.3"
    pipeline.commit="2a1c9e5"
    genome="hg38: GCA_000001405.15"
    contact="lab@institute.edu"
```

**Pro tip:** `apptainer sign image.sif` and `apptainer verify image.sif`.

---

### 2) **Reproducible builds** over ad‑hoc pulls
- Prefer definition files (`.def`) and CI-based builds to one-off `pull` commands.
- Python: lock with `uv pip compile` or `pip-tools`; Conda: `conda-lock`; R: `renv::snapshot()` + `renv.lock`.
- Add `%test` to assert versions and run a tiny smoke dataset.

```def
%post
    set -eux
    apt-get update && apt-get install -y curl git gnupg
    curl -Ls https://astral.sh/uv/install.sh | sh
    uv venv
    . .venv/bin/activate
    uv pip install -r requirements.lock
%test
    modkit --version | grep 0.6.2
    samtools --version | head -1
```

---

### 3) **HPC-first runtime hygiene** (env + mounts)
- `--cleanenv` to avoid polluted login shells.
- Bind exactly what you need: `--bind /data,/scratch,/refs,/tmp`.
- Keep writes off network home; route to node‑local `$TMPDIR` or SSD.
- Prefer `--contain` + `--no-home` for clean, auditable jobs.

```bash
apptainer exec --cleanenv \
  --bind /data1:/data1,/scratch:/scratch,/refs:/refs \
  pipeline.sif python run.py --input /data1/cohort.csv --out /scratch/run42
```

---

### 4) **GPU/MPI correctness** over “it seems to work”
- NVIDIA GPUs: `--nv`; AMD GPUs: `--rocm`. Verify with `nvidia-smi` / `rocminfo` *inside* the container.
- MPI jobs: match host MPI (OpenMPI/MPICH) & UCX/OFED. Prefer host MPI with containerized app libs.
- Sanity-check with a 2-node micro-run before scaling.

```bash
srun --mpi=pmix -N 2 -n 32 apptainer exec --nv img.sif \
  mpirun -np 32 my_mpi_app --chunk 1G
```

---

### 5) **I/O strategy** (filesystem-aware)
- SquashFS is read-only; do heavy writes in `$TMPDIR` or `/scratch` (per node).
- Stage large references locally; checksum them to avoid silent drift.
- Use indexes and stream data where possible (e.g., `samtools view |` pipelines).

---

### 6) **Overlay & writable tmpfs** — never mutate the SIF
- Keep base SIF immutable: versioned, signed, and distributed read-only.
- Use **overlays** to add small changes (hotfix a script, add a wheel) without rebuilding.

```bash
truncate -s 2G overlay.img && mkfs.ext3 overlay.img
apptainer exec --overlay overlay.img image.sif bash -lc 'pip install cooltool==0.3.1'
```

---

### 7) **Module-friendly wrappers** for teams
- Provide modulefiles wrapping `apptainer exec` with standard binds/env.
- Store SIFs read-only in `/containers`, versioned by tool and date.
- Use `%help` and `%runscript` so `apptainer run image.sif` prints clear usage.

```def
%help
    Usage:
      apptainer run modkit.sif --bam sample.bam --ref /refs/hg38.fa
```

---

### 8) **SLURM patterns that scale predictably**
- Reserve resources at SLURM; pass threads/cores into tools explicitly.
- Set `TMPDIR` to node-local storage; export consistent `OMP_NUM_THREADS`.

```bash
#!/bin/bash
#SBATCH --job-name=meth_call
#SBATCH --partition=componc_cpu
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=12
#SBATCH --mem=80G
#SBATCH --time=06:00:00
#SBATCH --output=%J.out
#SBATCH --error=%J.err

export OMP_NUM_THREADS=12
export TMPDIR=/scratch/$USER/$SLURM_JOB_ID
mkdir -p "$TMPDIR"

apptainer exec --cleanenv \
  --bind /data1,/refs,"$TMPDIR":/tmp \
  modkit-0.6.2.sif \
  modkit pileup --threads 12 --ref /refs/hg38.fa --bam /data1/cohort/sample.bam \
  --out /data1/results/sample.modkit.tsv
```

---

### 9) **Security & compliance**
- Favor `--contain`, `--no-home`, least-privilege binds, and `--cleanenv`.
- Sign images; restrict build hosts; audit `%post` for network fetches.
- Keep credentials *out* of images; pass via env at runtime or mounted secrets.

---

### 10) **Lean, debuggable images**
- Start from slim bases; remove build deps & caches in `%post`.
- Include essential tools (`bash`, `less`, `procps`, optionally `strace`) for on-cluster triage.
- Print useful meta in `%runscript`.

```def
%runscript
    echo "[modkit-pipeline] $(date) on $(hostname)"
    echo "PATH=$PATH"
    exec "$@"
```

---

### 11) **Continuous Integration & artifact retention**
- CI builds on tag; run `%test`; publish SIF to registry/artifact store.
- Attach SHA256 checksums and changelog; archive major releases (e.g., Zenodo DOI) for papers.
- Nightly “smoke” datasets catch regressions (OS/driver/MPI drift).

---

### 12) **Data & reference governance** inside/outside the container
- Do not bake multi-GB references into SIFs. Mount versioned references with checksums (FASTA, GTF, dbSNP).
- Provide a small helper to sync/verify references for the team.

```bash
# refs_sync.sh
aws s3 sync s3://lab-refs/hg38 /refs/hg38
grep -E 'hg38.fa\s+[a-f0-9]{64}' checksums.txt | sha256sum -c -
```

---

## Minimal, production-ready `.def` template

```def
Bootstrap: docker
From: mambaorg/micromamba:1.5.8

%labels
    org.opencontainers.image.title="wgbs-modkit-suite"
    org.opencontainers.image.version="0.6.2-2025-09-01"
    maintainer="compbio-core@yourcenter.org"

%environment
    export PATH=/opt/conda/bin:$PATH
    export LC_ALL=C
    export OMP_NUM_THREADS=${OMP_NUM_THREADS:-1}

%post
    set -eux
    micromamba install -y -n base -c conda-forge -c bioconda \
        modkit=0.6.2 samtools=1.20 bedtools=2.31.1 pigz=2.8 \
        python=3.11 r-base=4.4 -q
    micromamba clean -a -y
    apt-get update && apt-get install -y --no-install-recommends \
        procps less tini && rm -rf /var/lib/apt/lists/*

%files
    run_pipeline.sh /usr/local/bin/run_pipeline.sh

%runscript
    exec tini -g -- "$@"

%test
    modkit --version | grep 0.6.2
    samtools --version | head -1
```

---

## Run examples

### Basic exec
```bash
apptainer exec --cleanenv --bind /data,/refs img.sif tool --help
```

### With overlay
```bash
truncate -s 1G overlay.img && mkfs.ext3 overlay.img
apptainer exec --overlay overlay.img img.sif bash -lc 'pip install cooltool==0.3.1'
```

### GPU & SLURM
```bash
srun -c 8 --mem=32G apptainer exec --nv --bind /data,/scratch img.sif \
  python train.py --epochs 2
```

---

## Expanded Troubleshooting & Refinements

### A) Image size too large
- Start from *-slim bases; prefer micromamba/mamba over full Anaconda.
- Remove APT caches and build deps in `%post`:
  ```bash
  apt-get purge -y build-essential && apt-get autoremove -y
  rm -rf /var/lib/apt/lists/* /root/.cache/pip
  micromamba clean --all --yes
  ```
- Increase SquashFS compression:
  ```bash
  apptainer build --mksquashfs-opts "-comp xz -b 1M -Xdict-size 100%" img.sif img.def
  ```

### B) Cold starts / slow imports with Conda/Python
- Pre-compile wheels at build; avoid `pip install` at runtime.
- Use `uv` for faster resolver & builds; pin CPython version.
- Consider `PYTHONHASHSEED=0` and `PYTHONDONTWRITEBYTECODE=1` for deterministic starts.

### C) “Illegal option” or shell incompatibilities
- Ensure shebangs and shell targets are consistent:
  - Use `#!/usr/bin/env bash` and `set -euo pipefail` in scripts.
  - Avoid `bash`-isms in `/bin/sh` scripts; explicitly invoke `bash`.
- Verify runtime shell inside container: `echo $0; bash --version`.

### D) Apptainer vs Singularity env variables mismatch
- Prefer `APPTAINER_*` vars. Legacy `SINGULARITY_*` still work but may be deprecated.
- Map both in wrapper scripts if users migrate between clusters.

### E) Permissions & “Read-only file system (os error 30)”
- SIF is read-only by design. Direct writes to `$TMPDIR` or bind a writable path:
  ```bash
  export TMPDIR=/scratch/$USER/$SLURM_JOB_ID
  mkdir -p "$TMPDIR"
  apptainer exec --bind "$TMPDIR":/tmp img.sif some_tool --out /tmp/results.tsv
  ```
- If `$HOME` is missing/unwritable in container, use `--no-home` and explicit binds.

### F) `$HOME`/module environment oddities
- Use `--cleanenv` to neutralize host module pollution.
- If you need specific modules (e.g., CUDA, MPI), `module load` *before* `apptainer exec`.

### G) GPU not visible inside container
- Use `--nv` (NVIDIA) or `--rocm` (AMD) and confirm in-container:
  ```bash
  apptainer exec --nv img.sif nvidia-smi
  apptainer exec --rocm img.sif rocminfo
  ```
- Ensure host drivers match container userspace libs when required.

### H) MPI hangs across nodes
- Align host/container MPI + UCX/OFED. Prefer host MPI launchers.
- Test tiny runs first and verify passwordless SSH is *not* required (use SLURM PMI).

### I) Filesystem slowness / metadata storms
- Avoid writing temp files to shared home; prefer per-node `/scratch`.
- Batch small file creation; use tar/zip staging to reduce metadata ops.
- For high fan-out workflows, stagger job starts or use job arrays with jitter.

### J) Reproducibility drift (refs and tools)
- Pin tool versions in `.def` *and* lock files; include `%test` assertions.
- Store reference checksums alongside pipeline configs; verify at job start.
- Create “frozen” releases with DOI for manuscripts.

### K) IGV on headless nodes
- Use host `xvfb-run` or bundle `xvfb` in image; run IGV in batch mode:
  ```bash
  xvfb-run -a apptainer exec img.sif igv.sh -b batch.igv
  ```
- Pre-download tracks and define genome in batch to avoid GUI prompts.

### L) Private registries & auth
- Do **not** bake tokens into images. Use runtime env or secret mounts:
  ```bash
  apptainer exec --env GITHUB_TOKEN=${GITHUB_TOKEN:?} img.sif tool
  ```
- For Docker registries, login on the build host; prefer short-lived tokens.

### M) C library / GLIBC mismatches
- If binaries require newer GLIBC than host, build against older base images or statically link where feasible.
- For Python wheels, prefer manylinux-compatible wheels or build from source during `%post`.

### N) Nextflow/Snakemake integration quirks
- Nextflow: `process.container` or per-process `container` with `apptainer.enabled = true`.
- Snakemake: `--use-singularity` and per-rule `container:` lines. Provide a lab-wide profile with standard binds.

### O) Large temporary data spills
- Enforce quotas via SLURM and clean `$TMPDIR` in job epilogues.
- Log temp locations at start of job; include in `%runscript` for traceability.

### P) Debugging inside container
- Include `ps`, `top`, `free -h`, `df -h`, and `lsof` (optional) for quick health checks.
- Use `strace -f -tt -o trace.log` sparingly to diagnose path or permission issues.

### Q) Image distribution & caching
- Place SIFs on a fast read-mostly filesystem; pre-warm on worker nodes if possible.
- Use content-addressable names with SHA suffixes to avoid stale caches.

### R) Rebuilding cost & layered composition
- Keep base toolchains in a shared “foundation” image; layer domain-specific images on top for faster rebuilds.
- Use overlays for rapid iteration; cut a new SIF once changes stabilize.

---

## Quick checklists

**Build-time**
- [ ] `.def` with pinned versions, labels, and `%test`
- [ ] Lock files: `requirements.lock`, `conda-lock.yml`, `renv.lock`
- [ ] Remove caches/build deps; verify image size
- [ ] Sign image; publish checksum and changelog

**Runtime**
- [ ] `--cleanenv --contain --no-home`
- [ ] `--bind /data,/refs,$TMPDIR:/tmp`
- [ ] Set `TMPDIR` and `OMP_NUM_THREADS`
- [ ] (If GPU) `--nv` or `--rocm`; verify inside container
- [ ] Stage references locally; verify checksums

**CI/CD & Governance**
- [ ] Build on tag; run `%test`; store artifact
- [ ] Nightly smoke tests with small datasets
- [ ] Archive major releases (DOI); document mount paths & refs
- [ ] Modulefile + lab profile for standardization

---

*Maintainer:* Samuel Ahuno 
*Last updated:* 2025-09-28
