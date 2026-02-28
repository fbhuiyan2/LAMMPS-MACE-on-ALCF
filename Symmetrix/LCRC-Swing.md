# Installation

Symmetrix-specific details can be found [here](https://github.com/wcwitt/symmetrix/tree/main/pair_symmetrix).

```
# create installation dir and download files
mkdir lammps-symmetrix && cd lammps-symmetrix
git clone -b release https://github.com/lammps/lammps
git clone --recursive https://github.com/wcwitt/symmetrix

# obtain a compute node
qsub -I -l select=1:ngpus=1 -l walltime=01:00:00 -q gpu -A <project_name>

# Load necessary modules
module add hpcx/2.23-gcc/hpcx-ompi cuda/13.0.0 cmake intel-oneapi-mkl/2023.1.0

cd lammps
mkdir build && cd build
```

You should have the following modules:

```
module list
Currently Loaded Modules:
  1) hpcx/2.23-gcc/hpcx-ompi   2) cuda/13.0.0   3) cmake/3.30.2-ufv3dko   4) intel-oneapi-tbb/2021.9.0   5) intel-oneapi-mkl/2023.1.0
```

Create a script named `cmake.sh` with the following contents:

```
cmake \
    -D CMAKE_BUILD_TYPE=Release \                                                                                                                                                                                                                
    -D CMAKE_CXX_STANDARD=20 \                                                                                                                                                                                                                   
    -D CMAKE_CXX_STANDARD_REQUIRED=ON \
    -D CMAKE_CXX_COMPILER=$(pwd)/../lib/kokkos/bin/nvcc_wrapper \
    -D CMAKE_CXX_FLAGS="${CMAKE_CXX_FLAGS} -march=znver2 -ffast-math" \
    \
    -D BUILD_SHARED_LIBS=ON \
    -D BUILD_OMP=ON \
    -D BUILD_MPI=ON \
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
    -D PKG_KOKKOS=ON \
    -D Kokkos_ENABLE_SERIAL=ON \
    -D Kokkos_ENABLE_OPENMP=ON \
    -D Kokkos_ARCH_ZEN2=ON \
    -D Kokkos_ENABLE_AGGRESSIVE_VECTORIZATION=ON \
    -D Kokkos_ENABLE_CUDA=ON \
    -D Kokkos_ARCH_AMPERE80=ON \
    \
    -D SYMMETRIX_KOKKOS=ON \
    -D SYMMETRIX_SPHERICART_CUDA=ON \
    ../cmake
```

Note, some packages here like PKG_ELECTRODE, PKG_MANYBODY, PKG_MOLECULE etc. are not needed for Symmetrix. Install whatever package is useful to you.

Run script:
`sh cmake.sh`

Build:
`make -j 16`
