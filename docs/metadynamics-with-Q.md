## Defining native contacts (Q) as collective variable
In order to understand how to implement Q as a collective variable in PLUMED
one needs to understand how Q is defined:

<center><img src="https://latex.codecogs.com/svg.image?Q(X)=\frac{1}{N}\sum_{\left(i,i\right)}\frac{1}{1&plus;exp\left[\beta\left(r_{ij}(X)-\lambda&space;r^{0}_{ij}\right)\right]}"><center>

where the sum runs over the _N_ pairs of native contacts _(i,j)_, <img src="https://latex.codecogs.com/svg.image?r_{ij}(X)"> 
is thedistance between _i_ and _j_ in configuration _X_, <img src="https://latex.codecogs.com/svg.image?r_{ij}^{0}"> 
is the distance between _i_ and _j_ in the native state, <img src="https://latex.codecogs.com/svg.image?\beta"> 
is a smoothing parameter taken to be 5 <img src="https://latex.codecogs.com/svg.image?\AA ^{-1}"> 
and the factor <img src="https://latex.codecogs.com/svg.image?\lambda"> accounts for fluctuations when the 
contact is formed, taken to be 1.2.

Therefore, the first step is to compute the number of native contacts of our system. To do so,
I load the native structure. In the case of this example, this native structure has been loaded
as a PDB file, although one could select the file type of their preference.

```
pdb = md.load("Heliat-1/Heliat-1.pdb")
```

For this particular example, I will consider my native contacts to be exclusively given by
C$`\alpha`$ atoms. Therefore, I make a dictionary to save the atom index of each C$`\alpha`$
atom alongside their residue index number (this will be usefull in a future step):

```
ca = {}

for i in range(1,17):
    if (i == 1):
        ca[top.select("residue %i and name C"%i)[0]] = i
    else:
        ca[top.select("residue %i and name CA"%i)[0]] = i
```

Notice that I am working with an Heliat, which is cyclated using the sidechain of a CYS and an
ACE. The C atom of this ACE is involved in the formation of $`\alpha`$-helix and therefore I
define it as an C$`\alpha`$ atom. Once the atom selection is done, I compute the number of native
contacts:

```
NATIVE_CUTOFF = 1.0  # nanometers

alpha_carbons = [x for x in ca.keys()]

Ca_pairs = np.array(
    [(i,j) for (i,j) in combinations(alpha_carbons, 2)
        if abs(pdb.topology.atom(i).residue.index - \
                pdb.topology.atom(j).residue.index) > 2])

Ca_pairs_distances = md.compute_distances(pdb[0], Ca_pairs)[0]

native_contacts = Ca_pairs[Ca_pairs_distances <= NATIVE_CUTOFF]
```

In this case, I am considering as a native contact any interaction between to C$`\alpha`$ atoms that are
at least at 2 residues apart and that their distance is lower than 1 nm. These native contacts are
then stored in the _native\_contacts_ array, which should look like this:

```
array([[  3,  38],
       [  3,  59],
       [  3,  69],
       [  7,  59],
...
```

where each number belongs to the atom index of each C$`\alpha`$. Since I want to translate this to a synthax 
PLUMED can understand, I translate this contacts from atom indexes to residue indexes as follows:

```
pairs = []

for i in range(len(native_contacts)):
    pairs.append([ca[native_contacts[i,0]], ca[native_contacts[i,1]]])
```

Once this step is performed, I am now ready to start writing the first actual PLUMED input files. To do so,
the first thing I do is to create a file where I select each C$`\alpha`$ internally with PLUMED. I will call
this file `CA-list.dat` and it will contain the following:

```
file = open("Heliat-1/CA-list.dat", "w")

file.write("MOLINFO STRUCTURE=Heliat-1.pdb \n")

for i in range(1, 17):
    if (i == 1):
        file.write("c%i: GROUP ATOMS={@mdt:{residue %i and (name C)}}"%(i,i))
        file.write("\n")
    else:
        file.write("c%i: GROUP ATOMS={@mdt:{residue %i and (name CA)}}"%(i,i))
        file.write("\n")

file.close()
```

