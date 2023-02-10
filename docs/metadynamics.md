<script
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"
  type="text/javascript">
</script>

## Metadynamics simulations using PLUMED
One of the key functionalities from PLUMED is the possibility
of running simulations using metadynamics. The contents from
this section are borrowed from [masterclass 
21.4](https://www.plumed.org/doc-v2.7/user-doc/html/masterclass-21-4.html), where the methodology is introduced in simple terms.

### Alanine dipeptide in vacuum
To run this example, you will need the input files in the `data`
folder from this repository. Then, write a `plumed.dat` file
with the following contents

    MOLINFO STRUCTURE=dialaA.pdb
    phi: TORSION ATOMS=@phi-2
    psi: TORSION ATOMS=@psi-2
    
    metad: METAD ARG=phi ...
       PACE=500 HEIGHT=1.2 BIASFACTOR=8
       SIGMA=0.3
       FILE=HILLS GRID_MIN=-pi GRID_MAX=pi

The key difference with umbrella sampling is that we are biasing 
with the `METAD` action.
