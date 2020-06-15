---
title: "SVD via Lanczos Iteration"
date: 2020-06-15
tags:
  - math
---

## Background

Every few years, I try to figure out the Lanczos method to approximate SVD of a rectangular matrix. Unfortunately, every resource I find always leaves out enough details to confuse me. All of the information I want is available across multiple writeups, but everyone uses different notation, making things even more confusing.

This time I finally sat down and got to a point where I finally felt like I understood it. This writeup will hopefully clarify how you can use the Lanczos iteration to estimate singular values/vectors of a rectangular matrix. It's written for the kind of person who is interested in implementing it in software. If you want more information or mathematical proofs, I cite the important bits in the footnotes. Anything uncited is probably answered in [^gvl].

I assume you know what singular/eigen-values/vectors are. If you don't know why someone would care about computing approximations of them cheaply, a good applications is truncated PCA. If you just take (approximations of) the first few rotations, then you can visualize your data in 2 or 3 dimensions. And if your dataset is big, that can be a valuable time saver.

Throughout, we will use the following notation. For a vector $v$, let $\lVert v \rVert_2$ denote the Euclidean norm $\sqrt{\sum v_i^2}$. Whenever we refer to the norm of a vector, we will take that to be the Euclidean norm.

We can "square up" any rectangular matrix $A$ with dimension $m\times n$ by padding with zeros to create the square, symmetric matrix $H$ with dimension $(m+n)\times (m+n)$:

$$
\begin{align*}
H = 
\begin{bmatrix}
  0 & A \\\\
  A^T & 0
\end{bmatrix}
\end{align*}
$$

Finally, given scalars $\alpha_1, \dots, \alpha_k$ and $\beta_1, \dots, \beta_{k-1}$, we can form the tri-diagonal matrix $T_k$:

$$
\begin{align*}
T_k = 
\begin{bmatrix}
  \alpha_1 & \beta_1 & 0 & \dots & 0 & 0 \\\\
  \beta_1 & \alpha_2 & \beta_2 & \dots & 0 & 0 \\\\
  0 & \beta_2 & \alpha_3 & \dots & 0 & 0 \\\\
  \vdots & \vdots & \vdots & \ddots & \vdots & \vdots \\\\
  0 & 0 & 0 & \dots & \alpha_{k-1} & \beta_{k-1} \\\\
  0 & 0 & 0 & \dots & \beta_{k-1} & \alpha_k
\end{bmatrix}
\end{align*}
$$



## Lanczos Iteration

We're going to define the Lanczos iteration (Algorithm 36.1 of [^nla]) in the abstract. Its mathematical motivation is probably too complicated for anyone who would find this writeup helpful. For now, we will just think of it as a process that may have some application in the future.

* Inputs: 
    * Square, real, symmetric matrix $A$ of dimension $n\times n$
    * Integer $1\leq k \leq n$.
* Outputs:
    * $k$-length vector $\alpha = \left[ \alpha_1, \alpha_2, \dots, \alpha_k \right]^T$
    * $k$-length vector $\beta = \left[ \beta_1, \beta_2, \dots, \beta_k \right]^T$
    * $n\times k$-dimensional matrix $Q_k = \left[ q_1, q_2, \dots, q_k \right]$ (the $q_i$ are column vectors).
* Initialize $q_1$ to a random vector with norm 1. For the sake of notational convenience, treat $\beta_0=0$ and $q_0$ to be an $n$-length vector of zeros.
* For $i = 1, 2, \dots, k$:
    * $v = Aq_i$
    * $\alpha_i = q^T_i v$
    * $v = v - \beta_{i-1}q_{i-1} - \alpha_i q_i$
    * $\beta_i = \lVert v \rVert_2$
    * $q_{i+1} = \frac{1}{\beta_i}v$

In R, this might look like:

```r
l2norm = function(x) sqrt(sum(x*x))

lanczos = function(A, k=1)
{
  n = nrow(A)
  
  alpha = numeric(k)
  beta = numeric(k)
  
  q = matrix(0, n, k)
  q[, 1] = runif(n)
  q[, 1] = q[, 1]/l2norm(q[, 1])
  
  for (i in 1:k)
  {
    v = A %*% q[, i]
    alpha[i] = crossprod(q[, i], v)
    
    if (i == 1)
      v = v - alpha[i]*q[, i]
    else
      v = v - beta[i-1]*q[, i-1] - alpha[i]*q[, i]
    
    beta[i] = l2norm(v)
    
    if (i<k)
      q[, i+1] = v/beta[i]
  }
  
  list(alpha=alpha, beta=beta, q=q)
}
```

