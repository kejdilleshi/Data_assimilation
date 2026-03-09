# Data Assimilation for Aletsch Glacier

This repository contains the configuration and input data for running a data assimilation experiment on the Aletsch Glacier using [IGM](https://github.com/instructed-glacier-model/igm) (Instructed Glacier Model).

The experiment inverts for ice thickness by jointly fitting observed surface velocities and thickness measurements, subject to a smoothness regularization.

## Repository Structure

```
.
├── data/
│   └── input_da.nc          # Input NetCDF file with observations and initial fields
├── experiment/
│   └── params_sr_da.yaml    # Hydra parameter file for the DA experiment
└── README.md
```

## Prerequisites

- **IGM** — Install from the [IGM repository](https://github.com/instructed-glacier-model/igm) (follow instructions there for pip install)
- **Python environment** with TensorFlow, netCDF4, Hydra, and tf_keras
- **Pretrained emulator** — Download `gelu_32x64x5` from:
  [https://uzh-my.sharepoint.com/:u:/g/personal/sebastian_rosier_geo_uzh_ch/IQBw9iYw312ETZpExRsW3dDWAdp9aXjddJP5s4IFZuxpmO4?e=JnypkJ](https://uzh-my.sharepoint.com/:u:/g/personal/sebastian_rosier_geo_uzh_ch/IQBw9iYw312ETZpExRsW3dDWAdp9aXjddJP5s4IFZuxpmO4?e=JnypkJ)

## Setup

### 1. Install IGM

```bash
pip install igm-model
# or install from source:
git clone https://github.com/instructed-glacier-model/igm.git
cd igm
pip install -e .
```

### 2. Install tf_keras (required for TensorFlow compatibility)

```bash
pip install tf_keras==2.17.0
```

### 3. Download and place the emulator

Download the pretrained emulator from the link above, unzip it, and note the absolute path to the `gelu_32x64x5/` folder. Then update `experiment/params_sr_da.yaml`:

```yaml
network:
  pretrained_path: "/absolute/path/to/gelu_32x64x5/"
```

### 4. HDF5 library conflict fix

TensorFlow bundles its own HDF5 library, which conflicts with netCDF4. To avoid `OSError: [Errno -101] NetCDF: HDF error`, you must ensure `netCDF4` is imported **before** TensorFlow.

In `igm/igm_run.py`, add before `import tensorflow as tf`:
```python
os.environ.setdefault("TF_USE_LEGACY_KERAS", "1")
import netCDF4  # must be imported before TensorFlow
```

In `igm/__init__.py`, add at the top:
```python
import os
os.environ.setdefault("TF_USE_LEGACY_KERAS", "1")
import netCDF4  # must be imported before TensorFlow
```

## Running the Experiment

From this repository's root directory:

```bash
igm_run +experiment=params_sr_da
```

For a full stack trace on errors:
```bash
HYDRA_FULL_ERROR=1 igm_run +experiment=params_sr_da
```

Results will be written to `outputs/<date>/<time>/`, which is git-ignored.

## Input Data

The file `data/input_da.nc` contains the following fields on a 94 x 61 grid:

| Variable | Description |
|---|---|
| `x`, `y` | Spatial coordinates |
| `usurf` | Surface elevation |
| `thk` | Ice thickness (initial guess) |
| `icemask` | Ice mask (1 = ice, 0 = no ice) |
| `uvelsurfobs`, `vvelsurfobs` | Observed surface velocity components |
| `thkobs` | Observed ice thickness |
| `usurfobs` | Observed surface elevation |
| `icemaskobs` | Observed ice mask |
| `thkinit` | Initial thickness field |

## Experiment Configuration

The experiment inverts for **ice thickness** using the following objective function:

### Inversion variable
- `thk` (ice thickness) — bounded between 0 and 1000 m, masked by `icemask`

### Misfit terms (data fidelity)
- **Surface velocity**: Gaussian misfit on `(uvelsurf, vvelsurf)` vs observed `(uvelsurfobs, vvelsurfobs)` with `std = 30.0` m/yr
- **Ice thickness**: Gaussian misfit on `thk` vs observed `thkobs` with `std = 10.0` m

### Regularization
- Squared Laplacian penalty on `thk` with `lam = 100000.0` (enforces spatial smoothness)

### Optimizer
- L-BFGS with memory 30 and Hager-Zhang line search
- Maximum 6000 iterations

### Emulator
- Pretrained CNN with 32 layers, 64 filters, kernel size 5, GELU activation
- MOLHO vertical basis, Q1 horizontal basis
- Inputs: `[thk, usurf, slidingco]`

## Tuning

Key parameters to adjust in `experiment/params_sr_da.yaml`:

| Parameter | Effect |
|---|---|
| `std` (velocity misfit) | Lower = tighter fit to velocity observations |
| `std` (thickness misfit) | Lower = tighter fit to thickness observations |
| `lam` (regularization) | Higher = smoother thickness solution |
| `nbitmax` | Maximum number of optimization iterations |
