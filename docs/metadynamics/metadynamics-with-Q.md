## Defining native contacts (Q) as collective variable
In order to understand how to implement Q as a collective variable in PLUMED
one needs to understand how Q is defined:

$$Q(X) = \frac{1}{N}\sum_{\left(i,i\right)}\frac{1}{1+exp\left[\beta \left(r_{ij}(X)-\lambda r^{0}_{ij}\right)\right]}$$

where the sum runs over the _N_ pairs of native contacts _(i,j)_, _r_ $`_{ij}`$ _(X)_ is the 
distance between _i_ and _j_ in configuration _X_, _r_ $`^{0}_{ij}`$ is the distance between
_i_ and _j_ in the native state, $\beta$ is a smoothing parameter taken to be 5 Å$`^{-1}`$ 
and the factor $`\lambda`$ accounts for fluctuations when the contact is formed, taken to be
1.2.

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




Once the structure is loaded I run the _DESHAW\_q()_ code from the MDTraj library.