This file contains the information of where the atom selection is being made from (PDB file in this case) and
makes the atom selection internally making use of the MDTraj synthax thanks to the `@mdt` flag. Once this step
is done, I will write the actual PLUMED input file that contains the information for the metadynamics 
simulation. In order to fill this file, one should understand how Q should be defined as a collective variable (CV)
in PLUMED. The first step is to define a contact map where the contacts are your native contacts. Inside the 
definition of this contact map, one should define five important parameters: cutoff distance of a native contact, 
$`\beta`$, $`\lambda`$, _r_ $`^{0}_{ij}`$ and weight (which should be 1 divided by the number of native contacts).
The synthax is the following:

```
__ARG1__: CONTACTMAP ...
   ATOMS=atom1,atom2 SWITCH1={Q R_0=1.0 BETA=50.0 LAMBDA=1.2 REF=__ARG2__} WEIGHT=__ARG3__
   ...
   SUM
...
```
where `__ARG1__` should be substituted by the name with which one wants to define the contact map (for now on,
I will be using `cmap`), `__ARG2__` should be substituted by the reference distance (_r_ $`^{0}_{ij}`$) and
`__ARG3__` should be substituted by the weight of each distance (1 divided by the number of native contacts).
In order to let PLUMED know that it should perform a calculation of Q, we add the Q inside the `SWITCH` function
and add the `SUM` line at the end of the definition.

Once the function is defined, I will add the following lines to define this contact map (`cmap` in my case) as
a CV:

```
metad: METAD ARG=cmap ...
   PACE=500 HEIGHT=1.2
   SIGMA=__ARG4__
   FILE=HILLS GRID_MIN=0 GRID_MAX=1
...
```

where `PACE` defines the frequency (in frames) with which a new Gaussian hill will be added,`HEIGHT` defines the hieghts
of the Gaussian hills and `SIGMA` defines the width of the Gaussian hills. Here,  `__ARG4__` should be defined depending 
on each individual case. PLUMED recommends to set the value of `SIGMA` to be a third of the fluctuations of the CV one
is using. In this particular case, `SIGMA` should be of around 0.005-0.01. Lastly, `GRID_MIN` and `GRID_MAX` define the
lower and upper bound of the grid. The information of the Gaussian hills will then be written to the `HILLS` file. This
will be important in future steps.

In order to create an input file to run a metadynamics calculation with PLUMED which will be named `plumed.dat` I run 
the following lines:

```
file = open("Heliat-1/plumed.dat", "w")

w = 1 / len(pairs)

ref = md.compute_distances(pdb, native_contacts)[0]

file.write("INCLUDE FILE=CA-list.dat \n \n")
file.write("cmap: CONTACTMAP ... \n")

c = 1
for i in pairs:
    file.write("   ATOMS%i=c%i,c%i SWITCH%i={Q R_0=1.0 BETA=50.0 LAMBDA=1.2 REF=%.6f} WEIGHT%i=%.6f \n"%(c,i[0],i[1],c,ref[c-1],c,w))
    c += 1

file.write("\n   SUM \n")
file.write("... \n")

file.write("metad: METAD ARG=cmap ... \n")
file.write("   PACE=500 HEIGHT=1.2 \n")
file.write("   SIGMA=0.005 \n")
file.write("   FILE=HILLS GRID_MIN=0 GRID_MAX=1 \n \n")
file.write("... \n")

file.write("\nPRINT ARG=cmap FILE=COLVAR")

file.close()
```

Once the `plumed.dat` file is created, the metadynamics simulation is ready to be performed. One should run it in an
environment where MDTraj is present (as it is imprescindible to define the C$`\alpha`$ atoms we defined in the `CA-list.dat` file.
The line to run this calculation is the following:

```
gmx_mpi mdrun -s *tpr -plumed plumed.dat -ntomp 1
```

Once the calculation is finished, one can extract the evolution of how Q changed during the simulation from the
`COLVAR` file, where the first column gives the time value and the second column gives the value of Q. Regarding
the `HILLS` file, in order to extract information one must run the following line:

```
plumed sum_hills --hills HILLS
```

Once this line is run, a file named `fes.dat` will be created, which containts the information of the Free Energy Surface
of your metadynamics calculation.

Lets see
