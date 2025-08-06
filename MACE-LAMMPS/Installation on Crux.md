# Loading modules

```
module load cray-python/3.11.7
module load PrgEnv-gnu/8.5.0
# I do not have to load in gcc since loading PrgEnv-gnu auto loads a gcc 13.2 on crux
export CC=cc
export CXX=CC
```
# Download Pytorch for CPU

```
wget https://download.pytorch.org/libtorch/cpu/libtorch-shared-with-deps-2.6.0%2Bcpu.zip
unzip libtorch-shared-with-deps-2.6.0+cpu.zip
rm libtorch-shared-with-deps-2.6.0+cpu.zip
```

# Clone the ACEsuit Lammps repo

```
git clone --branch mace --depth=1 https://github.com/ACEsuit/lammps
```

# Download Cmake

Crux does not have a module for cmake when writing this. So, have to install your own cmake.

```
wget https://github.com/Kitware/CMake/releases/download/v3.31.5/cmake-3.31.5-linux-x86_64.tar.gz
tar -zxvf cmake-3.31.5-linux-x86_64.tar.gz
rm cmake-3.31.5-linux-x86_64.tar.gz
# add cmake to path
PATH=$(pwd)/cmake-3.31.5-linux-x86_64/bin:$PATH

# check installation
which cmake
```

# Building Lammps

```
cd lammps; mkdir build; cd build

cmake \
-D CMAKE_INSTALL_PREFIX=$(pwd) \
-D CMAKE_C_COMPILER=cc \
-D CMAKE_CXX_COMPILER=CC \
-D CMAKE_CXX_STANDARD=17 \
-D CMAKE_CXX_STANDARD_REQUIRED=ON \
-D CMAKE_PREFIX_PATH=$(pwd)/../../libtorch \
-D PKG_KOKKOS=ON \
-D Kokkos_ARCH_ZEN3=ON \
-D Kokkos_ENABLE_OPENMP=ON \
-D Kokkos_ENABLE_SERIAL=ON \
-D BUILD_MPI=ON \
-D BUILD_OMP=ON \
-D PKG_OPENMP=ON \
-D PKG_ML-MACE=ON \
-D PKG_MEAM=ON \
-D PKG_REAXFF=ON \
-D PKG_ELECTRODE=ON \
-D PKG_PYTHON=ON \
-D PKG_QEQ=ON \
-D PKG_REPLICA=ON \
-D PKG_MANYBODY=ON \
-D PKG_MISC=ON \
-D PKG_RIGID=ON \
-D USE_MKL=OFF \
-D USE_MKLDNN=OFF \
-D MKL_INCLUDE_DIR="" \
-D MKL_LIBRARY="" \
../cmake

make -j 4
make install
```


