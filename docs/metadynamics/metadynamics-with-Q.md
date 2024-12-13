<script
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"
  type="text/javascript">
</script>

\usepackage{amsmath}

## Defining native contacts (Q) as collective variable
In order to understand how to implement Q as a collective variable in PLUMED
one needs to understand how Q is defined:

$$Q(X) = \frac{1}{N}\sum_{\left(i,i\right)}\frac{1}{1+exp\left[\beta \left(r_{ij}(X)-\lambda r^{0}_{ij}\right)\right]}$$

where the sum runs over the $N$ pairs of native contacts $$(i,j)$$, $$r_{ij}(X)$$ is the 
distance between $`i`$ and $`j`$ in configuration $`X`$, $`r^{0}_{ij}`$ is the distance between
&i; and &j; in the native state, $`\beta`$ is a smoothing parameter taken to be 5 $`\AA ^{-1}`$ 
and the factor $`\lambda`$ accounts for fluctuations when the contact is formed, taken to be
1.2.
