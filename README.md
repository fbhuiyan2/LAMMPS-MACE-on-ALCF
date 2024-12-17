# Installing LAMMPS with MACE on ALCF HPC Systems

This repo details the step-by-step process to install LAMMPS with MACE forcefield (and some other optional packages) on ALCF HPC systems.


### Running MACE finetuning using multigpu setting on ALCF systems

To use multigpu for training, you have to write the DistributedEnvironment class (mace/tools/slurm_distributed) for PBS scheduler. Then edit the relevant import in run_train.py (amce/cli/run_train.py, line 56, from mace.tools.slurm_distributed import DistributedEnvironment) 
