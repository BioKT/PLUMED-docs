
<script
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"
  type="text/javascript">
</script>

## Working with multiple replicas
For a general description of how to integrate PLUMED when working with multiple
replicas, check the PLUMED [masterclass 21.5](https://github.com/plumed/masterclass-21-5).
Note that for using this you will need both PLUMED and Gromacs to be installed 
using MPI. Check installation instructions 
[here](https://www.plumed.org/doc-v2.8/user-doc/html/_installation.html).

### REST2
Here we illustrate how to use the replica exchage with solute tempering method in its 
second iteration (i.e. [REST2](https://doi.org/10.1021/jp204407d)), which uses a different
 scaling scheme that the original [REST](https://doi.org/10.1073/pnas.0506346102).
Before you start, please check the materials at the PLUMED website on [Hamiltonian
replica exchange](https://www.plumed.org/doc-v2.7/user-doc/html/hrex.html).

The first thing to check is whether all the files you will need are available in your folder. 
In the terminal, type

    $ ls data/replex

You should find a number of files, including the Gromacs structure file `alaTB_ff03_tip3p_npt.gro`
and another file called `alaTB_ff03_tip3p_pp.top`. The former is a properly equilibrated 
structure file, of the alanine dipeptide in TIP3P water. The latter is a pre-processed topology
 containing all the information required to run a simulation, and
hence devoid of [`include` directives](https://manual.gromacs.org/current/dev-manual/includestyle.html).
Note that in the `[ atoms ]` section, we have modified the standard names of the atoms with
an underscore (i.e. `HC` is now `HC_`). This will mark these atoms as the solute whose
interactions will be scaled in the REST2 scheme.

The first step is hence to scale the interactions, and in order to do this, we
use PLUMED `partial_tempering' program. Because we are running multiple 
replicas, each with a different scaling, the command will be run inside a loop
in the following Python script:

    import sys, os

    gro = "alaTB_ff03_tip3p_npt.gro"
    toppp = "alaTB_ff03_tip3p_pp.top"
    mdp = "sd_nvt.mdp"
    
    # scale solute interactions
    nrep = 4
    tmax = 1000
    t0 = 300
    for i in range(nrep):
        exponent = i/(nrep - 1)
        ti = t0*(tmax/t0)**exponent 
        lmbd = t0/ti
    
        folder = "rep%i"%i
        try:
            os.mkdir(folder)
        except OSError as e:
            print (e)
            
        command = "plumed partial_tempering %g < %s > %s/scaled.top"%(lmbd, toppp, folder)
        os.system(command)
    
        top = "%s/scaled.top"%folder 
        tpr = "%s/alaTB_ff03_tip3p_nvt.tpr"%folder
        command = gmx + " grompp -f %s -p %s -c %s -o %s"%(mdp, top, gro, tpr)
        os.system(command)

In your machine, `gmx` should point to a working MPI installation of Gromacs.
This script will generate a number of folders, each with a scaled topology file `scaled.top`. 
Because we have generated the run input files for Gromacs, we can simply run the
simulations using

    mpiexec -np 4  gmx_mpi mdrun -s alaTB_ff03_tip3p_nvt -multidir rep*  -nsteps 1000000 -plumed ../plumed.dat -hrex -replex 500 -v

Note that, as I did, you may need the flag `--mca opal_cuda_support 1` for running the simulations
with GPU support.