As presented, this has nothing to do with calculating eigen-and/or-singular values/vectors. That is the subject of the next section.



## Application to Spectral Decomposition and SVD

If $A$ is a square, symmetric matrix of order $n$, then you can estimate its **eigenvalues** by applying the lanczos method. The $\alpha$ and $\beta$ values can be used to form the tridiagonal matrix $T_k$, and the eigenvalues of $T_k$ approximate the eigenvalues of $A$ (Theorem 12.5 of [^lsl]). For the **eigenvectors**, let $S_k$ be the eigenvectors of $T_k$. Then approximations to the eigenvectors are given by $Q_k S_k$ (Theorem 12.6 of [^lsl]). In practice with $k\ll n$ the $T_k$ will be small enough that explicitly forming it and using a symmetric eigensolver like LAPACK's `Xsyevr()` or `Xsyevd()` will suffice. You could also use `Xsteqr()` instead.

To calculate the **SVD** of a rectangular matrix $A$ of dimension $m\times n$ and $m>n$, you square it up to the matrix $H$. With $H$, you now have a square, symmetric matrix of order $m+n$, so you can perform the Lanczos iteration $k$ times to approximate the eivenvalues of $H$ by the above. Note that for full convergence, we need to run the iteration $m+n$ times, not $n$ (which would be a very expensive way of computing them). The approximations to the **singular values** of $A$ are given by the non-zero eigenvalues of $H$. On the other hand, the **singular vectors** (left and right) are found in $Y_k := \sqrt{2}\thinspace Q_k S_k$ (Theorem 3.3.4 of [^anla]). Specifically, the first $m$ rows of $Y_k$ are the left singular vectors, and the remaining $n$ rows are the right singular vectors (note: not their transpose).

Since the involvement of the input matrix $A$ in computing the Lanczos iteration is in the matrix-vector product $Hq_i$, it's possible to avoid explicitly forming the squard up matrix $H$, or even to apply this to sparse problems, a common application of the algorithm. For similar reasons, this makes it very simple (or as simple as these things can be) to use it in [out-of-core algorithms](https://en.wikipedia.org/wiki/External_memory_algorithm) (although not [online](https://en.wikipedia.org/wiki/Online_algorithm) variants, given the iterative nature). Finally, note that if you do not need the eigen/singular vectors, then you do not need to store all column vectors $q_i$ of $Q_k$.

If you have $m<n$ then I think you would just use the usual "transpose arithmetic", although maybe even the above is fine. I haven't thought about it and at this point I don't care.

## Key Takeaways and Example Code

tldr

* Spectral Decomposition
    * Let $A$ be a real-valued, square, symmetric matrix of order $n$.
    * **Eigenvalues**: The non-zero eigenvalues of $T_k$ (formed by Lanczos iteration on the matrix $A$) for $k\leq n$ are approximations of the corresponding (ordered greatest to least) eigenvalues values of $A$.
    * **Eigenvectors**: If $S_k$ are the eigenvectors of $T_k$, then the eigenvectors of $A$ are approximated by $Q_k S_k$.
* SVD
    * Let $A$ be a real-valued matrix with dimension $m\times n$ and $m<n$. Let $H$ be the matrix formed by "squaring up" $A$.
    * **Singular values**: The non-zero eigenvalues of $T_k$ (formed by Lanczos iteration on the matrix $H$) for $k\leq m+n$ are approximations of the corresponding (ordered greatest to least) singular values values of $A$.
    * **Singular vectors**: If $S_k$ are the eigenvectors of $T_k$, then $\sqrt{2}\thinspace Q_k S_k$ contain approximations to the left (first $m$ rows) and right (last $n$ rows) singular vectors.
    * If $H$ is full rank, as $k\rightarrow m+n$ (not $n$), then the approximation becomes more accurate (if calculated in exact arithmetic).
    * Experimentally I find that using twice the number of Lanczos iterations as desired singular values seems to work well. Examination of the error analysis (Theorem 12.7 of [^lsl]) may reveal why, but I haven't thought about it.

