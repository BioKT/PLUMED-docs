<script
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"
  type="text/javascript">
</script>

## Umbrella sampling with PLUMED

### Alanine dipeptide in vacuum
First of all, we will download the data required to run the calculations, which 
should be in the `data/umbrella` folder of this repository.
It should contain the following files from the 
[21.3 PLUMED masterclass](https://www.plumed.org/doc-v2.7/user-doc/html/masterclass-21-3.html).
```bash
foo@bar:~$ ls data/umbrella/
   ala_dipeptide_analysis.ipynb topolA.tpr                   wham.py
   reference.pdb                topolB.tpr
```
Using the typical Gromacs syntax, you could use these `tpr` files 
to run MD simulations using 
```bash
foo@bar:~$ gmx mdrun -s topolA.tpr -nsteps 200000 -x traj_unbiased.xtc
```
This will result in an unbiased simulation trajectory of the 
alanine dipeptide. 

When we run simulations with the PLUMED we will use an additional
flag `-plumed` that points to the unique input required to bias the 
simulations. Below, you can find an example input file to
estimate the values of the \\( \phi \\) and \\( \psi \\) dihedrals 
and histogram their values.

```
MOLINFO STRUCTURE=reference.pdb
phi: TORSION ATOMS=@phi-2
psi: TORSION ATOMS=@psi-2

# make histograms
hhphi: HISTOGRAM ARG=phi STRIDE=100 GRID_MIN=-pi GRID_MAX=pi GRID_BIN=600 BANDWIDTH=0.05
ffphi: CONVERT_TO_FES GRID=hhphi
DUMPGRID GRID=ffphi FILE=fes_phi.dat STRIDE=200000

hhpsi: HISTOGRAM ARG=psi STRIDE=100 GRID_MIN=-pi GRID_MAX=pi GRID_BIN=600 BANDWIDTH=0.05
ffpsi: CONVERT_TO_FES GRID=hhpsi
DUMPGRID GRID=ffpsi FILE=fes_psi.dat STRIDE=200000

PRINT ARG=phi,psi FILE=colvar.dat STRIDE=100
```
Note that we are using a number of actions (`TORSION`, 
`HISTOGRAM` and `DUMPGRID`, to name a few). You should 
familiarize yourself with their 
[syntax](https://www.plumed.org/doc-v2.7/user-doc/html/glossary.html).

Introducing an umbrella bias in this script is very 
easy. It just requires adding an extra line to the script.
For example, to restrain the simulation with a harmonic
potential at 0 radians with a spring constant of 200,
we would write 
```
# add restraining potential
bb: RESTRAINT ARG=phi KAPPA=200.0 AT=0
```

Of course, we will be interested in running multiple 
window umbrella sampling, and hence the structure of
our directory should change with as many plumed files
as umbrellas windows. To generate 32 PLUMED input files, 
we can use the following Python script:
```python
at=np.linspace(-np.pi,np.pi,33)[:-1]
print(at)
for i in range(32):
    with open("plumed_" + str(i) + ".dat","w") as f:
        print("""
# vim:ft=plumed
MOLINFO STRUCTURE=reference.pdb
phi: TORSION ATOMS=@phi-2 
psi: TORSION ATOMS=@psi-2
bb: RESTRAINT ARG=phi KAPPA=200.0 AT={}
PRINT ARG=phi,psi,bb.bias FILE=colvar_multi_{}.dat STRIDE=100
""".format(at[i],i),file=f)
```

And then we will run the 32 simulations
```bash
foo@bar:~$ for i in $(seq 0 1 31)
do 
   gmx mdrun -plumed plumed_${i}.dat -s topolA.tpr -nsteps 200000 -x traj_comp_${i}.xtc
done
```
The result from this will be a dataset of 32 simulations, all
initialized at the same conformation, but with umbrella potentials
that will bias the sampling to different regions of conformational
space.
 
After completion of the runs, we must analyze the resulting 
trajectories. In order to do that we concatenate the trajectories
```bash
foo@bar:~$ gmx trjcat -cat -f `for i in $(seq 0 1 31); do echo "traj_comp_${i}.xtc"; done` -o traj_multi_cat.xtc
```
Then, we analyze them using plumed driver:
```bash
foo@bar:~$ for i in $(seq 0 1 31)
do 
    plumed driver --plumed plumed_${i}.dat --ixtc traj_multi_cat.xtc --trajectory-stride 100
done
```

To visualize the output of the simulations, we can read the 
`colvar_multi_\*.dat` files and see what region of the 
Ramachandran map has been sampled by each of the runs. Using
a Jupyter-notebook we can run the following 
```python
import plumed
import wham

col=[]
for i in range(32):
    col.append(plumed.read_as_pandas("colvar_multi_" + str(i)+".dat"))
 
bias = np.zeros((len(col[0]["bb.bias"]),32))
for i in range(32):
    bias[:,i] = col[i]["bb.bias"][-len(bias):]
w = wham.wham(bias,T=kBT)
colvar = col[0]
colvar["logweights"] = w["logW"]
plumed.write_pandas(colvar,"bias_multi.dat")
```
We then write a file called `plumed_multi.dat` to obtain 
reweighted free energy surfaces. The file should look as
follows:
```
# vim:ft=plumed
phi: READ FILE=bias_multi.dat VALUES=phi IGNORE_TIME
psi: READ FILE=bias_multi.dat VALUES=psi IGNORE_TIME
lw: READ FILE=bias_multi.dat VALUES=logweights IGNORE_TIME

hhphi: HISTOGRAM ARG=phi GRID_MIN=-pi GRID_MAX=pi GRID_BIN=600 BANDWIDTH=0.05
ffphi: CONVERT_TO_FES GRID=hhphi
DUMPGRID GRID=ffphi FILE=fes_phi_cat.dat

hhpsi: HISTOGRAM ARG=psi GRID_MIN=-pi GRID_MAX=pi GRID_BIN=600 BANDWIDTH=0.05
ffpsi: CONVERT_TO_FES GRID=hhpsi
DUMPGRID GRID=ffpsi FILE=fes_psi_cat.dat

hhphir: HISTOGRAM ARG=phi GRID_MIN=-pi GRID_MAX=pi GRID_BIN=600 BANDWIDTH=0.05 LOGWEIGHTS=lw
ffphir: CONVERT_TO_FES GRID=hhphir
DUMPGRID GRID=ffphir FILE=fes_phi_catr.dat

hhpsir: HISTOGRAM ARG=psi GRID_MIN=-pi GRID_MAX=pi GRID_BIN=600 BANDWIDTH=0.05 LOGWEIGHTS=lw
ffpsir: CONVERT_TO_FES GRID=hhpsir
DUMPGRID GRID=ffpsir FILE=fes_psi_catr.dat
```
This script is again read by plumed driver, and from this program we get our outputs.
```bash
foo@bar:~$ plumed driver --plumed plumed_multi.dat --ixtc traj_multi_cat.xtc --trajectory-stride 100 --kt 2.4943387854
```

In the folder where you have found the data to run these tests you can find a script
 that will let you plot the free energy landscapes as estimated from umbrella
 sampling and compare them with those from the equilibrium MD trajectory.
Clearly, using umbrella sampling we have been able to cover much more ground 
for our reaction coordinates, while obtaining a very consistent PMF in the
 regions that actually matter. One of the assignments in the masterclass is to run 
from a different initial state, that corresponding to the `topolB.tpr` Gromacs input file.

#### Running with multiple replicas
Typically, if we want to do molecular simulations using umbrella sampling, we will 
take advantage of the parallel capabilities of modern computers. Additionally, 
we may able to swap conformations between umbrella windows in order to enhance the
sampling. This may be helpful to avoid getting trapped in local minima. The specifics
on how to do this using PLUMED in Gromacs are described in the first part of the 
[PLUMED 21.5 masterclass](https://www.plumed.org/doc-v2.7/user-doc/html/masterclass-21-3.html).
In order to take advantage of this, the software must have been compiled using MPI.

The PLUMED input file for this type of calculation is only incrementally more complicated
than the one used before
```
MOLINFO STRUCTURE=../reference.pdb
phi: TORSION ATOMS=@phi-2
psi: TORSION ATOMS=@psi-2
bb: RESTRAINT ARG=phi KAPPA=200.0 AT=@replicas:-3.141,-2.945,-2.748,-2.552,-2.356,-2.159,-1.963,-1.767,-1.570,-1.374,-1.178,-0.981,-0.785,-0.589,-0.392,-0.196,0.0,0.196,0.392,0.589,0.785,0.981,1.178,1.374,1.570,1.767,1.963,2.159,2.356,2.552,2.748,2.945
PRINT ARG=phi,psi,bb.bias FILE=../colvar_multi.dat STRIDE=100
```
Note the `AT=@replicas` section in the `RESTRAINT` action. In this case, we must generate
as many directories (here termed `run0`, `run1`, and so on), each of which will contain 
the `tpr` file. Simulations are then run using
```bash
foo@bar:~$ mpiexec -np 32 --oversubscribe gmx_mpi mdrun -multidir run* -plumed ../plumed.dat -s topolA.tpr -replex 200 -nsteps 200000 
```
While much of the analysis will remain unchanged from the case of running each umbrella
window independently, it is important to carefully check in the 
[demuxed trajectories](https://github.com/srnas/demux) whether
the different copies of the system diffuse across windows. 
Some copies may remain isolated from the rest, which may have catastrophic results. 
