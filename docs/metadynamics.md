<script
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"
  type="text/javascript">
</script>

## Metadynamics simulations using PLUMED
One of the key functionalities from PLUMED is the possibility
of running simulations using metadynamics. The contents from
this section are borrowed from [masterclass 
21.4](https://www.plumed.org/doc-v2.7/user-doc/html/masterclass-21-4.html), 
where the methodology is introduced in simple terms.

### Alanine dipeptide in vacuum
To run this example, you will need the input files in the `data`
folder from this repository. Then, write a `plumed.dat` file
with the following contents
```
MOLINFO STRUCTURE=dialaA.pdb
phi: TORSION ATOMS=@phi-2
psi: TORSION ATOMS=@psi-2

metad: METAD ARG=phi ...
   PACE=500 HEIGHT=1.2 BIASFACTOR=8
   SIGMA=0.3
   FILE=HILLS GRID_MIN=-pi GRID_MAX=pi
...
PRINT ARG=phi,psi FILE=colvarA.dat STRIDE=10
```
The key difference with umbrella sampling is that we are biasing 
with the `METAD` action. This allows us to explore the whole Ramachandran
map in a single run. 

Again, we can simply run our simulation using the usual Gromacs command
with an additional `-plumed` flag
```
foo@bar:~$ gmx mdrun -s topolA.tpr --deffnm metadA -plumed plumed.dat -nsteps 10000000
```

The most important differences come in the analysis. The simulation 
has generated a file `HILLS` including the information corresponding
to the adaptive biases added during the simulation. We can estimate 
the free energy surface on the biasing coordinate using the `sum_hills`
command
```
foo@bar:~$ plumed sum_hills --hills HILLS --stride 500 --mintozero
```
This results in a series of files winth free energy surfaces on the
collective variable with increasingly large amounts of data. Hopefully, 
this will show that the free energy estimation is well converged. 

Additionally, we can use the `driver`, for more analysis. For example,
we can write the following script to reweight the simulation 
data and obtain the energy surface not only on the biased coordinate
but also on other coordinates (in this case, the psi torsion angle)
```
MOLINFO STRUCTURE=dialaA.pdb
phi: TORSION ATOMS=@phi-2
psi: TORSION ATOMS=@psi-2
metad: METAD ARG=phi ...
   PACE=500 HEIGHT=1.2 BIASFACTOR=8
   SIGMA=0.3
   FILE=HILLS GRID_MIN=-pi GRID_MAX=pi
   RESTART=YES
...
bias: REWEIGHT_BIAS ARG=metad.bias
hhphi: HISTOGRAM ARG=phi STRIDE=50 GRID_MIN=-pi GRID_MAX=pi GRID_BIN=50 BANDWIDTH=0.05 LOGWEIGHTS=bias
hhpsi: HISTOGRAM ARG=psi STRIDE=50 GRID_MIN=-pi GRID_MAX=pi GRID_BIN=50 BANDWIDTH=0.05 LOGWEIGHTS=bias
ffphi: CONVERT_TO_FES GRID=hhphi
ffpsi: CONVERT_TO_FES GRID=hhpsi
DUMPGRID GRID=ffphi FILE=ffphi.dat
DUMPGRID GRID=ffpsi FILE=ffpsi.dat
PRINT ARG=phi,psi,metad.bias FILE=colvar_reweight.dat STRIDE=1
```
Then, on the command line, we run
```bash
foo@bar:~$ plumed driver --mf_xtc metadA.xtc --plumed plumed_reweight.dat --kt 2.494339
```
The results are output to the `ffphi.dat` and `ffpsi.dat` files, the latter
containing a notably noisy free energy landscape. 

We may want to overcome this problem adding a bias in two independent coordinates.
```
MOLINFO STRUCTURE=dialaA.pdb
phi: TORSION ATOMS=@phi-2
psi: TORSION ATOMS=@psi-2

metad: METAD ARG=phi,psi ...
   PACE=500 HEIGHT=1.2 BIASFACTOR=8
   SIGMA=0.3,0.3
   FILE=HILLS2d GRID_MIN=-pi,-pi GRID_MAX=pi,pi
...
PRINT ARG=phi,psi FILE=colvarA2d.dat STRIDE=10
```
Again, the `HILLS` file can be analyzed using `plumed sum_hills`to produce a smooth
two dimensional free energy landscape.
![2D free energy lansdcape on Ramachandran space](img/meta2d.png)