Ignoring some of the performance/memory concerns addressed above, we might implement this in R like so:

```r
square_up = function(x)
{
  m = nrow(x)
  n = ncol(x)
  
  rbind(
    cbind(matrix(0, m, m), x),
    cbind(t(x), matrix(0, n, n))
  )
}

tridiagonal = function(alpha, beta)
{
  n = length(alpha)
  td = diag(alpha)
  for (i in 1:(n-1))
  {
    td[i, i+1] = beta[i]
    td[i+1, i] = beta[i]
  }
  
  td
}

#' @param A rectangular, numeric matrix with `nrow(A) > ncol(A)`
#' @param k The number of singular values/vectors to estimate. The number
#' of lanczos iterations is taken to be twice this number. Should be
#' "small".
#' @param only.values Should only values be returned, or also vectors?
#' @return Estimates of the first `k` singular values/vectors. Returns
#' the usual (for R) list of elements `d`, `u`, `v`.
lanczos_svd = function(A, k, only.values=TRUE)
{
  m = nrow(A)
  n = ncol(A)
  if (m < n)
    stop("only implemented for m>n")
  
  nc = min(n, k)
  
  kl = 2*k # double the lanczos iterations
  
  H = square_up(A)
  lz = lanczos(H, k=kl)
  T_kl = tridiagonal(lz$alpha, lz$beta)
  
  ev = eigen(T_kl, symmetric=TRUE, only.values=only.values)
  d = ev$values[1:nc]
  
  if (!only.values)
  {
    UV = sqrt(2) * (lz$q %*% ev$vectors)
    u = UV[1:m, 1:nc]
    v = UV[(m+1):(m+n), 1:nc]
  }
  else
  {
    u = NULL
    v = NULL
  }
  
  list(d=d, u=u, v=v)
}
```

And here's a simple demonstration

```r

set.seed(1234)
m = 300
n = 10
x = matrix(rnorm(m*n), m, n)

k = 3
lz = lanczos_svd(x, k=k, FALSE)
sv = svd(x)

round(lz$d, 5)
```

    [1] 18.43443 15.83699  0.00160

```r
round(sv$d[1:k], 5)
```

    [1] 19.69018 18.65107 18.41093

```r
firstfew = 1:4
round(lz$v[firstfew, ], 5)
```

            [,1]    [,2]     [,3]
    [1,] 0.40748 0.02210 -0.00228
    [2,] 0.21097 0.18573 -0.00051
    [3,] 0.13179 0.01563 -0.00411
    [4,] 0.39096 0.27893 -0.00117

```r
round(sv$v[firstfew, 1:k], 5)
```
            [,1]     [,2]     [,3]
    [1,] 0.44829 -0.28028 -0.06481
    [2,] 0.18420  0.00095  0.67200
    [3,] 0.07991  0.01054 -0.14656
    [4,] 0.16961 -0.27779 -0.37683


[^anla]: Demmel, James W. *[Applied numerical linear algebra](https://books.google.com/books?hl=en&lr=&id=P3bPAgAAQBAJ&oi=fnd&pg=PR9&dq=applied+numerical+linear+algebra+demmel&ots=I7OxKaWh-y&sig=QKRZPGe0SiuBstzxYCg4j35gctE#v=onepage&q=applied%20numerical%20linear%20algebra%20demmel&f=false)*. Vol. 56. Siam, 1997.
[^gvl]: Golub, Gene H., and Charles F. Van Loan. *Matrix computations*. Vol. 3. JHU press, 2012.
[^lsl]: Caramanis and Sanghavi, *[Large scale learning: lecture 12](http://users.ece.utexas.edu/~sanghavi/courses/scribed_notes/Lecture_13_and_14_Scribe_Notes.pdf)*. ([archive.org link](https://web.archive.org/web/20171215080524/http://users.ece.utexas.edu/~sanghavi/courses/scribed_notes/Lecture_13_and_14_Scribe_Notes.pdf))
[^nla]: Trefethen, Lloyd N., and David Bau III. *[Numerical linear algebra](https://www.cs.cmu.edu/afs/cs/academic/class/15859n-f16/Handouts/TrefethenBau/LanczosIteration-36.pdf)*. Vol. 50. Siam, 1997. 
