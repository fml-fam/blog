---
title: "Introducing fml - the Fused Matrix Library"
date: 2020-06-21
---


## What is fml?

[fml](https://github.com/fml-fam/fml) is the Fused Matrix Library, a [permissively licensed](https://opensource.org/licenses/BSL-1.0), header-only C++ library for dense matrix computing. The emphasis is on real-valued matrix types (`float`, `double`, and `__half`) for numerical operations useful for data analysis.

The library provides 3 main classes: `cpumat`, `gpumat`, and `mpimat`. There is currently an experimental `parmat` class, but it is at present not well developed. For the main classes:

* CPU: Single node cpu computing (multi-threaded if using multi-threaded BLAS and linking with OpenMP).
* GPU: Single gpu computing.
* MPI: Multi-node computing via ScaLAPACK (+gpus if using [SLATE](http://icl.utk.edu/slate/)).

There are some differences in how objects of any particular type are constructed. But the high level APIs are largely the same between the objects. The goal is to be able to quickly create laptop-scale prototypes that are then easily converted into large scale gpu/multi-node/multi-gpu/multi-node+multi-gpu codes.



### Example

Here's a quick example computing the singular values of a simple CPU matrix:

```c++
#include <fml/cpu.hh>

int main()
{
  fml::cpumat<float> x(3, 2);
  x.fill_linspace(1, 6);
  
  x.info();
  x.print(0);
  
  fml::cpuvec<float> s;
  fml::linalg::svd(x, s);
  
  s.info();
  s.print();
  
  return 0;
}
```

If we save this in `svd.cpp`, we can compile it with something like:

```
g++ -I/path/to/fml/src -fopenmp svd.cpp -o svd -llapack -lblas
```

And then running it, we see:

```
# cpumat 3x2 type=f
1 4 
2 5 
3 6 

# cpuvec 2 type=f
9.5080 0.7729 
```

With only minor modifications, we can do this same operation on a GPU:

```c++
#include <fml/gpu.hh>

int main()
{
  auto c = fml::new_card(0);
  c.info();
  
  fml::gpumat<float> x(c, 3, 2);
  x.fill_linspace(1, 6);
  
  x.info();
  x.print(0);
  
  fml::gpuvec<float> s(c);
  fml::linalg::svd(x, s);
  
  s.info();
  s.print();
  
  return 0;
}
```

If we save this in `svd.cu`, we can compile it with something like:

```
nvcc -arch=sm_61 -Xcompiler "-I/path/to/fml/src" svd.cu -o svd -lcudart -lcublas -lcusolver -lcurand -lnvidia-ml
```

And then running it, we see:

```
## GPU 0 (GeForce GTX 1070 Ti) 860/8116 MB - CUDA 10.2

# gpumat 3x2 type=f 
1 4 
2 5 
3 6 

# gpuvec 2 type=f 
9.5080 0.7729 
```



## The Interface

I think that one person's high-level is another person's low-level. With that in mind, the goal of fml is to be "medium-level". It's high-level compared to working directly with e.g. the BLAS or cuBLAS, but low(er)-level compared to most other C++ matrix frameworks and things like R and Matlab.

If you want to learn more about the interface, you can read the [doxygen documentation](https://fml-fam.github.io/fml/html/index.html). There are also some [basic examples](https://github.com/fml-fam/fml/tree/master/examples).

I also maintain R bindings in the [fmlr](https://github.com/fml-fam/fmlr) package. That may be a good place to start. I sort of think of it like an interactive REPL for fml with some minor modifications (e.g., replace `x.method()` with `x$method()`). The package is [well-documented](https://fml-fam.github.io/fmlr/html/index.html), and contains some long-form articles. I will have much more to say about fmlr in a later post.



## Philosophy and Similar Projects

There are many high quality C/C++ matrix libraries. Of course [Armadillo](http://arma.sourceforge.net/) and [Eigen](http://eigen.tuxfamily.org/) are worth mentioning. [Boost](http://www.boost.org/) also has matrix bindings. [GSL](https://www.gnu.org/software/gsl/) is a C library with numerous matrix operations. There is also [PETSc](https://www.mcs.anl.gov/petsc/), which can do some distributed operations, although it is mostly focused on sparse matrices.

With the exception of GSL, all of these are permissively licensed. Some have at least some level of GPU support, for example Armadillo and Eigen (perhaps others). 

But I believe it is fair to say that none of these projects satisfies all of

* Permissively licensed.
* Single interface for CPU, GPU, and MPI backends (and multiple fundamental types).
* All functions support all backends.
* Only do the least amount of work necessary (e.g., QR factorization returns the compact QR and separate functions can compute Q and/or R).

That's essentially the goal of fml. I believe this offers something legitimately new in what is already a crowded space of high-quality libraries.



## State of the Project

The project is still pretty young, and at present only maintained/contributed to by one person, and only in my spare time. That said, it is currently usable and has quite a lot to offer if you operate at the level of linear algebra.

What's there now:

* Most linear algebra you would want, including
    - Basic operations like matrix sums, products
    - Helper operations like transpose and inversion
    - Factorizations like SVD and QR
* [NVIDIA® CUDA®](https://developer.nvidia.com/cuda-zone) support for GPU computing
* First-class [R bindings](https://github.com/fml-fam/fmlr) developed in parallel with the core library

What's coming soon:

* More statistics/data operations
* [AMD ROCm](https://www.amd.com/en/graphics/servers-solutions-rocm) and/or hip support for GPU computing

What I would like to have some day:

* More statistics/data operations than you would know what to do with
* Support for [MAGMA](https://icl.cs.utk.edu/magma/) for GPU matrix factorizations
* A more fleshed-out `parmat`
* Python bindings
* At least some basic in-library I/O
