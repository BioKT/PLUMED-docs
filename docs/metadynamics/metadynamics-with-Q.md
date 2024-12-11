<script
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"
  type="text/javascript">
</script>

## Defining native contacts (Q) as collective variable
In order to understand how to implement Q as a collective variable in PLUMED
one needs to understand how Q is defined:

```
$$ 
Q(X) = \frac{1}{N} 
\sum_{\left(i,i\ritgh)\frac{1}{1+exp\left[\beta \left(r_{ij}(X)-\lambda r^{0}_{ij}\right)]}}
$$
```

