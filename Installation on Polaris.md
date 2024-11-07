# Polaris-Lammps-with-mace-installation

# Installing LAMMPS with MACE on Polaris

This guide provides step-by-step instructions to install [LAMMPS](https://lammps.org/) with the [MACE](https://github.com/ACEsuit/mace) package on Polaris, adapting the successful installation from Polaris.

---

## Table of Contents
- [Prerequisites](#prerequisites)
- [Step 1: Set Up the Environment](#step-1-set-up-the-environment)  
- [Step 2: Download Necessary Files on the Login Node](#step-2-download-necessary-files-on-the-login-node)
- [Step 3: Transfer Files to a Shared Filesystem](#step-3-transfer-files-to-a-shared-filesystem)
- [Step 4: Acquire a Compute Node for Compilation](#step-4-acquire-a-compute-node-for-compilation)
- [Step 5: Install the Latest CMake](#step-5-install-the-latest-cmake)
- [Step 6: Install Kokkos](#step-6-install-kokkos)
- [Step 7: Download and Prepare LibTorch](#step-7-download-and-prepare-libtorch)
- [Step 8: Clone LAMMPS with MACE Package](#step-8-clone-lammps-with-mace-package)
- [Step 9: Set Environment Variables](#step-9-set-environment-variables)
- [Step 10: Configure LAMMPS with CMake](#step-10-configure-lammps-with-cmake)
- [Step 11: Build LAMMPS](#step-11-build-lammps)
- [Step 12: Verify LAMMPS Installation](#step-12-verify-lammps-installation)
- [Step 13: Create a Job Submission Script](#step-13-create-a-job-submission-script)
- [Notes on `-l place=scatter` in Job Scripts](#notes-on--l-placescatter-in-job-scripts)
- [Troubleshooting](#troubleshooting)
- [References](#references)

---

## Prerequisites
- **Compute Node Access**: On Polaris, compilation must be performed on compute nodes, not login nodes.
- **Project Short Name**: Replace `yourProjectShortName` with your actual project short name in commands.
- **Internet Access**: Compute nodes on Polaris lack internet connectivity. All necessary files must be downloaded on the login nodes and transferred to the compute nodes via a shared filesystem.
- **Shared Filesystem**: Ensure you use directories that are accessible from both login and compute nodes (e.g., your home directory or a project directory).

---

## Step 1: Set Up the Environment

### Load Necessary Paths and Modules

```bash
module restore
module use /soft/modulefiles/
module add cudatoolkit-standalone/12.4.0
module add PrgEnv-gnu
module load gcc-native/12.3
module add  nvhpc-mixed

```

- You will also need to create a virtual python or conda environment. Instructions to do so can be found [here](https://docs.alcf.anl.gov/polaris/data-science-workflows/python/). After creating your venv, activate it, and then install MACE using:

```bash
pip install --upgrade pip
pip install mace-torch
```

- If you have previously created a venv with mace installed, then remember to activate it after loading the modules.

```bash
source /path/to/YourPythonVenv/bin/activate
```

---
## Step 2: Download Necessary Files on the Login Node

Since compute nodes do not have internet access, download all required files on the login node.

**On the login node (e.g., polaris-login-01 or polaris-login-02):**

- **Create a directory** for the installation files:  
```bash
mkdir ~/lammps_installation_files
cd ~/lammps_installation_files
```

- Note, I am using the home directory to show the installation process. However, this is not recommended since the quota for the home directory is very small. Instead, it is always better to use your project directory, such as, `/lus/eagle/projects/yourProjectShortName/yourusername` to download and install files. For simplicity for writing, I will use home directory here.

- **Download required packages:**

**CMake:**  
```bash
wget https://github.com/Kitware/CMake/releases/download/v3.27.6/cmake-3.27.6-linux-x86_64.tar.gz
```
**Kokkos:**  
```bash
wget https://github.com/kokkos/kokkos/releases/download/4.4.01/kokkos-4.4.01.tar.gz   # v 4.4.01 is, at the time of writing this, is the most recent version
```

**LibTorch:**  
You can see the current CUDA version on Polaris using this:

```bash    
nvcc --version
```

Note, there are or can be multiple CUDA version available on Polaris, I am using v12.4 here. 

You may have to get into a compute node to see an output. If that is the case, see "Step 4" which describes how to get a compute node in interactive mode. After getting into an interactive session, do `nvcc --version` to see the CUDA version.
  
  - **Download Matching LibTorch:**
    - Go to [Pytorch website](https://pytorch.org/), then select:
      - **Pytorch Build** = Stable
      - **Your OS** = Linux
      - **Package** = Libtorch
      - **Language** = C++/Java
      - **Computer Program** = CUDA 12.4 (or whatever is the matching version with Polaris)
    - Then copy the link for 'Download here (cxx11 ABI):' (let's call it `pytorch_link`)

```bash
wget pytorch_link  
```

**LAMMPS with MACE:**  
```bash
git clone --branch=mace --depth=1 https://github.com/ACEsuit/lammps
```

---

## Step 3: Transfer Files to a Shared Filesystem

Ensure the directory ~/lammps_installation_files is in a shared filesystem accessible from both login and compute nodes (e.g., your home directory or a project directory). Or if you want to transfer the files to a different directory in your project directory, for example, on eagle, then do so.


---

## Step 4: Acquire a Compute Node for Compilation

Since compilation cannot be performed on Polaris's login nodes, request an interactive session on a compute node:

```bash
qsub -I -l select=1 -l walltime=00:60:00 -q queuename -l filesystems=home:eagle -A yourProjectShortName
```

Replace `yourProjectShortName` with your actual project short name.

- **Explanation**:
  - `-I`: Interactive session.
  - `-q queuename`: Specifies the queue. On Polaris, you are limited to using the `debug` queue interactively.
  - `-l select=1`: Number of nodes.
  - `-t 0:60:00`: Time allocation (60 minutes).

Once you have access to the compute node, proceed with setting up your environment.

---

## Step 5: Install the Latest CMake

On the compute node, navigate to the directory with the installation files:

```bash
cd ~/lammps_installation_files
```

Extract and set up CMake:

```bash
tar -zxvf cmake-3.27.6-linux-x86_64.tar.gzexport
```

Add Cmake to path
```bash
PATH=$(pwd)/cmake-3.27.6-linux-x86_64/bin:$PATH
```

Check Cmake was installed properly
```bash
cmake --version
```

If you see the version information, then the installation was good.

---

## Step 6: Install Kokkos

Extract and build Kokkos:
```bash
cd ~/lammps_installation_files   # making sure you are on the right directiory
tar -zxvf kokkos-4.4.01.tar.gz
cd kokkos-4.4.01
mkdir build && cd build
CRAYPE_LINK_TYPE=dynamic \
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
make -j 16 && make install
```

---

## Step 7: Download and Prepare LibTorch

Extract LibTorch:
```bash
cd ~/lammps_installation_files   # making sure you are on the right directiory
unzip libtorch-shared-with-deps-2.5.1+cu124.zip  # whatever libtorch you downloaded
mv libtorch libtorch-gpu
```

---

## Step 8: Clone LAMMPS with MACE Package

```bash
cd ~/lammps_installation_files   # making sure you are on the right directiory
git clone --branch=mace --depth=1 https://github.com/ACEsuit/lammps
cd lammps
mkdir build && cd build
```

**Note**: If you have already cloned the git repo then you do not need to clone it again here, just cd into the directory and create the build folder.

---

## Step 9: Set Environment Variables

### 9.1 Set CUDA Paths
```bash
export CUDA_HOME="/soft/compilers/cudatoolkit/cuda-12.4.0"
export CUDA_ROOT=$CUDA_HOME
export CUDA_PATH=$CUDA_HOME
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH
```

### 9.2 Set Kokkos Paths
```bash
export KOKKOS_PATH=$(pwd)/../../kokkosinstall
export LD_LIBRARY_PATH=$(pwd)/../../kokkosinstall/lib:$LD_LIBRARY_PATH
export LIBRARY_PATH=$(pwd)/../../kokkosinstall/lib:$LIBRARY_PATH
export PATH=$(pwd)/../../kokkosinstall/bin:$PATH
```

### 9.3 Set LibTorch Path
```bash
export PATHTORCH=$(pwd)/../../libtorch-gpu
```

### 9.5 Craype
```bash
CRAYPE_LINK_TYPE=dynamic
```

---

## Step 10: Configure LAMMPS with CMake
```bash
cmake \
    -D CMAKE_CXX_COMPILER=CC \
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
    -D PKG_KOKKOS=ON \
    -D Kokkos_ARCH_AMDAVX=ON \
    -D Kokkos_ARCH_AMPERE80=ON \
    -D Kokkos_ENABLE_CUDA=ON \
    -D Kokkos_ENABLE_OPENMP=ON \
    -D Kokkos_ENABLE_SERIAL=ON \
    -D Kokkos_CXX_STANDARD=17 \
    -D CMAKE_PREFIX_PATH=${PATHTORCH} \
    -D CUDA_TOOLKIT_ROOT_DIR=$CUDA_HOME \
    -D CUDA_NVCC_EXECUTABLE=$CUDA_HOME/bin/nvcc \
    -D CUDA_HOST_COMPILER=$(which gcc) \
    -D BUILD_MPI=ON \
    -D BUILD_SHARED_LIBS=ON \
    -D CMAKE_BUILD_TYPE=Release \
    -D CMAKE_CXX_STANDARD=17 \
    -D CMAKE_CXX_STANDARD_REQUIRED=ON \
    -D USE_MKL=OFF \
    -D USE_MKLDNN=OFF \
    -D MKL_INCLUDE_DIR="" \
    -D MKL_LIBRARY="" \
    ../cmake
```

**Note:**

- If you encounter errors, clean the build directory and retry:  
```bash
rm -rf *
```

- If you are repeating the cmake step in the `/lammps/build` directory for any reason, then first delete the previous build directory and then make the build directory again before going into the build directory.

---

## Step 11: Build LAMMPS

```bash
make -j 16
```

---

## Step 12: Verify LAMMPS Installation

### 12.1 Locate the LAMMPS Executable

```bash
realpath lmp
```

### 12.2 Test the LAMMPS Build

Navigate to an example directory and run a test:

```bash
cd ../examples/friction
../../build/lmp -in in.friction -k on g 1 -sf kk
```
This should start running the simulation in lammps. If you see no proper output or see any error, then there is something wrong with the installation.

---

## Step 13: Create a Job Submission Script

Below is a sample job submission script adapted for Polaris.

**File: run_lammps_polaris.sh**
```bash
#!/bin/bash

NNODES=`wc -l < $PBS_NODEFILE`
echo NNODES = $NNODES
NRANKS=4
#NRANKSSOCKET=2
NDEPTH=1
NTHREADS=8
NGPUS=4

NTOTRANKS=$(( NNODES * NRANKS ))


module restore
module use /soft/modulefiles/
module add cudatoolkit-standalone/12.4.0
module add PrgEnv-gnu
module load gcc-native/12.3
module add  nvhpc-mixed
source path/to/YOUR-PYTHON-VENV/bin/activate


export CUDA_HOME="/soft/compilers/cudatoolkit/cuda-12.4.0"
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$CUDA_HOME/lib64:/path/to/lammps/build


EXE=/path/to/lammps/build/lmp

EXE_ARG="-in in_here.lammps  -k on g ${NGPUS} -sf kk  "

MPI_ARG="-n ${NTOTRANKS} --ppn ${NRANKS} --depth=${NDEPTH} --cpu-bind depth --env OMP_NUM_THREADS=${NTHREADS} --env OMP_PROC_BIND=spread --env OMP_PLACES=cores"


COMMAND="mpiexec ${MPI_ARG} ${EXE} ${EXE_ARG}"
echo "COMMAND= ${COMMAND}"
${COMMAND}
```


**Notes:**

- Replace /path/to/lammps/build/lmp with the actual path to your LAMMPS executable.
- Ensure the script is executable:  
```bash
chmod +x run_lammps_polaris.sh
```

**PBS Job Script: submit_lammps.pbs**

An example submission script for running a 64-node KOKKOS-enabled LAMMPS executable is below as an example. Additional information on LAMMPS application flags and options is described on the LAMMPS website.

```bash
#!/bin/bash
#PBS -l select=64:system=polaris
#PBS -l place=scatter
#PBS -l walltime=0:15:00
#PBS -l filesystems=home:eagle
#PBS -q prod
#PBS -A YourProjectName

# Load necessary modules or environment variables if needed
. /etc/profile

# Navigate to the directory containing the script and input files
cd $PBS_O_WORKDIR

# Execute the script
./run_lammps_polaris.sh
```

- Submit the job:  
```bash
qsub submit_lammps.pbs
```

---

## Notes on -l place=scatter in Job Scripts

The -l place=scatter option in a PBS job script specifies how to allocate resources across nodes.

- **place=scatter**:
  - **Purpose**: Distributes your job's processes across multiple nodes to minimize resource contention.
  - **Benefit**: Reduces competition for resources like CPUs, memory bandwidth, and network interfaces, potentially improving performance.

- **Usage in Job Script**:  
```bash
#PBS -l place=scatter
```
- **Considerations**:
  - Use scatter when your job can benefit from being spread across multiple nodes.
  - Use pack if you want to pack processes onto as few nodes as possible.

---

## Troubleshooting

### Compute Nodes Lack Internet Access

- **Compute nodes on Polaris do not have network connectivity**.
- **Solution**: Download all necessary files on the login node and transfer them via a shared filesystem.

### Bash Profile Adjustment

This may not be necessary, but if you are having problems with anything that you cannot solve then do this. For your interactive session, it’s important to source the global profile so that all necessary environment variables and paths are correctly set. Add the following to your job script or profile:

```bash
. /etc/profile
```

---

## References
- **Polaris User Guides**: [Compiling and Linking on Polaris](https://docs.alcf.anl.gov/polaris/getting-started/)
- **LAMMPS Documentation**: [LAMMPS Installation Guide](https://docs.lammps.org/Install.html)
- **Kokkos Documentation**: [Kokkos GitHub Repository](https://github.com/kokkos/kokkos)
- **LibTorch Documentation**: [PyTorch C++ Documentation](https://pytorch.org/cppdocs/)
- **PBS Scheduler Documentation**: [PBS Professional User's Guide](https://www.altair.com/pdfs/pbsworks/PBSUserGuide2021.pdf)
- **Open MPI Documentation**: [Process Management Interface](https://www.open-mpi.org/doc/current/man1/mpirun.1.php)
