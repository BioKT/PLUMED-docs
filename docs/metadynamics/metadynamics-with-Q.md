<script
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"
  type="text/javascript">
</script>

\usepackage{amsmath}
\usepackage{mathtools}

## Defining native contacts (Q) as collective variable
In order to understand how to implement Q as a collective variable in PLUMED
one needs to understand how Q is defined:

$$Q(X) = \frac{1}{N}\sum_{\left(i,i\right)}\frac{1}{1+exp\left[\beta \left(r_{ij}(X)-\lambda r^{0}_{ij}\right)\right]}$$

where the sum runs over the _N_ pairs of native contacts _(i,j)_, _r_ $_{ij}$ _(X)_ is the 
distance between _i_ and _j_ in configuration _X_, r\^{0}_{ij}\ is the distance between
_i_ and _j_ in the native state, $\beta$ is a smoothing parameter taken to be 5 $\{AA}^{-1}$ 
and the factor $\lambda$ accounts for fluctuations when the contact is formed, taken to be
1.2.
