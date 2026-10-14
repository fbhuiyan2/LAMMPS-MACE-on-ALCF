# Installation

*Last updated: 2026-10-14. Tested against the LAMMPS `30 Sep 2026` stable release (`patch_30Sep2026`). This guide includes Sphericart support, which is now built directly into [symmetrix](https://github.com/wcwitt/symmetrix).*

Symmetrix-specific details can be found [here](https://github.com/wcwitt/symmetrix/tree/main/pair_symmetrix).

```
# create installation dir and download files
mkdir lammps-symmetrix && cd lammps-symmetrix
git clone -b release https://github.com/lammps/lammps
git clone --recursive https://github.com/wcwitt/symmetrix

# obtain a compute node 
qsub -I -l select=1,walltime=1:00:00,place=scatter -l filesystems=home:flare -A your_project -q debug

# Load modules
module load oneapi/release/2026.1.0
module load gcc/14.3.0
module load mpich/prd/5.0.0.aurora_test.87e2045
module load cmake/3.31.11
```

Of these, only `cmake` needs to be loaded explicitly. The other three (`oneapi`, `gcc`, `mpich`) are shown above to document Aurora's default software stack at the time of writing. Compute nodes (should) already load them by default, and the specific versions may drift in future as Aurora's software/module stack is updated. Run `module list` on a compute node to check what's actually current. As of this writing it shows:
```
Currently Loaded Modules:
  1) gcc/14.3.0   2) oneapi/release/2026.1.0   3) mpich/prd/5.0.0.aurora_test.87e2045   4) libfabric/2.3.1   5) cray-pals/1.8.0   6) cray-libpals/1.8.0
```

If you want to build the Python package (`PKG_PYTHON=ON` in cmake.sh below) and the build may fail with the module setup above. If the cmake.sh script does not run properly, also try `module load frameworks` (which contains conda and Python).

```
# Symmetrix specific steps
cd symmetrix/pair_symmetrix
./install.sh ../../lammps
cd ../..

cd lammps

# this requires cmake 3.27 or higher. Check version
cmake --version

mkdir build && cd build
```

On Aurora, users are encouraged to use CMake to build LAMMPS using the default compilers and the kokkos-sycl-intel.cmake configuration file (path/to/lammps/cmake/presets/kokkos-sycl-intel.cmake) included in the LAMMPS repo.
However, the current version of the preset file has `set(CMAKE_CXX_STANDARD 17 CACHE STRING "" FORCE)`. Symmetrix requires setting `CMAKE_CXX_STANDARD=20`. So, you have to delete or comment out the `set(CMAKE_CXX_STANDARD 17 CACHE STRING "" FORCE)` line in the preset before moving on to the next step.
We will use `-D CMAKE_CXX_STANDARD=20 \` in our cmake command below.

Create a file named cmake.sh with the following contents (note you may not need all the packages. Necessary packages for Symmetrix can be found on their [repo](https://github.com/wcwitt/symmetrix/tree/main/pair_symmetrix)):
```
# note, not all the PKG here are mandatory for symmetrix. Install PKGs as you like or need
cmake \
  -C ../cmake/presets/kokkos-sycl-intel.cmake \
  -D CMAKE_BUILD_TYPE=Release \
  -D CMAKE_INSTALL_PREFIX=$(pwd) \
  \
  -D CMAKE_CXX_STANDARD=20 \
  -D CMAKE_CXX_STANDARD_REQUIRED=ON \
  \
  -D BUILD_MPI=ON \
  -D BUILD_OMP=ON \
  -D BUILD_SHARED_LIBS=ON \
  -D PKG_PYTHON=ON \
  -D PKG_MEAM=ON \
  -D PKG_REAXFF=ON \
  -D PKG_ELECTRODE=ON \
  -D PKG_REPLICA=ON \
  -D PKG_MANYBODY=ON \
  -D PKG_MOLECULE=ON \
  -D PKG_MISC=ON \
  -D PKG_RIGID=ON \
  -D PKG_ML-PACE=ON \
  \
  -D SYMMETRIX_KOKKOS=ON \
  -D SPHERICART_OPENMP=ON \
  -D SPHERICART_ENABLE_SYCL=ON \
  -D SPHERICART_SYCL_DEVICE=gpu \
  ../cmake
```

Run script:
`sh cmake.sh`

Build:
`make -j 32`

# Notes on running simulations

Aurora has 2 tiles per GPU. Simulation speed was highest with 1 tile-1 rank, and speed started to drop with more tiles/GPUs. for multi tile or multi GPU runs, use the gpu affinity scripts (gpu_dev_compact or gpu_tile_compact, [instructions here](https://docs.alcf.anl.gov/aurora/running-jobs-aurora/#1-binding-mpi-ranks-to-gpus-using-gpu_tile_compactsh-and-gpu_dev_compactsh-scripts)). Use `-k on g 1 -sf kk -pk kokkos` (note `g 1`) when running the LAMMPS binary.

# Performance tests
![Weak scaling performance for Symmetrix with and without Sphericart on Aurora](https://github.com/user-attachments/assets/f69dc496-4745-4e9a-89a3-ca5837f9d8d3)

The table below shows a weak scaling test comparing the Symmetrix Aurora.md build with the Symmetrix + Sphericart build.


**Number of atoms:** 8512

| Build                     | Performance (ns/day) | Speedup vs matching Vanilla baseline |
|---------------------------|----------------------|---------------------------------------|
| Symmetrix w/o Sphericart (Aurora-6 tile)  | 0.667                | (baseline)                            |
| Symmetrix with Sphericart (Aurora-6 tile) | 0.822                | +23.2%                                |
| Symmetrix w/o Sphericart (Aurora-12 tile) | 1.139                | (baseline)                            |
| Symmetrix with Sphericart (Aurora-12 tile)| 1.475                | +29.5%                                |
