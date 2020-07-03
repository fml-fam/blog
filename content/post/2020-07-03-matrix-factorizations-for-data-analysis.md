---
title: "Matrix Factorizations for Data Analysis"
date: 2020-07-03
tags:
  - math
  - r
---

Integers can be factored into products of special kinds of integers with useful properties called primes. Similarly, matrices can be factored into products of special kinds of matrices with useful properties.

Because data analysis often involves operating on numeric matrices, understanding how and why to factor matrices can be very helpful. We're going to talk about the major factorizations, namely LU, Cholesky, QR, SVD, and Eigendecomposition. There are others, but if you master these, then you're off to a great start.

Throughout we're going to assume that all matrices are real-valued, because it makes the math slightly more pleasant most of the time, and because statisticians would start flipping over tables if you gave them matrices with complex numbers. Most of the core ideas are the same; only the small details change for the most part --- replace transposes with hermitians and so on. If you work with complex numbers, you know the drill.

This is written for the person who knows what a matrix is, has done some data analysis, and wants to know more about the math and computation behind many of the core operations of the field.


## LU

Anyone who took a matrix algebra course in university has seen the [LU decomposition](https://en.wikipedia.org/wiki/LU_decomposition) for square matrices, where you factor

$$ A = LU $$

Here, $L$ is a lower-triangular matrix and $U$ is an upper triangular matrix (hence the naming - lower/upper). This factorization is unique with some modest assumptions. If $A$ is invertible, then there is a unique unit-diagonal (which is a fancy way of saying the diagonal is all 1's) $L$ and corresponding unique $U$.

This factorization is the basis of [the LINPACK benchmark](https://en.wikipedia.org/wiki/LINPACK_benchmarks#HPLinpack), which is used to rank the peak performance of [the Top 500 supercomputers](https://top500.org/). Specifically, the goal is to perform an LU decomposition with "partial pivoting" (row permutations only). It looks like

$$ A = PLU $$

where $P$ is a [permutation matrix](https://en.wikipedia.org/wiki/Permutation_matrix). The $PLU$ variant is more versatile, as there are many more matrices you can factor as $PLU$ than $LU$. But everyone just calls this LU, and the subtlety isn't worth spending much time thinking about.

LU is useful for things like solving square systems of equations and computing matrix inverses. This is why R uses `solve(x, y)` for solving a system of equations and `solve(x)` for inverting a matrix. It's not a good interface choice, but there is a kind of logic to it. LU can also be used to calculate the determinant, which R's `det()` and `determinant()` functions also use.

Solving a square system of equations is a pretty rare task in data analysis, but inverting things is common enough. Often times an analyst needs the elements of the inverse of a covariance matrix, for example. Although in this case, the matrix is probably symmetric (i.e. the upper and lower triangles are the same) depending on what you mean by "a covariance matrix". So you would actually be better off using a Cholesky factorization instead (more on that next section).

It's also worth noting that you usually don't actually need to compute the matrix inverse, but the *action* of the inverse. As noted above, statisticians sometimes legitimately need the elements of the inverse of a correlation matrix or whatever. But more often you need to compute something like $A^{-1} B$ rather than just $A^{-1}$ by itself. Yes, there is a difference. In this case, you would be better off doing something like factoring $A=LU$, then using [forward substitution followed by back substitution](https://en.wikipedia.org/wiki/Triangular_matrix#Forward_and_back_substitution) with $L$ and $U$, respectively, against $B$. Numerically, this is not the same as actually constructing $A^{-1}$ and then multiplying against $B$. The output may be more accurate if $A$ is ill-conditioned, and it will probably be faster.

You may be wondering why we've talked so much about matrix inversion and haven't mentioned something like Gauss-Jordan elimination. That's because this is not actually useful numerically for matrix inversion. Your math professor wasted your time by making you compute a bunch of those problems by hand.

Your favorite matrix software probably uses the [LAPACK](https://performance.netlib.org/lapack/) API for matrix factorizations. There are lots of implementations of the LAPACK interface, including free ones like [OpenBLAS](https://www.openblas.net/) and proprietary ones like [Intel's MKL](https://software.intel.com/content/www/us/en/develop/tools/math-kernel-library.html). There are also quasi-clones like [NVIDIA's cuSOLVER](https://docs.nvidia.com/cuda/cusolver/index.html), which has a subset of LAPACK functionality for GPUs. There is also a distributed version called [ScaLAPACK](https://netlib.org/scalapack/). Then there are modern replacements, like [MAGMA](https://icl.cs.utk.edu/magma/) and [SLATE](https://icl.utk.edu/slate/). Currently, [fml](https://github.com/fml-fam/fml) supports all of these except for MAGMA, which is on the to-do list.

All of these interfaces follow LAPACK conventions. If your matrix software doesn't, it's because they're hiding those conventions from you --- they're still using LAPACK or something that looks a lot like it. So throughout, I'm going to reference the LAPACK function (without leading type) 

Anyway, to factor $A=LU$ in LAPACK, you would use the `getrf()` function. This uses the convention that $L$ is unit-diagonal, which is actually important, but more on that in a bit. Once you've factored your matrix, you can call `getrs()` to solve a system of equations, or `getri()` to invert the matrix. Or, once factored you could make two calls to `trsm()` to use the action of the inverse.

So why does it matter that the $L$ is unit-diagonal in LAPACK? A common trick in LAPACK and its derivatives is to store factorizations compactly. If you factor $A = PLU$ explicitly, then you need to triple the storage of the input. In practice, the `getrf()` routines will replace $A$ by a unit-diagonal $L$ in the lower triangle of the input (not actually storing the diagonal because you know what it is), and the corresponding $U$ in the upper triangle plus diagonal part of the input, with the $P$ information living in a vector of swaps. Why store a bunch of zeros if you don't have to?

Pretty much all of the LAPACK matrix factorizations will do things like this in one way or another. At the end of the day, your input matrix will be modified, either to store the return or as a kind of temporary workspace when the original data is no longer needed. Your favorite LAPACK bindings (Matlab, R, Armadillo, Julia, numpy, ...) will hide this from you by operating on a copy of the input. This is why operating at the level of LAPACK directly instead of a high-level interface can have massive performance and memory advantages, particularly for iterative algorithms. This is a big motivating reason why we try to stay closer to LAPACK style conventions in [fml](https://github.com/fml-fam/fml).



## Cholesky

The Cholesky decomposition is a powerful but often overlooked matrix factorization. But first, and most importantly, how do you even say it? My understanding is that this is properly pronounced "ko-less-kee", although I think most Anglophones substitute "cho" for "ko". You'd have to ask a native French speaker if you wanted to know for sure.

That out of the way, we can think of the Cholesky decomposition as a factorization of $A = LU$ where $L$ and $U$ are lower/upper triangular, and $L^T = U$. Remember that we could factor many square matrices as $A = PLU$. However, here we additionally require that the matrix be symmetric (its transpose is itself again) and positive definite (eigenvalues are all positive). More on these assumptions in a bit.

But why even bother with this when we already have the more general LU? For the cases where Cholesky applies, in software it is generally faster to deal with. One of the reasons it's faster is that you know for sure it does not have to pivot, while the LU might (and that's not cheap). Memory consumption should basically be the same, although slightly less for Cholesky since there is no vector of pivots to store.

Usually the matrices in the Cholesky factorization are referred to as the left and right Cholesky factors. So they are usually denoted as $L$ and $R$. Because of this and the transpose connection between the two factors, you can basically take your pick of:

$$
A = LR = LL^T = R^TR
$$

Because the factorization takes a matrix and re-writes it as a product of essentially two copies of the same matrix (one merely transposed), this factorization is often thought of as computing the square root of a matrix. A square root of a number is another number such that when you square it you get the original number back. Similar idea here, but for matrices.

There are weirdos who deal with actual matrix square roots, where you factor $A = BB$ (no transpose). But I'm not aware of any real use for this kind of thing. And there are other "functions on matrices" that you can do, like the [matrix exponential](https://en.wikipedia.org/wiki/Matrix_exponential). While lots of fun to talk about, this isn't really related to matrix factorizations.

Let's return for a moment to the assumptions, namely that the matrix is symmetric positive definite. It's generally pretty easy to tell if a matrix is symmetric; positive definite is a bit harder, especially when you're probably only interested in Cholesky for its low compute cost (and eigenvalues are expensive). However, for the applications that statisticians care about, you almost always get both for free.

If you want to calculate the left/right Cholesky factor of a covariance matrix, then you're in luck! It must be symmetric because if $\Sigma = A^TA$ (I know there's more to computing a covariance matrix but just go with it), then $\sigma_{ij} = A_i \cdot A_j$ for any $i, j$, where $A_i$ is the $i$'th column vector of $A$. And of course the dot product is commutative. To see that it is positive definite intuitively, we're squaring the matrix $A$ to compute $\Sigma$. So we're squaring the eigenvalues, so we can take square roots, so they must be non-negative. Now that's not a proof, but it's correct intuition. We also might get into trouble if one is zero (positive semi-definite), but that's not worth worrying about right now.

As noted several times, Cholesky is pretty quick, assuming your matrix meets the requirements. You can compute a Cholesky factor with LAPACK's `potrf()` function. This will give you either the left or right factor depending on which triangle of the input you told it to use. By convention, R's `chol()` will always return the right factor; in [fml](https://github.com/fml-fam/fml) we use the left. Once factored, you can use `potri()` to compute the inverse, or `potrs()` to solve a system of equations.



## QR

A very useful, but perhaps under-appreciated factorization is the [QR factorization](https://en.wikipedia.org/wiki/QR_decomposition). Let's assume for the moment that $A$ is $m\times n$ with $m \geq n$. Then we can factor

$$
A_{m\times n} = Q_{m\times n} R_{n\times n}
$$

where $R$ is upper triangular, and $Q$ is [orthogonal](https://en.wikipedia.org/wiki/Orthogonal_matrix).

To say that a matrix is orthogonal is the same as saying that its transpose is its inverse. For a rectangular matrix, this only works along the short dimension. So if $Q$ is orthogonal of dimension $m\times n$ with $m\geq n$ then $Q^TQ = I_{n \times n}$ but $QQ^T$ will not be $I_{m\times m}$ (because the rows of $Q$ can not be linearly independent in this case). But if $m\leq n$ then $Q^TQ = QQ^T = I_{n\times n}$.

If $m < n$ then you still could factor $A=QR$, but the dimensions work out a little differently. And frankly, I don't think there's ever a good reason to do that. Instead you would use the LQ factorization

$$
A_{m\times n} = L_{m\times m} Q_{m\times n}
$$

where $L$ is lower triangular and $Q$ is again orthogonal. This time, $QQ^T = I_{m\times m}$.

Mathematically, LQ is just QR on a transposed matrix, because $\left(QR\right)^T = R^TQ^T$. But on a computer, you don't want to have to store the matrix transpose if $A$ is large. Most people who say "QR" really mean "QR or LQ, whichever is cheaper", without bothering to explicitly state it. Sadly, this means that poor old LQ doesn't always get the attention it deserves. In fact, R considers LQ so uninteresting that they don't even have LQ analogues of its QR functions. This annoys me more than you might expect. I mean come on, it's the 21st century. They can put a man on the moon, but you can't compute an LQ without a transpose? Really now.

So right away, this is in a different league than the previous two factorizations. First, we can apply it to rectangular matrices --- you know, the important ones? And while in principle you could use it to solve a square system of equations or even invert a matrix, there's not really a good reason to do that. You already have LU and Cholesky, and QR will be slower for those applications.

So what can you use it for? One very useful application of QR is linear regression. You can build a linear model fitter fairly directly using QR. For generalized linear models, you can use [iteratively reweighted least squares](https://en.wikipedia.org/wiki/Iteratively_reweighted_least_squares) with the linear model fitter as the core driver.

I've gotten into borderline fist fights with statisticians when I say that "linear regression *is* linear least squares". Disagreement usually lives in whether or not you mean "ANOVA" when you say "linear regression". Anyway I'm right and they're wrong, and in linear least square you have a data matrix $X$ and response vector $y$ and you want to find a parameter vector $\beta$ that minimizes

$$
\lVert X_{m\times n}\beta_{n\times 1} - y_{m\times 1} \lVert
$$

If $X$ is full rank, then this has analytical solution

$$
\beta = \left( X^T X \right)^{-1} X^T y
$$

Often $X$ is not full rank, and in that case your cute little stats101 trick won't work. It's also a bad way to compute this in floating point arithmetic regardless of the column rank of the input; but to really get into that I need to talk about condition numbers, and I will do that in a later post. For now it's worth pointing out that any statistical software written by anyone who remotely knows what they're doing won't implement linear regression this way.

So what *is* the correct way to do it? Note that $\lVert Q \lVert = \lVert Q^T \lVert = 1$ and $\lVert AB \lVert \leq \lVert A \lVert \lVert B \lVert$, so

$$
\begin{align*}
\lVert X\beta - y \lVert &= \lVert QR\beta - y \lVert \\\\
  &= \lVert Q^TQR\beta - Q^T y \lVert \\\\
  &= \lVert R\beta - Q^T y \lVert
\end{align*}
$$

This is equivalent to solving the system

$$
R\beta = Q^T y
$$

Because $R$ is triangular, this can be done quickly and accurately using triangular solvers, like LAPACK's `trtrs()` functions. If your system is under-determined (i.e. $m<n$), you can play a similar game with LQ, but the math is a little more annoying.

Interestingly, R uses a modified version of the QR routine in the ancient predecessor to LAPACK called [LINPACK](https://www.netlib.org/linpack/) --- no, not the benchmark. They use this because QR solvers will make choices that can destroy information useful in ANOVA if you started with a rank degenerate model matrix.

For the sake of numerical stability, the LAPACK QR driver will choose the column with largest partial norm for pivoting while forming the Householder reflections. However, this can re-order the columns so that a statistical interpretation
in an ANOVA is destroyed. The modifications in R's QR driver (used in `lm()` and `lm.fit()`) implements a "limited column pivoting" strategy. Here, when a column is detected to have "small" partial norm, it is pushed to the back of the columns. The author of the modifications (who I believe to be Ross Ihaka) wrote this in the comments:

>a limited column pivoting strategy based on the 2-norms of the reduced columns moves columns with near-zero norm to the right-hand edge of the x matrix. this strategy means that sequential one degree-of-freedom effects can be computed in a natural way.

>i am very nervous about modifying linpack code in this way. if you are a computational linear algebra guru and you really understand how to solve this problem please feel free to suggest improvements to this code.

For what it's worth, I've spoken to several members of the LAPACK team and they seem to believe that this strategy is essentially fine numerically. Outside of R's linear model fitter, I modified the ScaLAPACK linear model fitter to support the limited pivoting strategy in [the pbdBASE package](https://github.com/RBigData/pbdBASE). I suspect basically every quality statistical software does something like this, although I do not know for sure.

LAPACK has a least squares driver `gels()` that will automatically use QR or LQ to find the least squares solution. If you want to do a QR directly, there are a few options; some pivot, some don't. Pivoting is necessary if your matrix is (or might be if you don't know) rank degenerate. If it does no pivoting, then it is basically assuming that the matrix has full column rank. The pivoting ones can be used as so-called "rank-revealing" QR. If pivoting, you would probably want `geqp3()`, and if not you would use `geqrf()`. Replace `qr` with `lq` for the LQ variants.

Like with LU, the QR factorization from LAPACK is stored compactly, replacing the input matrix. However, the way it is stored is quite a bit more complicated than LU. For QR, the upper triangle plus diagonal will be the $R$ matrix. Obviously the lower triangle can't be $Q$, because it's not a lower triangular matrix. Instead, you would use the additional function `ormqr()` to operate on the action of $Q$ or its transpose. If you really want the elements of $Q$, you can use `orgqr()` which is in-place, or use `ormqr()` against a diagonal matrix of all 1's (a "rectangular identity matrix"). The same things all hold true for LQ.



## Singular Value Decomposition

Probably the most important and powerful of the factorizations is the [singular value decomposition](https://en.wikipedia.org/wiki/Singular_value_decomposition), aka SVD. 

If $A$ is an $m\times n$ matrix then

$$
A_{m\times n} = U_{m\times k} \Sigma_{k\times k} V_{n\times k}^T
$$

where $k$ is usually the minimum of $m$ and $n$ (although it may be taken to be the rank of $A$ which is no greater than the minimum of $m$ and $n$), $\Sigma$ is a diagonal matrix, and $U$ and $V$ are orthogonal. This is sometimes called the "compact SVD". Although it's basically never done in software, we could take the "full SVD" as

$$
A_{m\times n} = U_{m\times m} \Sigma_{m\times n} V^T_{n\times n}
$$

When I said SVD is the most powerful factorization, I wasn't joking. You can do ***anything*** with an SVD. Matrix inverse? Easy

$$
A^{-1} = V \Sigma^{-1} U^T
$$

Solve a system of equations? Don't make me laugh

$$
Ax=b \iff x = V \Sigma^{-1} U^Tb
$$

Determinant? Are you even trying?

$$
det(A) = det(U \Sigma V^T) = det(\Sigma) = \displaystyle\prod_{i=1}^n \sigma_{ii}
$$

Linear regression? HAVE YOU EVEN BEEN PAYING ATTENTION?

$$
y = X\beta \iff \beta = V \Sigma^{-1} U^T y
$$

Condition number? SVD.

Column rank? SVD.

PCA ***is*** SVD.

SVD is *love*.

SVD is *life*.

***ALL HAIL SVD***.

Frankly I'm not convinced there's anything you can do with a matrix that can't be improved with an SVD. Almost all matrix math problems get easier when you take the SVD. Mathematically, it is strictly better than QR. Computationally, it is more expensive to compute and needs more memory.

SVD being so obviously important, a lot of study and care has been devoted to calculating this factorization. The core technique for most solvers uses something called the [QR algorithm](https://en.wikipedia.org/wiki/QR_algorithm). You may be surprised to learn that this involves a QR factorization, although it's a bit more involved than I want to get into.

LAPACK has several routines for computing singular values/vectors. Probably the most commonly used are `gesvd()` and `gesdd()`. The first uses the QR algorithm, while the second uses a divide-and-conquer algorithm. If you are only computing singular values, they should be the same. For computing singular vectors, the divide-and-conquer approach should be much faster. LAPACK also includes `gesvdj()`, which uses Jacobi rotations to compute the SVD. Personally I never see any reason to use these over `gesdd()`, but I encourage you to explore for yourself.

All of these functions will compute all singular values and/or vectors (for the compact SVD), even if you just need a few. In R's `La.svd()` and `svd()` bindings, you can request a subset of singular vectors. But actually, your choices are none of them or all of them (you can compute only the singular values without any singular vectors though). So if you only want one, it will compute all of them and then throw away what you didn't want.

If you only want a few singular vectors (e.g. making a plot from the first few principal components), you may be interested in a "truncated SVD", not to be confused with "compact SVD". There are some common techniques for approximating truncated SVD. There is the [Lanczos algorithm](https://en.wikipedia.org/wiki/Lanczos_algorithm), which I wrote about extensively [in a previous post](/blog/2020/06/15/svd-via-lanczos-iteration/). Then there's my personal favorite method, that uses [random projections](https://arxiv.org/abs/0909.4061).

Sometimes, especially in algorithmic explanations, SVD will only be described for square matrices. There's a fun little trick you can do to reduce a rectangular matrix to a square one for computing SVD. Let's say $A_{m\times n}$ is a rectangular matrix with $m>n$, and you know how to compute the SVD of a square matrix. Then we can factor $A=QR$ first, and take the SVD of the square matrix $R$. It goes something like this:

$$
\begin{align*}
A &= QR \\\\
  &= Q \left( U_R \Sigma_R V_R^T \right) \\\\
  &= \left( Q U_R \right) \Sigma_R V_R^T
\end{align*}
$$

Then $U_A = Q U_R$ because a product of orthogonal matrices is orthogonal, and $\Sigma_A = \Sigma_R$ and $V_A = V_R$. The LAPACK routines discussed above use this strategy.



## Eigendecomposition

Given a square, symmetric matrix $A$ of order $n$, its eigenvalue decomposition (aka "eigendecomposition", aka "spectral decomposition") is

$$
A = V \Lambda V^T
$$

where $V$ is an orthogonal matrix and $\Lambda$ is diagonal, each of order $n$. Both are guaranteed to be real-valued since $A$ is symmetric. You can discuss eigendecompositions for non-symmetric matrices, but both the eigenvalues (diagonals of the $\Lambda$ matrix) and eigenvectors are no longer guaranteed to be real-valued, even if $A$ is. The reason why is explored in the fundamental theorem of algebra.

In school, everyone learns about eigenvalues because they're important to engineers. For data analysis, they are largely unimportant. They can be used to compute the [trace](https://en.wikipedia.org/wiki/Trace_%28linear_algebra%29) (i.e., the sum of the diagonal) of a square matrix in the most computationally expensive way imaginable. They can also be used to compute the determinant, although it is generally much faster to use LU.

Probably the only reason anyone interested in data analysis would actually care about spectral decomposition is because of its interesting relationship to SVD. If $A$ is a rectangular matrix and $A = U\Sigma V^T$ then

$$
\begin{align*}
A^TA &= \left( U \Sigma V^T \right)^T \left( U \Sigma V^T \right) \\\\
  &= V \Sigma U^T U \Sigma V^T \\\\
  &= V \Sigma^2 V^T
\end{align*}
$$

So the eigenvalues of $A^TA$ are the square of the singular values of $A$, and the eigenvectors of $A^TA$ are the right singular vectors of $A$. If we know $\Sigma$ and $V$ then we can recover $U$ easily since $U = A V \Sigma^{-1}$.

Likewise, $AA^T = U \Sigma^2 U^T$.

This trick is how many big data frameworks will compute SVD. If $n$ is small in absolute terms, then you can distribute you matrix across multiple processes by rows. Once you compute $A^TA$ --- which will require communication --- all the remaining operations are local. For the product, you first compute the local product, then add up all of the local matrices. This is equivalent to an MPI allreduce operation. After that, the eigendecomposition is local, and you can use LAPACK or whatever.

The advantages to this strategy are that it can be very quick computationally, and it's very easy to implement. But there are some issues with this approach. For one, expressly computing $A^TA$ or $AA^T$ will square the [condition number](https://en.wikipedia.org/wiki/Condition_number) of $A$. I will have more to say about condition numbers in a later post, but let's just say for now that this is a bad thing. So if you are doing your computing in floating point and accuracy is important, this may not be a good strategy for computing singular values. It would be better to use something like a tall skinny QR algorithm (TSQR). I don't feel like getting into what that is right now, but trust me, it's a real thing.

The other major issue staring us in the face is, of course, how we handle the case where $n$ is large in absolute terms, or if $m$ and $n$ are "kinda large" but $A$ is nearly square. Then computing either $A^TA$ or $AA^T$ potentially consumes a lot of memory.

In LAPACK, you have quite a few choices for symmetric eigensolvers. The two I think are most immediately worth looking at are the relatively robust representations method implemented in `syevr()` and the divide-and-conquer method implemented in `syevd()`. Years ago, [I wrote about the differences](http://librestats.com/2016/10/28/comparing-symmetric-eigenvalue-performance/) between these two in yet another way too lengthy post. The tldr is that the divide-and-conquer method is more scalable for multicore but uses a TON of memory. 

There are other libraries like [EISPACK](https://www.netlib.org/eispack/) (har har har, get it?) and [ARPACK](https://www.caam.rice.edu/software/ARPACK/). But really all you need is in LAPACK, its clones like cuSOLVER, or its modern variants like [MAGMA](https://icl.cs.utk.edu/magma/).

Like with SVD, I'm sure there are lots of ways of computing truncated eigendecompositions, although I don't know anything about them because again, it's a mostly useless factorization for data analysis. If you only want one eigen value/vector, you can use [power iteration](https://en.wikipedia.org/wiki/Power_iteration), which is actually related to the matrix exponential; but I'm not getting into that now.



## Others

In my opinion, these make up the "core" matrix factorizations, particularly for data scientists. There are of course others, although many are what I would call less useful generalizations of the above (e.g. [LDU](https://en.wikipedia.org/wiki/LU_decomposition#LDU_decomposition) and [QZ](https://en.wikipedia.org/wiki/Schur_decomposition#Generalized_Schur_decomposition)). There are also some more specialized things like [non-negative matrix factorization](https://en.wikipedia.org/wiki/Non-negative_matrix_factorization). Also, apparently k-means clustering [can be formulated as a matrix factorization problem](https://arxiv.org/abs/1512.07548), although I think this is more interesting than it is useful.

If you studied math in school, you may well have spent ages talking about [the Jordan canonical form](https://en.wikipedia.org/wiki/Jordan_normal_form) without ever hearing about anything actually useful like SVD (I did). These kinds of formulations are interesting to mathematicians because they are powerful at proving yet more things that are equally useless. Best to ignore them.



## Exercises

Well this kind of got out of hand. I didn't mean to turn this into a book. But here we are, so I figure the least I can do is give you some problems.

Math problems

1. Let $A$ be a square matrix of order $n$ and let $A = PLU$.
    1. What is the contribution of $P$ to the determinant of $A$? Recall that $det(AB) = det(A)det(B)$.
    2. Describe a strategy to calculate its determinant using this factorization. You may assume without proof that $det(T) = \displaystyle\prod_{i=1}^n t_{ii}$ for a triangular matrix $T_{n\times n}$
2. Let $A$ be a square matrix of order $n$.
    1. If $A$ is positive definite, describe a strategy to calculate its determinant using its cholesky factorization.
    2. What if $A$ is positive semi-definite but not positive definite? (hint: see problem 7 part 1)
3. 
    1. Prove that a product of 2 orthogonal matrices is again orthogonal.
    2. Prove that if $Q$ is a square orthogonal matrix, its determinant must be 1 or -1 (hint: start with $Q^TQ = I$).
    3. If a square matrix has determinant 1, is it necessarily orthogonal?
4. Let $X$ be an $m\times n$ matrix with $m < n$ and let $y$ be an $m$-length vector. Use $X=LQ$ to solve the linear least square problem $\displaystyle\min_\beta \lVert X\beta - y \lVert$. Are there any subtleties here for implementing this in software?
5. Let $A$ be a square, symmetric matrix of order $n$, and let $A = U \Sigma V^T$ be its SVD.
    1. Show that $U = V$.
    2. Assume that each singular value $\sigma_i \geq 0$. Given this, find the matrix square root of $A$, that is, the matrix $B$ so that $BB = A$.
6. Let $A$ be a square matrix of order $n$. The polar decomposition is a matrix factorization where $A = UP$, where $U$ is orthogonal and $P$ is positive semi-definite, each of order $n$. Derive $U$ and $P$ via the SVD (hint: $U_\text{polar}$ is not $U_\text{svd}$).
7. Let $A$ be a square symmetric matrix of order $n$.
    1. Use its eigendecomposition to calculate $det(A)$.
    2. Using only its eigendecomposition, show that the trace of $A$ is equal to the sum of the eigenvalues (hint: first show that by combining the definitions of trace and matrix product, $tr(AB) = tr(BA)$; or use your fancy inner product math if you know that).

Programming problems

1. Implement a function that can solve a (square) system of equations $Ax=b$ by factoring $A=LU$, but without ever constructing $A^{-1}$. You may use your programming language's triangular system solvers (e.g., `backsolve()` and `forwardsolve()` in R).
2. Construct a symmetric matrix $A$ of order $n$ of random normal values.
    1. Compare the run time computing the inverse using a general matrix inverter (LU) vs using Cholesky. Try with various values of $n$.
    2. Compare the run time computing the determinant with an implementation that uses LU (e.g. R's `det()`) to one using Cholesky. Try with various values of $n$.
    3. When was Cholesky better than LU? What are the tradeoffs to each approach?
3. Let $X$ be an $m\times n$ matrix with $m>n$ and $y$ an $m$-length vector.
    1. Implement a function `lm_xtx()` that takes arguments $X$ and $y$ and computes the linear regression coefficients $\beta$ using the analytical solution.
    2. Implement a function `lm_qr()` that takes arguments $X$ and $y$ and computes the linear regression coefficients $\beta$ using the QR formulation.
    3. Let $X$ be the $7\times 7$ [Hilbert matrix](https://en.wikipedia.org/wiki/Hilbert_matrix), and let $y$ be $[1, 2, 3, 4, 5, 6, 7]^T$. Test your `lm_qr()` function first, then test `lm_xtx()`. Did anything interesting happen?
4. Implement a matrix square root function (use problem 5 above). If the input is an invalid matrix under our assumptions (non-square, non-symmetric, non- positive semi-definite), it should detect this and throw an error.
5. For each of the following, assume all matrix inputs are square and symmetric.
    1. Implement a function to compute the determinant using eigendecomposition. Compare the run time performance to an implementation that uses LU on a random matrix. Try many different matrix sizes.
    2. Implement a function to compute the trace using eigendecomposition. Compare the run time performance to simply summing the diagonal of the input. Try many different matrix sizes.

Challenge problems

1. (difficulty: ruthless shilling) Use [fml](https://github.com/fml-fam/fml) or [fmlr](https://github.com/fml-fam/fmlr) to experiment with the programming problems above.
2. (difficulty: fortran) Go learn about the strange naming scheme for LAPACK functions.
3. (difficulty: "we're, like, all the same, man") Compare the LU, Cholesky, QR, SVD, and Eigendecomposition API's across several of the mainstream languages (R, Matlab, Julia, python's numpy, ...).
4. (difficulty: impossible) Find something SVD isn't good for.
