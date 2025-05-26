# Installation Instructions for LAMMPS with MACE-MLIAP on Sophia

**Last updated: May 2025**

This guide provides step-by-step instructions to install LAMMPS with MACE-MLIAP implementation on Sophia. **Note** this is mostly similar to Polaris except a few changes in the loaded modules and the cmake cxx compiler.

## Get Interactive Node

Request an interactive node with the following command:

```bash
qsub -I -l select=1 -l walltime=02:00:00 -q by-gpu -l filesystems=home:eagle -A yourprojectname
```

## Load Necessary Modules

Load the required modules by executing:

```bash
module load conda
module load compilers/openmpi/5.0.3
```

These are all the modules that needs to be loaded on Sophia.

#### Create main installation dir in your project dir

`mkdir lammps_mace && cd lammps_mace`

## Create a New Conda Environment

Create a new Python conda environment named `mace-lammps-mliap`:

```bash
conda create --prefix ./mace-lammps-mliap python=3.11.7
```

**Note:** Python version 3.11 is necessary since the cuequivariance (0.4.0) module requires Python >= 3.10. Use Python 3.11.7 specifically to avoid known issues with version 3.11.8.

#### Activate Conda Environment

Activate the newly created conda environment:

```bash
conda activate ./mace-lammps-mliap
```

#### Install Necessary Libraries

Install the required libraries using pip:

```bash
pip install "torch==2.5.0" "torchvision==0.20.0" "torchaudio==2.5.0" cupy-cuda12x mace-torch ase cuequivariance-torch cython cuequivariance-ops-torch-cu12
```

Currently, MACE does not work with torch>2.5.

## Download and Install Kokkos

Installing Kokkos is not mandatory. However, when built with the kokkos available in lammps, I encountered errors related to Kokkos not being built with GPU support. So, I recommend this step. 

If you want to skip this, then you just need to exclude the environment path setup related to kokkos in the **Building LAMMPS** step. Everything else should remain the same.

1. Download Kokkos:
   
```bash    
wget https://github.com/kokkos/kokkos/releases/download/4.4.01/kokkos-4.4.01.tar.gz
```

kokkos-4.4.01 is tested and works. If newer version is available, you can try that.

2. Install Kokkos:

```bash
cd lammps_mace  # make sure in the main installation dir
tar -zxvf kokkos-4.4.01.tar.gz    
cd kokkos-4.4.01
mkdir build && cd build    
```

Create a file for the cmake commands `vi cmake.sh`:

```
cmake \
  -DCMAKE_CXX_COMPILER=$(which g++) \
  -DKokkos_ENABLE_CUDA=ON \
  -DKokkos_ENABLE_OPENMP=ON \
  -DKokkos_ENABLE_SERIAL=ON \
  -DKokkos_ARCH_ZEN2=ON \
  -DKokkos_ARCH_AMPERE80=ON \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=$(pwd)/../../kokkosinstall \
  ..
```

run the file with `sh cmake.sh`.

After the build is completed successfully, install using:

`make -j 16 && make install`

## Download Libtorch

Download the Libtorch library as required for your setup.

Check CUDA version (must be inside an **interactive node**):

