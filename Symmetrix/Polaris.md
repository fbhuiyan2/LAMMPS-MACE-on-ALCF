```
# Get interactive compute node
qsub -I -l select=1 -l walltime=00:59:00 -q debug -l filesystems=home:eagle -A yourprojectname

# Load required modules
module use /soft/modulefiles/
module load cudatoolkit-standalone/12.4.0
module load conda
module load spack-pe-base cmake
conda activate base    # required only if PKG_PYTHON=ON

# cd 2 project dir

# create installation dir and download files
mkdir lammps-symmetrix && cd lammps-symmetrix
git clone -b release https://github.com/lammps/lammps
git clone --recursive https://github.com/wcwitt/symmetrix

# Symmetrix specific steps
cd symmetrix/pair_symmetrix
./install.sh ../../lammps
cd ../..

cd lammps

export CUDA_HOME=$(dirname $(dirname $(which nvcc)))
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH

# I have my own kokkos installed at kokkosinstall

export KOKKOS_PATH=$(pwd)/../../kokkosinstall
export PATH=$KOKKOS_PATH/bin:$PATH
export LD_LIBRARY_PATH=$KOKKOS_PATH/lib:$LD_LIBRARY_PATH
export LIBRARY_PATH=$KOKKOS_PATH/lib:$LIBRARY_PATH

mkdir build && cd build
```

Create a file named cmake.sh with the following contents (note you may not need all the packages. Necessary packages for Symmetrix can be found on their [repo](https://github.com/wcwitt/symmetrix/tree/main/pair_symmetrix)):

```
cmake \
  -D CMAKE_CXX_COMPILER=CC \
  -D CMAKE_BUILD_TYPE=Release \
  -D CMAKE_INSTALL_PREFIX=$(pwd) \
  -D CMAKE_CXX_FLAGS="-O3 -ffast-math" \
  \
  -D CMAKE_CXX_STANDARD=20 \
  -D CMAKE_CXX_STANDARD_REQUIRED=ON \
  -D CMAKE_CUDA_ARCHITECTURES=80 \
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
  -D Kokkos_ARCH_AMPERE80=ON \
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
