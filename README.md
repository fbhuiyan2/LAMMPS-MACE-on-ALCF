# Sophia-Lammps-with-mace-installation

# Installing LAMMPS with MACE on Sophia

This guide provides step-by-step instructions to install [LAMMPS](https://lammps.org/) with the [MACE](https://github.com/ACEsuit/mace) package on Sophia, adapting the successful installation from Polaris. It addresses differences in module availability, compiler settings, network limitations, and environment configurations between Polaris and Sophia.

---

## Table of Contents
- [Prerequisites](#prerequisites)
- [Step 1: Acquire a Compute Node for Compilation](#step-1-acquire-a-compute-node-for-compilation)
- [Step 2: Set Up the Environment](#step-2-set-up-the-environment)  
  - [2.1 Bash Profile Adjustment](#21-bash-profile-adjustment)  
  - [2.2 Load Necessary Paths and Modules](#22-load-necessary-paths-and-modules)
- [Step 3: Download Necessary Files on the Login Node](#step-3-download-necessary-files-on-the-login-node)
- [Step 4: Transfer Files to a Shared Filesystem](#step-4-transfer-files-to-a-shared-filesystem)
- [Step 5: Install the Latest CMake](#step-5-install-the-latest-cmake)
- [Step 6: Download and Install Kokkos](#step-6-download-and-install-kokkos)
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
- **Compute Node Access**: On Sophia, compilation must be performed on compute nodes, not login nodes.
- **Project Short Name**: Replace `yourProjectShortName` with your actual project short name in commands.
- **Internet Access**: Compute nodes on Sophia lack internet connectivity. All necessary files must be downloaded on the login nodes and transferred to the compute nodes via a shared filesystem.
- **Shared Filesystem**: Ensure you use directories that are accessible from both login and compute nodes (e.g., your home directory or a project directory).

---

## Step 2: Set Up the Environment

### 2.1 Bash Profile Adjustment

For your interactive session, it’s important to source the global profile so that all necessary environment variables and paths are correctly set. Add the following to your job script or profile:

```bash
. /etc/profile
```

### 2.2 Load Necessary Paths and Modules

On Sophia, compilers and CUDA tools are typically installed system-wide on compute nodes.
#### Verify Compilers and Tools
```bash
# Check GCC version
gcc --version
# Check G++ version
g++ --version
# Check Fortran compiler
gfortran --version
# Check CUDA compiler
nvcc --version
```

#### Verify MPI Installation

Sophia's MPI is located at /usr/mpi/gcc/openmpi-4.1.5a1. Add MPI to your PATH and LD_LIBRARY_PATH:
```bash
export PATH=/usr/mpi/gcc/openmpi-4.1.5a1/bin:$PATH
export LD_LIBRARY_PATH=/usr/mpi/gcc/openmpi-4.1.5a1/lib:$LD_LIBRARY_PATH
```
#### Verify MPI Compilers
```bash
which mpicc
which mpicxx
which mpifort
```
---
## Step 3: Download Necessary Files on the Login Node

Since compute nodes do not have internet access, download all required files on the login node.

**On the login node (e.g., sophia-login-01 or sophia-login-02):**

- **Create a directory** for the installation files:  
```bash
mkdir ~/lammps_installation_files  cd ~/lammps_installation_files
```

- **Download required packages:**

**CMake:**  
```bash
wget https://github.com/Kitware/CMake/releases/download/v3.27.6/cmake-3.27.6-linux-x86_64.tar.gz
```
**Kokkos:**  
```bash
wget https://github.com/kokkos/kokkos/archive/refs/tags/4.1.00.zip -O kokkos-4.1.00.zip
```
**LibTorch:**  
  - **Determine the CUDA version on Sophia's compute nodes.**
    Request an interactive session:    
```bash
qsub -I -q workq -A yourProjectShortName -n 1 -t 00:30:00
```
    
On the compute node:    
```bash    
nvcc --version
```
  Note the CUDA version (e.g., CUDA 12.4).  
  - **Download Matching LibTorch:**    
```bash
wget https://download.pytorch.org/libtorch/cu124/libtorch-shared-with-deps-2.5.1%2Bcu124.zip
```
  **LAMMPS with MACE:**  
```bash
git clone --branch=mace --depth=1 https://github.com/ACEsuit/lammps
```

---

## Step 1: Acquire a Compute Node for Compilation
Since compilation cannot be performed on Sophia's login nodes, request an interactive session on a compute node:

```bash
qsub -I -l select=1 -l walltime=00:59:00 -q queuename -l filesystems=home:eagle -A yourProjectShortName
qsub -I -q queuename -A yourProjectShortName -n 1 -t 02:00:00
```
Replace `yourProjectShortName` with your actual project short name.

- **Explanation**:
  - `-I`: Interactive session.
  - `-q queuename`: Specifies the queue. Change `queuename` with a proper queue in Sophia.
  - `-l select=1`: Number of gpus, for by-gpu and bigmem queue.
  - `-t 02:00:00`: Time allocation (2 hours).

Once you have access to the compute node, proceed with setting up your environment.

---

## Step 4: Transfer Files to a Shared Filesystem

Ensure the directory ~/lammps_installation_files is in a shared filesystem accessible from both login and compute nodes (e.g., your home directory or a project directory).

---

## Step 5: Install the Latest CMake

On the compute node, navigate to the directory with the installation files:
```bash
cd ~/lammps_installation_files
```
Extract and set up CMake:
```bash
tar -zxvf cmake-3.27.6-linux-x86_64.tar.gzexport PATH=$(pwd)/cmake-3.27.6-linux-x86_64/bin:$PATHcmake --version
```
---

## Step 6: Download and Install Kokkos

Extract and build Kokkos:
```bash
unzip kokkos-4.1.00.zipcd kokkos-4.1.00
mkdir build && cd build
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
cd ~/lammps_installation_filesunzip libtorch-shared-with-deps-2.5.1+cu124.zipmv libtorch libtorch-gpu
```

---

## Step 8: Clone LAMMPS with MACE Package

```bash
cd ~/lammps_installation_files
git clone --branch=mace --depth=1 https://github.com/ACEsuit/lammps
cd lammps
mkdir build && cd build
```

---

## Step 9: Set Environment Variables

### 9.1 Set CUDA Paths
```bash
export CUDA_HOME=$(dirname $(dirname $(which nvcc)))
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH
```

### 9.2 Set Kokkos Paths
```bash
export KOKKOS_PATH=$(pwd)/../../kokkosinstall
export PATH=$KOKKOS_PATH/bin:$PATH
export LD_LIBRARY_PATH=$KOKKOS_PATH/lib:$LD_LIBRARY_PATH
export LIBRARY_PATH=$KOKKOS_PATH/lib:$LIBRARY_PATH
```

### 9.3 Set LibTorch Path
```bash
export PATHTORCH=$(pwd)/../../libtorch-gpu
```

### 9.4 Ensure Correct CMake is Used
```bash
export PATH=$(pwd)/../../cmake-3.27.6-linux-x86_64/bin:$PATH
```

---

## Step 10: Configure LAMMPS with CMake
```bash
cmake \
  -D CMAKE_CXX_COMPILER=$(which mpicxx) \
  -D PKG_ML-MACE=ON \
  -D PKG_KOKKOS=ON \
  -D Kokkos_ARCH_ZEN2=ON \
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
  -D PKG_MEAM=ON \
  -D PKG_REAXFF=ON \
  -D PKG_ELECTRODE=ON \
  -D PKG_PYTHON=ON \
  -D PKG_QEQ=ON \
  -D PKG_REPLICA=ON \
  -D PKG_MANYBODY=ON \
  -D PKG_MISC=ON \
  -D PKG_RIGID=ON \
  ../cmake
```

**Note:**

- Ensure that mpicxx is available in your PATH. If not, add it:  
```bash
export PATH=/usr/mpi/gcc/openmpi-4.1.5a1/bin:$PATH
export LD_LIBRARY_PATH=/usr/mpi/gcc/openmpi-4.1.5a1/lib:$LD_LIBRARY_PATH
```

- If you encounter errors, clean the build directory and retry:  
```bash
rm -rf *
```

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
cd ../examples/flow
mpirun -np 4 ../../build/lmp -in in.flow -k on g 1 -sf kk
```

---

## Step 13: Create a Job Submission Script

Below is a sample job submission script adapted for Sophia.

**File: run_lammps_sophia.sh**
```bash
#!/bin/bash

# Get the number of nodes allocated by PBS
NNODES=$(wc -l < $PBS_NODEFILE)
echo "NNODES = $NNODES"

# Configuration parameters
NRANKS=1        # Number of MPI ranks per node
NTHREADS=8      # Number of OpenMP threads per MPI rank
NGPUS=1         # Number of GPUs per node

# Total number of MPI ranks
NTOTRANKS=$(( NNODES * NRANKS ))

# Set CUDA environment variables
export CUDA_HOME=$(dirname $(dirname $(which nvcc)))
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$CUDA_HOME/lib64

# Add LAMMPS library to LD_LIBRARY_PATH if needed
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/path/to/lammps/build

# Set OpenMP environment variables
export OMP_NUM_THREADS=$NTHREADS
export OMP_PROC_BIND=spread
export OMP_PLACES=cores

# Path to the LAMMPS executable
EXE=/path/to/lammps/build/lmp

# LAMMPS input arguments
EXE_ARG="-in in_here.lammps -k on g $NGPUS -sf kk"

# MPI execution command
MPI_ARG="-np $NTOTRANKS"

# Additional Open MPI options for process binding and mapping
MPI_OPTIONS="--bind-to core --map-by slot:PE=$NTHREADS"

# Include OpenMP environment variables in MPI execution
MPI_ENV_VARS="-x OMP_NUM_THREADS -x OMP_PROC_BIND -x OMP_PLACES"

# Construct the final command
COMMAND="mpirun $MPI_ARG $MPI_OPTIONS $MPI_ENV_VARS $EXE $EXE_ARG"

# Display and execute the command
echo "COMMAND= $COMMAND"
$COMMAND
```


**Notes:**

- Replace /path/to/lammps/build/lmp with the actual path to your LAMMPS executable.
- Ensure the script is executable:  
```bash
chmod +x run_lammps_sophia.sh
```

**PBS Job Script: submit_lammps.pbs**
```bash
#!/bin/bash
#PBS -N lammps_job
#PBS -l select=1:ncpus=8:mpiprocs=1:ngpus=1
#PBS -l walltime=01:00:00
#PBS -q workq
#PBS -A yourProjectShortName
#PBS -j oe

# Load necessary modules or environment variables if needed
. /etc/profile

# Navigate to the directory containing the script and input files
cd $PBS_O_WORKDIR

# Execute the script
./run_lammps_sophia.sh
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

### CMake Cannot Find C++ Compiler

If you encounter the error:
```plaintext
No CMAKE_CXX_COMPILER could be found.
```
- **Ensure MPI Compilers Are in PATH**:  
```bash
export PATH=/usr/mpi/gcc/openmpi-4.1.5a1/bin:$PATH
export LD_LIBRARY_PATH=/usr/mpi/gcc/openmpi-4.1.5a1/lib:$LD_LIBRARY_PATH
```
- **Verify mpicxx Is Available**:  
```bash
which mpicxx
mpicxx --version
```
- **Clean Previous CMake Configurations**:  
```bash
rm -rf CMakeCache.txt CMakeFiles
```
- **Re-run CMake Configuration**:  
```bash
cmake -D CMAKE_CXX_COMPILER=$(which mpicxx) ... ..
```

### Compute Nodes Lack Internet Access

- **Compute nodes on Sophia do not have network connectivity**.
- **Solution**: Download all necessary files on the login node and transfer them via a shared filesystem.

---

## References
- **Sophia User Guides**: [Compiling and Linking on Sophia](https://docs.alcf.anl.gov/sophia/compiling-and-linking/compiling-and-linking-overview/)
- **LAMMPS Documentation**: [LAMMPS Installation Guide](https://docs.lammps.org/Install.html)
- **Kokkos Documentation**: [Kokkos GitHub Repository](https://github.com/kokkos/kokkos)
- **LibTorch Documentation**: [PyTorch C++ Documentation](https://pytorch.org/cppdocs/)
- **PBS Scheduler Documentation**: [PBS Professional User's Guide](https://www.altair.com/pdfs/pbsworks/PBSUserGuide2021.pdf)
- **Open MPI Documentation**: [Process Management Interface](https://www.open-mpi.org/doc/current/man1/mpirun.1.php)
