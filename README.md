# SISTER L1 preprocess bundle

SISTER (Space-based Imaging Spectroscopy and Thermal PathfindER) was JPL's pathfinder workflow for the Surface Biology and Geology (SBG) mission. Its first step, the L1 preprocess PGE, takes an imaging spectroscopy granule in its native format (AVIRIS-Classic, AVIRIS-NG, DESIS, PRISMA or EMIT) and writes the common L1B products every later SISTER step reads: an ENVI radiance cube (`L1B_RDN`), a location cube (`_LOC`: longitude, latitude, elevation) and an observation cube (`_OBS`: path length, sensor and sun angles, phase, slope, aspect, cosine i, UTC time). The SISTER V004 products made this way are archived at the ORNL DAAC (dataset 2335).

The PGE itself is a thin driver: the sensor readers live in the `sister` package and the ENVI and image tools in `hytools`. This repository holds all three as git submodules, each pinned to the release the PGE's `install.sh` names, so the step can be cloned in one go:

```sh
git clone --recurse-submodules https://github.com/gchang/sister-preprocess-bundle.git
```

| Submodule | Upstream | Commit | Role |
|---|---|---|---|
| `sister-preprocess` | https://github.com/sister-jpl/sister-preprocess | f613ef62e061f87aee64b63b11a5f3c3e105341f (tag `3.0.0`) | the PGE: `pge_run.sh`, `generate_runconfig.py`, `preprocess.py` |
| `sister` | https://github.com/sister-jpl/sister | 4e5e4a2e5d856c8d6ac26b93728cf493bfb36a9f (tag `1.3.1`) | sensor readers (`sister.sensors.aviris` and others), installed as a Python package |
| `hytools` | https://github.com/EnSpec/hytools | 3bc81b5541cfee1c297cc6f91cfde2c1e69fadfb (tag `1.5.0`) | ENVI reading and writing, installed as a Python package |

`sister-preprocess/install.sh` clones `https://github.com/EnSpec/sister.git`; that repository has moved to `sister-jpl/sister` (the old address redirects, and tag `1.3.1` is the same commit). Nothing inside the submodules is changed.

## Licenses

Each submodule keeps its own license: `sister-preprocess` is Apache-2.0, `sister` and `hytools` are GPL-3.0. See the `LICENSE` file in each.

## Software

The PGE was built on MAAP from `sister-preprocess/install.sh`: a conda env named `sister` with Python 3.8 and GDAL, then pip. The same env with conda-forge packages, from the bundle root:

```sh
conda create -n sister -c conda-forge --override-channels python=3.8 gdal=3.6.2 pip
conda activate sister
pip install pystac==1.8.4
pip install spectral==0.23.1
pip install ./sister      # sister 1.3.1; its setup.py pins numpy==1.19.2
pip install ./hytools     # hytools 1.5.0, after sister, as install.sh does
```

`install.sh` takes GDAL from the Anaconda defaults channel with no version; conda-forge `gdal=3.6.2` is what the check below used.

## Data

- AVIRIS-Classic input: one flight line as the JPL tarball. The example used here, and in `sister-preprocess/README.md`, is `f110513t01p00r03` (Grand Mesa, Colorado, 2011-05-13): https://popo.jpl.nasa.gov/avcl/y11_data/f110513t01p00r03.tar.gz (1,972,497,555 bytes, sha256 `c24a4ecbc8470f281b1421162c1a6499d41a3cc579f7dd82a9e108fb84ac8b2a`, public, no login).
- Official SISTER V004 output for the same line, to compare with: ORNL DAAC dataset 2335, "SISTER Workflow V004" (CMR collection `C3114843226-ORNL_CLOUD`, Earthdata login), files `EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004{.bin,.hdr,_LOC.bin,_LOC.hdr,_OBS.bin,_OBS.hdr}` under https://data.ornldaac.earthdata.nasa.gov/protected/sister/SISTER_Workflow_V004/data/.
- AVIRIS-Classic needs nothing else. PRISMA and DESIS download Copernicus DEM tiles from `https://copernicus-dem-30m.s3.amazonaws.com/` at run time, and PRISMA also needs a Landsat reference tarball next to the raw file; those paths were not checked here.

