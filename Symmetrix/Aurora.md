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

Create a file named cmake.sh with the following contents (note you may not need all the packages. Necessary packages for Symmetrix can be found on their [repo](https://github.com/wcwitt/symmetrix/tree/main/pair_symmetrix)):
```
cmake \
  -D CMAKE_BUILD_TYPE=Release \
  -D CMAKE_INSTALL_PREFIX=$(pwd) \
  -D CMAKE_CXX_FLAGS="${CMAKE_CXX_FLAGS} -march=native -ffast-math" \
  \
  -D CMAKE_CXX_STANDARD=20 \
  -D CMAKE_CXX_STANDARD_REQUIRED=ON \
  \
  -D BUILD_MPI=ON \
  -D BUILD_OMP=ON \
  -D BUILD_SHARED_LIBS=ON \
  -D PKG_KOKKOS=ON \
  -D PKG_PYTHON=ON \
  -D PKG_MEAM=ON \
  -D PKG_REAXFF=ON \
  -D PKG_ELECTRODE=ON \
  -D PKG_QEQ=ON \
  -D PKG_REPLICA=ON \
  -D PKG_MANYBODY=ON \
  -D PKG_MOLECULE=ON \
  -D PKG_MISC=ON \
  -D PKG_RIGID=ON \
  \
  -D Kokkos_ENABLE_SERIAL=ON \
  -D Kokkos_ENABLE_SYCL=ON \
  -D Kokkos_ENABLE_OPENMP=ON \
  -D Kokkos_ARCH_NATIVE=ON \
  -D Kokkos_ARCH_INTEL_PVC=ON \
  -D Kokkos_ENABLE_AGGRESSIVE_VECTORIZATION=ON \
  -D FFT_KOKKOS=MKL_GPU \
  -D SYMMETRIX_KOKKOS=ON \
  ../cmake
```

Run script:
`sh cmake.sh`

Build:
`make -j 16`

