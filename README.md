# mir-environment

![RitmoMaestro](https://shieldcn.dev/badge/RitmoMaestro-environments-7289da.svg?variant=secondary&logo=lu:Boxes)
![status](https://shieldcn.dev/badge/status-active-22c55e.svg?variant=secondary&logo=lu:CircleCheck)
![stack](https://shieldcn.dev/badge/stack-conda-44a833.svg?variant=secondary&logo=anaconda)

`mir-environment` holds the conda environment definitions of RitmoMaestro, a
system that predicts the beats and downbeats of Latin music live and turns
them into vibrations for dancers. The main environment is `MIR` (uppercase).
The laptop and the training cluster Prometheus both run the project code in
it.

## Quick start

Create `MIR` from this repository. The sibling checkouts `mir-core` and
`mir-desktop-app` must exist next to it, because the file installs
`../mir-core[native]` and `../mir-desktop-app/native-runtime` in editable mode:

```bash
cd /home/hashim/thesis-workspace/mir-environment
conda env create -f environment.yml
conda activate MIR
python -m mir_env.verify_installation
```

`mir_env.verify_installation` is part of `mir-core`.

On a checkout of the `msc-thesis` superproject, `scripts/workspace/init.sh`
does the same and also writes the workspace settings. It asks for the target
(`legion`, `daic` or `delftblue`), creates the environment when it is missing
and updates it otherwise. `legion` is the laptop profile and uses
`environment.yml`; the other two use `environment-hpc.yml`.

## Layout

| File | Environment | Use | GPU |
| --- | --- | --- | --- |
| `environment.yml` | `MIR` | Full stack: training, evaluation, desktop tools, tests. | `pytorch-cuda=12.4` |
| `environment-hpc.yml` | `MIR-hpc` | Host tools on DAIC and DelftBlue: DVC, helper scripts, report publishing. Not a training runtime. | none |
| `environment-apptainer.yml` | `MIR-apptainer` | Base of the Apptainer image that the `apptainer-*` Slurm templates of `mir-train-hpc` run in. | `pytorch-cuda=12.1` |
| `environment-webapp.yml` | `MIR-webapp` | `mir-webapp` development and machines without a GPU. | CPU only |
| `.github/workflows/` | | Project sync workflow. | |

`MIR` also contains the Rust toolchain, CMake and Ninja, ONNX and ONNX
Runtime (through the `native` extra of `mir-core`), TensorFlow 2.19.1 for the
YAMNet classifier, W&B 0.28, DVC with S3 support, and `latexmk` for the run
reports.

## How it connects

- `mir-core` and `mir-desktop-app/native-runtime` are installed into `MIR` in
  editable mode. The native runtime lets the Python desktop code select its
  Rust data plane; a machine without that extension can still use the
  Python-hosted reference backend.
- `mir-train-hpc` runs the staged BeatLab pipeline on Prometheus
  (`cluster/local/conda-*.slurm`) in `MIR`; `MIR_CONDA_ENV` defaults to `MIR`.
  Its `apptainer-*` templates run in the Apptainer image on all three cluster
  profiles. On DAIC and DelftBlue the host tools come from `MIR-hpc`.
- The `msc-thesis` superproject reads these files: `scripts/workspace/init.sh`
  creates `MIR` or `MIR-hpc`, and `scripts/apptainer/build.sh` builds the image
  from `environment-apptainer.yml`.
- `environment-webapp.yml` defines `MIR-webapp` for `mir-webapp`. The script
  `mir-webapp/run.sh` itself starts its backend in `MIR` unless
  `BACKEND_CONDA_ENV` names another environment.

## Status

Status on 2026-10-09.

Current:

- `MIR` is the environment in use on the laptop and on Prometheus.
- `environment-webapp.yml` is unchanged since April 2026.

Kept as history:

- `MIR-hpc` serves DAIC and DelftBlue only. The run reports in
  `mir-train-hpc/docs/` describe runs on Prometheus from 30 June 2026 on. The
  section "History: DAIC and DelftBlue setup" describes the setup on those
  clusters as it was written in spring 2026 (not verified, 2026-10-09).

## Notes on the dependencies

- `numpy<2.0` is a hard constraint: both madmom and BeatNet require it.
- `numba` must come from conda-forge, not pip: the pip wheel lacks proper LLVM
  bindings on some platforms.
- `madmom` is installed from its GitHub repository. Its last release on PyPI
  is 0.16.1 of November 2018 (checked on 2026-10-09).
- `BeatNet` is installed from the fork `Hashim-K/beatnet` at tag `v1.2.0`
  (stale numba pin removed in the fork).
- The HPC bootstrap environment keeps `dvc` and `dvc-s3` on conda to avoid pip
  builds of `pygit2` on cluster login nodes.

## Apptainer image

`environment-apptainer.yml` is the shared headless GPU runtime used by the
Apptainer build in `msc-thesis`:

```bash
cd /path/to/msc-thesis
./scripts/apptainer/build.sh
./scripts/apptainer/smoke-test.sh
```

## History: DAIC and DelftBlue setup

There are no separate full-stack `daic` or `delftblue` environments. Cluster
machines use `environment-hpc.yml` for host-side Git, DVC and helper tools and
the Apptainer image for the runtime.

### DAIC

Manual setup:

```bash
# 1. Load miniconda (DAIC system module)
module use /opt/insy/modulefiles
module load miniconda

# 2. Create the lightweight bootstrap env on a login node
conda env create -f environment-hpc.yml
conda activate MIR-hpc

# 3. Verify bootstrap and report-publishing tooling
dvc version
latexmk -version
```

DAIC's `/tudelft.net` mounts are Windows-based and have pip compatibility
issues. Keep conda environments in `$HOME` (the default); do not relocate them
to project storage. The full training stack runs through the shared Apptainer
image, not through a large conda environment on the login node.

The earlier README asked to add this to `~/.bashrc` on DAIC. It is superseded:
the file `.env.daic` of the superproject now sets these paths.

```bash
module use /opt/insy/modulefiles
module load miniconda
export MIR_DATA_ROOT=/tudelft.net/your-group/datasets
export MIR_OUTPUTS_ROOT=/tudelft.net/your-group/outputs
export MIR_CORE_PATH=/path/to/msc-thesis/repos/mir-core
export MIR_SHARED_ROOT=/tudelft.net/your-group/project-root
export MIR_RUNS_ROOT=/tudelft.net/your-group/project-root/runs
```

Slurm header that the earlier README recommended for A40 nodes. The DAIC
templates in `mir-train-hpc/cluster/daic/` request other values (`--gres=gpu:1`,
16 CPUs, 32 GB, four hours, QoS `short`):

```bash
#SBATCH --partition=general
#SBATCH --gres=gpu:a40:1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=12:00:00
```

### DelftBlue

Manual environment creation on a login node, after installing Miniconda or
Miniforge in `$HOME`:

```bash
conda env create -f environment-hpc.yml
conda activate MIR-hpc
dvc version
latexmk -version
```

Recommended storage split:

- keep repositories and conda in `$HOME`;
- keep large run outputs, the DVC cache and Apptainer images under the project
  storage root, such as `/tudelft.net/staff-umbrella/mirworkspace`.

On DelftBlue worker nodes, do not rely on `conda init`. Source `conda.sh`
directly in the job script before activating the environment:

```bash
module load openssh
module load git
unset CONDA_SHLVL
source "$HOME/miniconda3/etc/profile.d/conda.sh"
conda activate MIR-hpc
```