Resources for this scene: 12.7 GB of memory at peak, so allow at least 14 GB (the radiance cube is held as float32 while it is averaged to 30 m), and about 12 GB of disk in the work folder while it runs (the 2.0 GB tarball expands to 9.2 GB in `temp/`; 1.9 GB remain at the end, mostly the copied tarball). It takes about 4 to 5 minutes on 4 cores and needs no network for AVIRIS-Classic.

## Running one flight line

From the bundle root, in a new empty work folder (the PGE creates `output/` and `temp/` there and stops if they exist):

```sh
conda activate sister
mkdir work && cd work
cp /path/to/f110513t01p00r03.tar.gz .
../sister-preprocess/pge_run.sh --raw_dataset f110513t01p00r03.tar.gz --crid 004
```

`--crid 004` gives the official V004 file names; the default is `000`. `--experimental True` (the default) adds the `EXPERIMENTAL-` prefix the archived files carry.

The products land in a STAC catalog folder (`catalog.json`) with one subfolder per dataset. Each holds the `.bin`, `.hdr`, STAC `.json` and an empty `.met.json`; the radiance folder also holds the browse `.png`, the `.runconfig.json` and the `.log`:

```
output/EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004/
    EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004/        radiance (83 x 722 x 224, 30 m)
    EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004_LOC/
    EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004_OBS/
```

Those three subfolders are the `--radiance_dataset`, `--location_dataset` and `--observation_dataset` inputs of the next step, `sister-isofit` (a separate bundle, `sister-isofit-bundle`), and the `_OBS` folder is also an input of `sister-reflect_correct`.

## Known traps

- **`source activate sister`.** `pge_run.sh` starts with that line. Inside the MAAP image, where an env called `sister` exists and conda's `activate` script is on `PATH`, it switches to that env. Anywhere else the line fails with `activate: No such file or directory` and, since the script has no `set -e`, carries on with whatever `python` is on `PATH`. Activate the env yourself before running (as above), and do not keep an unrelated env called `sister` that the line could switch to.
- **`numpy==1.19.2` breaks GDAL's numpy bridge.** `sister/setup.py` pins numpy 1.19.2, which pip installs under a conda GDAL built for a newer numpy. `from osgeo import gdal` still works, but `from osgeo import gdal_array` fails with "module compiled against API version 0xe". The AVIRIS-Classic path never imports `gdal_array`, so this example runs; other sensor paths were not checked.
- **`/app` paths and `SISTER_APP_DIR`.** Not needed for this step: the PGE finds its own `data/` folder from its location. The `/app` layout matters only for `sister-isofit`, which hard-codes `/app/6s` and `/app/sRTMnet_v120.h5` (`SISTER_APP_DIR` is not read by the released code).
- **ISOFIT reruns.** Not this step, but for the next one: never rerun an interrupted `sister-isofit` job in the same work folder; start again in an empty one.
- **Reproducibility.** This step is deterministic. The later `sister-reflect_correct` step is not (its BRDF fit draws an unseeded random sample).

## Checked

Checked on 2026-10-25 from a fresh `git clone --recurse-submodules`, with the env built as above (Python 3.8.18, GDAL 3.6.2, numpy 1.19.2, scipy 1.9.3, scikit-image 0.19.3, numba 0.56.4, hy-tools 1.5.0), on `f110513t01p00r03.tar.gz`, with no network: the radiance and location cubes and all three headers are byte-identical to the official V004 `L1B_RDN`, `_LOC` and `_OBS` files at ORNL DAAC. The observation cube differs in one value of 599,260 (to-sun zenith, by 1.9e-6 degrees, the last bit of a float32). The run took 5 min 4 s on 4 cores with a peak of 12.7 GB.
