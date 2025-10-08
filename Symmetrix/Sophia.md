# Installation

Symmetrix-specific details can be found [here](https://github.com/wcwitt/symmetrix/tree/main/pair_symmetrix).

```
# create installation dir and download files
mkdir lammps-symmetrix && cd lammps-symmetrix
git clone -b release https://github.com/lammps/lammps
git clone --recursive https://github.com/wcwitt/symmetrix

# obtain a compute node
qsub -I -l select=1 -l walltime=02:00:00 -q by-gpu -l filesystems=home:eagle -A your_project

# Load modules
module load conda
module load compilers/openmpi/5.0.3

# Symmetrix specific steps
cd symmetrix/pair_symmetrix
./install.sh ../../lammps
cd ../..

cd lammps

# this requires cmake 3.27 or higher. Must install suitable cmake version first. here, I have it setup already
# Add cmake to path and check version
PATH=$(pwd)/../../cmake-3.27.6-linux-x86_64/bin:$PATH
cmake --version

export CUDA_HOME=$(dirname $(dirname $(which nvcc)))
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH

# i have my own kokkos installed at kokkosinstall

export KOKKOS_PATH=$(pwd)/../../kokkosinstall
export PATH=$KOKKOS_PATH/bin:$PATH
export LD_LIBRARY_PATH=$KOKKOS_PATH/lib:$LD_LIBRARY_PATH
export LIBRARY_PATH=$KOKKOS_PATH/lib:$LIBRARY_PATH

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/soft/compilers/openmpi/5.0.3/lib/

mkdir build && cd build
```

Create a file named cmake.sh with the following contents (note you may not need all the packages. Necessary packages for Symmetrix can be found on their [repo](https://github.com/wcwitt/symmetrix/tree/main/pair_symmetrix)):
```
cmake \
  -D MPI_CXX_COMPILER=$(which mpicxx) \
  -D CMAKE_CXX_COMPILER=$(pwd)/../lib/kokkos/bin/nvcc_wrapper \
  -D CMAKE_BUILD_TYPE=Release \
  -D CMAKE_INSTALL_PREFIX=$(pwd) \
  -D CMAKE_CXX_FLAGS="${CMAKE_CXX_FLAGS} -march=native -ffast-math" \
  \
  -D CUDA_TOOLKIT_ROOT_DIR=$CUDA_HOME \
  -D CUDA_NVCC_EXECUTABLE=$CUDA_HOME/bin/nvcc \
  -D CUDA_HOST_COMPILER=$(which gcc) \
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
  -D Kokkos_ARCH_AMDAVX=ON \
  -D Kokkos_ARCH_AMPERE100=ON \
  -D Kokkos_ENABLE_CUDA=ON \
  -D Kokkos_ENABLE_OPENMP=ON \
  -D Kokkos_ENABLE_SERIAL=ON \
  -D Kokkos_ENABLE_AGGRESSIVE_VECTORIZATION=ON \
  -D SYMMETRIX_KOKKOS=ON \
  -D SYMMETRIX_SPHERICART_CUDA=ON \
  \
  -D USE_MKL=OFF \
  -D USE_MKLDNN=OFF \
  -D MKL_INCLUDE_DIR="" \
  -D MKL_LIBRARY="" \
  ../cmake
```

Run script:
`sh cmake.sh`

Build:
`make -j 16`

# Scaling test

Scaling tests for Symmetrix was done using Mace-mpA0 (medium) foundation model.

## Strong scaling

![image](https://github.com/user-attachments/assets/25b03425-03a2-41a6-b17b-f813bdc8f95d)

![image](https://github.com/user-attachments/assets/56e0f508-b7fa-4fde-8d6e-27ae4ebf9962)

**Comparison with MACE-MLIAP** 

For MLIAP, tests were run with Mace-mp0a (small) model. With MACE-MLIAP, the same systems were also tested, however, the best performance achieved was  ~0.25 ns/day (with 672 atoms). The strong scaling trend was similar as with Symmetrix- increasing ranks quickly drgraded performance. MACE-MLIAP can also handle larger systems (up to 10k atoms tested), however, the performance is slower than with Symmetrix.

## Weak scaling

![image](https://github.com/user-attachments/assets/e5eac1e1-ed53-496d-8706-0b530d6fb661)


