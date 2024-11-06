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

## Step 1: Acquire a Compute Node for Compilation
Since compilation cannot be performed on Sophia's login nodes, request an interactive session on a compute node:

```bash
qsub -I -q workq -A yourProjectShortName -n 1 -t 02:00:00
```
Replace `yourProjectShortName` with your actual project short name.

- **Explanation**:
  - `-I`: Interactive session.
  - `-q workq`: Specifies the queue.
  - `-n 1`: Number of nodes.
  - `-t 02:00:00`: Time allocation (2 hours).

Once you have access to the compute node, proceed with setting up your environment.

---

## Step 2: Set Up the Environment

### 2.1 Bash Profile Adjustment

For your interactive session, it’s important to source the global profile so that all necessary environment variables and paths are correctly set. Add the following to your job script or profile:

```bash
. /etc/profile
```