```bash    
nvcc --version
```
  
  - **Download Matching LibTorch:**
    - Go to [Pytorch website](https://pytorch.org/), then select:
      - **Pytorch Build** = Stable, **OS** = Linux, **Package** = Libtorch
      - **Language** = C++/Java, **Computer Program** = CUDA 12.4 (current cudatoolkit version on Sophia is 12.4.0)
    - Then copy the link for 'Download here (cxx11 ABI):' (let's call it `libtorch_link`)

Currently, the available pytorch version on the website is 2.7 and CUDA 12.6. Since we are using torch==2.5 and cudatoolkit==12.4, you can use this [link](https://download.pytorch.org/libtorch/cu124) instead to download matching libtorch. Here is the `libtorch_link` I used:

`libtorch_link=https://download.pytorch.org/libtorch/cu124/libtorch-cxx11-abi-shared-with-deps-2.5.0%2Bcu124.zip`

```bash
cd lammps_mace  # make sure in the main installation dir
wget libtorch_link
unzip libtorch-shared-with-deps-2.5.0+cu124.zip  # whatever libtorch you downloaded
```


## Building LAMMPS

Clone the LAMMPS repository from GitHub:

```bash
cd lammps_mace  # make sure in the main installation dir
git clone https://github.com/lammps/lammps.git
cd lammps
mkdir build && cd build
```

**Note**: Current Lammps version in the develop branch is `2 Apr 2025` (can be checked in lammps/src/version.h). MACE-MLIAP appears to be sensitive to the specific Lammps version and works with this version. There is no gurantee that MACE-MLIAP will work with older or newer Lammps version.

#### Set up environment

Create a file for setting up environment variables using `vi env.sh`

```
#!/bin/bash

# Note, you will have to change PATHTORCH and KOKKOS_PATH if they are different. Currently, they are based on /lammps/build dir, where this script shoould be executed from


export CUDA_HOME=$(dirname $(dirname $(which nvcc)))
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH

export PATHTORCH=$(pwd)/../../libtorch


export KOKKOS_PATH=$(pwd)/../../kokkosinstall
export PATH=$KOKKOS_PATH/bin:$PATH
export LD_LIBRARY_PATH=$KOKKOS_PATH/lib:$LD_LIBRARY_PATH
export LIBRARY_PATH=$KOKKOS_PATH/lib:$LIBRARY_PATH

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/soft/compilers/openmpi/5.0.3/lib/

echo "Environment setup complete:"
echo "  CUDA_HOME = $CUDA_HOME"
echo "  KOKKOS_PATH = $KOKKOS_PATH"
echo "  PATHTORCH = $PATHTORCH"
```

Make the file executable using `chmod +x env.sh`

Source the file using `source env.sh` to setup environment.

#### Cmake command

Create a file for the cmake commands `vi cmake.sh`:

```
cmake \
  -D CMAKE_CXX_COMPILER=$(which mpicxx) \
  -D CMAKE_BUILD_TYPE=Release \
  -D CMAKE_INSTALL_PREFIX=$(pwd) \
  \
  -D BUILD_MPI=ON \
  -D BUILD_SHARED_LIBS=ON \
  -D PKG_ML-IAP=ON \
  -D PKG_ML-SNAP=ON \
  -D PKG_PYTHON=ON \
  -D MLIAP_ENABLE_PYTHON=ON \
  -D PKG_MEAM=ON \
  -D PKG_REAXFF=ON \
  -D PKG_ELECTRODE=ON \
  -D PKG_QEQ=ON \
  -D PKG_REPLICA=ON \
  -D PKG_MANYBODY=ON \
  -D PKG_MISC=ON \
  -D PKG_RIGID=ON \
  -D PKG_KOKKOS=ON \
  \
  -D CMAKE_PREFIX_PATH=${PATHTORCH} \
  -D CUDA_TOOLKIT_ROOT_DIR=$CUDA_HOME \
  -D CUDA_NVCC_EXECUTABLE=$CUDA_HOME/bin/nvcc \
  -D CUDA_HOST_COMPILER=$(which gcc) \
  -D CMAKE_CXX_STANDARD=17 \
  -D CMAKE_CXX_STANDARD_REQUIRED=ON \
  \
  -D Kokkos_ARCH_ZEN2=ON \
  -D Kokkos_ARCH_AMPERE80=ON \
  -D Kokkos_ENABLE_CUDA=ON \
  -D Kokkos_ENABLE_OPENMP=ON \
  -D Kokkos_ENABLE_SERIAL=ON \
  \
  -D USE_MKL=OFF \
  -D USE_MKLDNN=OFF \
  -D MKL_INCLUDE_DIR="" \
  -D MKL_LIBRARY="" \
  ../cmake
```

Run the script:

`sh cmake.sh`

**Note** The current `2 Apr 2025` version of Lammps in the develop branch of the repo has a potential bug in the `lammps/cmake/CMakeLists.txt` file. If your cmake script fails to build lammps complaining about not being able to find mpi or mpicxx, or an error related to mpi, then scroll down to the **Troubleshoot** section.

If no error, then continue.

After the build is completed successfully, install lammps:

`make -j 16`

Then install lammps Python package:

`make install-python`

#### Test run

To test if Lammps with MACE-MLIAP works properly, you can use the following command:

```
export MPICH_GPU_SUPPORT_ENABLED=0
exe=path/to/lammps/build/lmp
mpirun -np 1 $exe -k on g 1 -sf kk -pk kokkos newton on neigh half -in in_mliap.lammps
```

**Note**, you will need an mliap-compatible MACE model (`python mace/cli/create_lammps_model.py your_trained_model.model --format=mliap`) and the necessary input script (in_mliap.lammps).

## Troubleshoot

- **MPI related error during Lammps cmake build**

`2 Apr 2025` version of Lammps in the develop branch of the repo has a potential bug in the `lammps/cmake/CMakeLists.txt` file which can lead to this error. In this case, download a previous or newer lammps stable version from the main [website](https://www.lammps.org/download.html).

Currently, the most stable version is `29 Aug 2024` and the link is: `https://download.lammps.org/tars/lammps-stable.tar.gz`

Unpack using `tar -xzvf file.tar.gz `

copy the CMakeLists.txt from this lammps version to the lammps/cmake dir of the to-be-installed lammps version. This will replace the faulty (most probably) CMakeLists.txt file.

Now the cmake.sh script should work properly.
