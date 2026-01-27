# Installation

Symmetrix-specific details can be found [here](https://github.com/wcwitt/symmetrix/tree/main/pair_symmetrix).

```
# create installation dir and download files
mkdir lammps-symmetrix && cd lammps-symmetrix
git clone -b release https://github.com/lammps/lammps
git clone --recursive https://github.com/wcwitt/symmetrix

# obtain a compute node 
qsub -I -l select=1,walltime=1:00:00,place=scatter -l filesystems=home:flare -A your_project -q debug

# Load modules
module load frameworks   # required for Python
module load cmake

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
  \
  -D SYMMETRIX_KOKKOS=ON \
  ../cmake
```

Run script:
`sh cmake.sh`

Build:
`make -j 16`

# Notes on running simulations

Aurora has 2 tiles per GPU. Simulation speed was highest with 1 tile-1 rank, and speed started to drop with more tiles/GPUs. for multi tile or multi GPU runs, use the gpu affinity scripts (gpu_dev_compact or gpu_tile_compact, [instructions here](https://docs.alcf.anl.gov/aurora/running-jobs-aurora/#1-binding-mpi-ranks-to-gpus-using-gpu_tile_compactsh-and-gpu_dev_compactsh-scripts)). Use `-k on g 1 -sf kk -pk kokkos` (note `g 1`) when running the LAMMPS binary.
