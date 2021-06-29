---
title: "Matrix Computations in Constrained Memory Environments"
date: 2021-06-29
tags:
  - r
---

Many times in my professional career, a researcher will approach me for help performing some kind of matrix calculation that is getting "expensive" as their data grows. Sometimes the expense is in compute time: it's just taking too long. Recently, someone approached me who didn't care at all how long it took to compute something. Their problem was getting the thing to run within the RAM constraints in their particular computing environment, which they couldn't really deviate from.

This particular researcher was using R, which shows remarkably good taste on their part. And although I will be using R for the code snippets, the general principles apply to any language.

The actual problem is somewhat standard fare for someone coming from statistics. They have a few different data sets of varying sizes --- I don't want to give exact numbers, but something on the order of 100,000 rows, and a varying number of columns, order of 10 at the small end and 1000 at the large end. They were interested in computing a correlation matrix based on the rows rather than the columns; i.e. they wanted to scale the data by mean and variance to produce a matrix $S$ and then compute

$$ SS^T $$

At first I didn't blush at this because sometimes statisticians need to calculate correlation or covariance matrices to operate on their explicit values. That turns out to not be exactly what this researcher needed. Some readers may already see the punchline coming. For those who don't, this turns out to make for a remarkably good demonstration of some computational basic principles, so let us proceed with the problem as stated.

One option is to go out of core. There are a variety of R packages for this kind of thing, like [ff](https://cran.r-project.org/web/packages/ff/index.html) and [bigmemory](https://cran.r-project.org/web/packages/ff/index.html), although I have no direct experience with either and can't speak either way to how they handle numerics or what they offer. Another option is to go distributed. For that, your major options are [pbdR](https://pbdr.org/) and [fmlr](http://github.com/fml-fam/fmlr), the latter with the MPIMAT backend.

These approaches have different pros and cons. Pro of working out of core is you can stick with your workstation. A major con is that you're now reliant on the I/O performance of your disk, which sucks (yes, even your ssd). This can also be hard to do, which is a con; but as I have no direct use with the aforementioned packages, I can't comment on this. A pro of going distributed is that it's fast. The major con is that it's hard to use. Another con is that you actually need access to multi-node compute resources. And given that I have spent the last decade working in supercomputing, I sometimes forget that not everyone has the kinds of resources lying around that I do.

The researcher wasn't excited about either option: we couldn't move to a new resource, and they didn't want to orchestrate an entirely new workflow. But there is another possibility: reducing precision from double to float (64-bit to 32-bit). This will basically cut the problem size in half, at the cost of potentially worse numerical estimates (depending on how many decimal places you actually need). My belief is that for statistics in particular, float is almost always good enough (fight me). At any rate, this made the researcher's problem just barely fit in memory and seemed like a good enough compromise.

For this, we can use the [float package](https://cran.r-project.org/web/packages/float/index.html). When memory is really tight (as in this case), we need to be conscious about our copies. Here the dataset isn't the major memory consumer, but it's just big enough to be annoying when we need to compute the final crossproducts matrix. So we'll be dropping objects as we go. And throughout, we'll assume you have your data stored in the object named `df`.

So this is a little ugly, but we can do this as follows:

```r
library(float)

x = as.matrix(df)
rm(df)
invisible(gc())

x_fl = fl(x)
rm(x)
invisible(gc())

s = scale(x_fl, TRUE, TRUE)
co = tcrossprod(s) / fl(max(1L, nrow(x)-1))
```

And voila, a correlation matrix.

But unfortunately, that wasn't good enough. Because although this worked, what the researcher *really* wanted was the eigenvalues of this matrix. Yeah, that's a problem, because R will always operate on copies when data has to be modified. And when you give R's `eigen()` a symmetric matrix (as is the case here), it will eventually call out to LAPACK's `syevr()` function (specifically, `ssyevr()` in this case). That modifies the data, so R will try to copy that giant matrix that we could only barely fit in memory.

Now if you insist on this route for some reason, you can actually do this with fmlr. This is because fmlr (and the core C++ library fml) is designed to live just barely above the underlying linear algebra frameworks. It's high level enough to be easy to use, but low level enough to be performant in a way that basically no other library that I'm aware of can credibly claim to be (fight me).

fmlr will not make modifications to data in a copy --- if you don't like that, make your own damn copy. So this isn't going to win any beauty pageants, but it works:

```r
library(fmlr)

x = as.matrix(df)
rm(df)
invisible(gc())

x_fl = fl(x)
rm(x)
invisible(gc())

s = as_cpumat(x_fl, copy=FALSE)

dimops_scale(TRUE, TRUE, s)
alpha = max(1, s$nrows()-1)
co = linalg_tcrossprod(alpha, s)

rm(s)
rm(x_fl)
invisible(gc())

vals_fl = cpuvec(type="float")
linalg_eigen_sym(co, vals_fl)
vals = dbl(sqrt(vals_fl$to_robj()))
vals
```

Yeah I get it, wouldn't it be nice if we could just do `eigen(tcrossprod(x))`. That sure is a lot less typing. But the whole point is that we can't do that.

*Or can we?*

In an [earlier post]({{< ref "2020-07-03-matrix-factorizations-for-data-analysis.md" >}}) I outlined the details of the well-known connection between the square of the singular values of a matrix and the eigenvalues its derived crossproducts matrices. I won't go through it all again here, but basically we can solve the problem like this:

```r
sqrt(svd(df, nu=0, nv=0)$d)
```

And this can be done pretty quickly computationally, and it doesn't really take all that much memory. The cost is mostly in two copies of the data: one going from the dataframe to a numeric matrix, and the second created before the data is modified by the LAPACK `dgesdd()` driver. There is some additional overhead for workspace for the LAPACK function, but it's not substantial.

But if you were still highly memory constrained, you could get by with a single copy, and push the calculation down to float as before:

```r
library(float)

x = as.matrix(df)
rm(df)
invisible(gc())

x_fl = fl(x)
rm(x)
invisible(gc())

dbl(sqrt(svd(x_fl, nu=0, nv=0)$d))
```

And if even that is pushing the limits, you can avoid the extra copy using fmlr as before:

```r
library(fmlr)

x = as.matrix(df)
rm(df)
invisible(gc())

x_fl = fl(x)
rm(x)
invisible(gc())

s = as_cpumat(x_fl, copy=FALSE)

v = cpuvec(type="float")
linalg_svd(s, v)

rm(s)
rm(x_fl)
invisible(gc())

dbl(sqrt(v$to_robj()))
```
