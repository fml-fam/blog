---
title: "Computing Eigenvalues Out-of-Core"
date: 2021-10-25
tags:
  - math
  - r
---

In an [earlier post]({{< ref "2020-06-15-svd-via-lanczos-iteration.md" >}}), I discussed how you can compute SVD (and spectral decomposition) via Lanczos Iteration. Several times in that post, I mentioned that the method lends itself well to out-of-core data --- that is, data stored on disk rather than in memory. But I never bothered to actually go into any details. Recently I had the need for an out-of-core eigensolver, and I have implemented it in [this R package](https://github.com/wrathematics/hdfmat). But I would like to describe the details, because assuming you understand the Lanczos part from the earlier article, then this really isn't too complicated.

I would say that the hard part is working out-of-core. There are many choices for something like this. You can go with some kind of data standard like HDF5 (which is what I did), "roll your own" with something like `fwrite()`, or split the difference with some kind of binary xml monstrosity. I'm not an I/O specialist so I went the route I felt was easiest, even if it cost me some performance. It's also worth mentioning that I sort of feel like once you make the compromise of working out-of-core that being overly concerned with performance is a bit silly. If you have a giant dataset and performance matters, assuming you're working in open science, you should get access to a compute cluster. There are tons of free resources available.

For each Lanczos iteration $i$ from $1, \dots, k$ do:

1. Compute $v$ scalar by scalar, iterating over the rows $j$ of $A$, with $j = 1, \dots, n$
    1. Read row $j$ of $A$, denoted $A_j$
    2. Denote the $i$'th column of $Q$ by $Q^T_i$
    2. Compute the $j$'th element of $v$ (a scalar) as the dot product $v_j = A_j \cdot Q^T_i$
2. Proceed as before calculating $alpha$, $v$ updates, etc.

Some of the formalism obscures how easy this really is. Here it makes the most sense to store $Q$ in column-major storage and $A$ on disk in row-major format. Then you just have to shift your $Q$ array by $n\times i \times \texttt{sizeof}(Q)$ bytes.

Of course, you don't necessarily have to read rows. You can read whatever blocks you want that your out-of-core storage supports. The bigger the read, the more data you can operate on at a time, which will probably improve the run-time performance. I chose rows because it's easy to understand and implement. See also the above caveats about working out-of-core.

With an R-like pseudo-code, it might look something like this:

```r
# allocate/initialize alpha, beta, q as before
# ...
for (i in 1:k)
{
  for (j in 1:n)
  {
    A_j = read_row(row_offset=j)
    v[j] = A_j %*% q[, i]
  }
  
  # continue as before
}
```

I have implemented this and a few other operations useful for computing shrink correlation in the aforementioned [hdfmat package](https://github.com/wrathematics/hdfmat). My primary motivation was to make it easy to compute the eigenvalues of large shrink correlation matrices out-of-core, which actually dovetails with another [earlier post]({{< ref "2021-06-29-Matrix-Computations-in-Constrained-Memory-Environments.md" >}}).

The hdfmat package uses HDF5 as the on-disk matrix container. As with basically anything, there are some tradeoffs here. HDF5 is a standard in HPC, so I was already familiar with it. If performance is critical, you could probably do better. But this really is the hill I will die on: if performance is so critical, then why are you working out-of-core in the first place?

Here's a little benchmark of the package. I used 8 cores of an 8-core AMD processor, and the disk is some garbage SSD that will probably die in 6 months. OpenBLAS is doing most of the hard work. The benchmark computes the crossproduct matrix of a "large"-ish matrix and then computes various numbers of estimates of the eigenvalues. These are basically the bottlenecks for the shrink correlation thing I mentioned earlier.

```r
library(hdfmat)
set.seed(1234)

f = tempfile()
name = "mydata"
type = "float"

m = 1000
n = 100000

system.time({
  x = matrix(rnorm(m*n), m, n)
})
##    user  system elapsed 
##   4.245   0.294   4.539 

system.time({
  h = hdfmat::hdfmat(f, name, n, n, type)
})
##    user  system elapsed 
##   0.003   0.007   0.011 

system.time({
  h$crossprod(x)
})
##      user    system   elapsed 
## 25721.003  1227.089  3375.184 

system.time({
  h$eigen(k=3)
})
##    user  system elapsed 
## 196.455  21.059  27.211 

system.time({
  h$eigen(k=100)
})
##     user   system  elapsed 
## 6149.752  641.622  849.200 

system.time({
  h$eigen(k=1000)
})
##      user    system   elapsed 
## 61362.140  6363.350  8468.052 
```

When I check the resulting file with `du -h`, I see:

```
38G	rdata.h5
```

Which is reasonable given the size to store it in-memory is pretty close to that:

```r
memuse::howbig(n, n, prefix="SI")/2
## 40.000 GB
```

Amusingly, I have enough memory to store that matrix on this workstation, but not enough to store it and compute the eigenvalues with a call to something like `eigen()`:

```r
memuse::Sys.meminfo()
## Totalram:  62.814 GiB 
## Freeram:   50.744 GiB 
```

That's because `eigen()` will make a copy of the matrix, and I only have enough room for one on this workstation. However, I can actually compute this using [fmlr](https://github.com/fml-fam/fmlr):

```r
suppressMessages(library(fmlr))
set.seed(1234)

m = 1000
n = 100000

system.time({
  x = cpumat(m, n, type="float")
  x$fill_rnorm()
})
##  user  system elapsed 
## 1.519   0.045   1.565 

system.time({
  cp = linalg_crossprod(1, x)
})
##    user  system elapsed 
## 212.216  10.629  36.144 

system.time({
  ev = cpuvec(type="float")
  linalg_eigen_sym(x, ev)
})
##   user  system elapsed 
## 40.442   9.524   6.720
```

Which is just *a bit* faster.
