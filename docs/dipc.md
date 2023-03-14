# USE GROMACS + PLUMED IN @DIPC

## DIPC SYSTEMS
For any information related with how DIPC computing system works (ATLAS), please see the corresponding web pages:
* [ATLAS](https://cc.dipc.org/computing_resources/)
* [ATLAS EDR System](https://cc.dipc.org/computing_resources/systems/atlas-edr/)
* [ATLAS FDR System](https://cc.dipc.org/computing_resources/systems/atlas-fdr/)

The main difference between the 2 systems is that EDR has GPUs while FDR do not. For this reason, we will use for our purpose the **EDR system**.

* * * 
## STARTING
The first thing you need to do in Atlas for using any software is to load it. Use the command below to load the GROMACS software with the PLUMED plugin installed.

To date: 13 / 03 / 2023
```
module load GROMACS/2021.3-fosscuda-2020b-PLUMED-2.7.2
```

The very basic Batch script for parallel execution of GROMACS is the following:
```
#!/bin/bash
#SBATCH --partition=regular
#SBATCH --job-name=GROMACS_job
#SBATCH --mem=200gb
#SBATCH --cpus-per-task=6
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=8
#SBATCH --output=%x-%j.out
#SBATCH --error=%x-%j.err

module load GROMACS/2021.3-fosscuda-2020b-PLUMED-2.7.2

srun  gmx_mpi mdrun -ntomp ${SLURM_CPUS_PER_TASK} -s input.tpr
```

For the use of GPUs you can use:
```
#!/bin/bash
#SBATCH --partition=regular
#SBATCH --job-name=GROMACS_job
#SBATCH --mem=200gb
#SBATCH --cpus-per-task=1
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=48
#SBATCH --gres=gpu:p40:2
#SBATCH --output=%x-%j.out
#SBATCH --error=%x-%j.err

module load GROMACS/2021.3-fosscuda-2020b-PLUMED-2.7.2

srun gmx_mpi mdrun -ntomp $SLURM_CPUS_PER_TASK -nb auto -bonded auto -pme auto -gpu_id 01 -s input.tpr
```
