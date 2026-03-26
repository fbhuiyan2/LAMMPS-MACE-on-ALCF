# Installation

Symmetrix-specific details can be found [here](https://github.com/wcwitt/symmetrix/tree/main/pair_symmetrix).

Original Symmetrix cannot be used with Sphericart on Aurora. Follow the instructions below to install Lammps-Symmetrix with Sphericart on Aurora. This build provides some speed-up compared to non-Sphericart version of Lammps.

```
# create installation dir and download files
mkdir lammps-symmetrix && cd lammps-symmetrix
git clone -b release https://github.com/lammps/lammps

# Cloning Intel GPU Modified Symmetrix
git clone --recurse-submodules https://github.com/alvarovm/symmetrix
cd symmetrix

# Deinit and remove the old submodule
git submodule deinit -f -- libsymmetrix/external/sphericart
git rm -f libsymmetrix/external/sphericart
rm -rf .git/modules/libsymmetrix/external/sphericart
git submodule add https://github.com/alvarovm/sphericart libsymmetrix/external/sphericart
cd ..

# Symmetrix specific steps
cd symmetrix/pair_symmetrix
./install.sh ../../lammps
cd ../..


# obtain a compute node 
qsub -I -l select=1,walltime=1:00:00,place=scatter -l filesystems=home:flare -A your_project -q debug

# Load modules
module load frameworks   # required for Python
module load cmake

# this requires cmake 3.27 or higher. Check version
cmake --version

# cd to lammps-symmetrix dir; then cd to lammps
cd lammps
mkdir build && cd build

# Create a file called cmake.sh and paste the content from the next block there
vi cmake.sh
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
Vanilla Aurora.md build vs Alvaro's Sphericart build weak scaling test.
![Weak scaling performance comparison of Vanilla Aurora build vs Alvaro's Sphericart build](https://github.com/user-attachments/assets/2316fcf6-5225-4365-9ae2-b2bc60426791)

**Number of atoms:** 8512

| Build                     | Performance (ns/day) | Speedup vs matching Vanilla baseline |
|---------------------------|----------------------|---------------------------------------|
| Symmetrix w/o Sphericart (Aurora-6 tile)  | 0.667                | (baseline)                            |
| Symmetrix with Sphericart (Aurora-6 tile) | 0.822                | +23.2%                                |
| Symmetrix w/o Sphericart (Aurora-12 tile) | 1.139                | (baseline)                            |
| Symmetrix with Sphericart (Aurora-12 tile)| 1.475                | +29.5%                                |
