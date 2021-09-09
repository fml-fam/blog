---
title: "A Bit About Checkpoint/Restart"
date: 2021-09-09
tags:
  - r
---

Checkpoint/Restart (C/R) is a fault tolerance strategy common in high performance computing, but as far as I can tell, almost totally unknown to statistical computing. The basic idea is that you save some/all of the state from your running program in order to be able to resume it in the event that the computation is interrupted.

Maybe it's storming outside and your power is spotty. Even if you have a laptop or a power supply, how long will that downed tree knock out your power? Perhaps a more typical problem is that you are working on a shared computing resource, like a campus cluster or even a large HPC machine. These kinds of resources typically operate with fixed wall clock run windows. Maybe that window is 24 hours, and you aren't sure that your job will finish in that time. In that case, what do you do? You can potentially go parallel (multicore and/or multi-node) or otherwise make your job run faster. Or maybe you don't feel like doing that for some reason. I'm not here to judge you, friend.

In cases like this, C/R is a good strategy to make sure that your job actually completes some day. But before we get into the details, let's focus our attention on a specific kind of computation. Because if your task runs in 5 seconds, what exactly are you worried about? So probably this is something that is at least potentially long running. And based on my experience supporting users with big statistical compute problems, the majority of these are task-based. So imagine you have many "small" operations you want to run, rather than one single "large" operation (what exactly these things mean is a bit ambiguous -- just bear with me).

Because my target audience is statistical computing folks, I will be giving examples in R. But obviously these concepts apply to every language. For running many-task things in R, you could use `lapply()` or its many clones. If you want to go parallel, there's the built-in parallel package, which is great. If you want some kind of different interface, I like the [future package](https://cran.r-project.org/web/packages/future/index.html). If you want to go distributed, I think the [pbdMPI package](https://cran.r-project.org/web/packages/pbdMPI/index.html) (of which I am a co-author) offers some unique advantages. And there are about 100 other such packages, which all have pros and cons, but all do similar things. And although I will mention some packages I have written at the tail of this article, the point here is not to shill my software. My hope is that more packages implement something like this.

Ok, so we're going to be saving the state of some kind of computation. We can serialize with R's `save()` and `load()`. You may be able to make the I/O faster with some of the recent, boutique packages for serializing R data. And for some applications, this may be very important. But this approach is dependency-free, which I feel is an advantage which often goes underappreciated in the R community.

There are some caveats worth discussing where this strategy ranges from difficult to impossible to implement. Again, we're focusing for the moment on task parallelism (because that's the easy problem). But let's say you have some really long running, single function evaluation - like some really big matrix operation, fitting a linear model, etc. In this case there may not be much you can do unless you wrote the thing yourself. Some encapsulated compiled code that you call can't really be intercepted mid-evaluation. You may be able to save/load intermittent parts of your overall pipeline, but that's much more ad hoc than what I want to talk about here.

Another caveat is that some package do custom memory allocation, which R calls "external pointers". R does not actually understand these objects; it merely pretends to. Any package that uses these -- for example, the [fmlr package](https://github.com/fml-fam/fmlr), but there are many others -- can't be used for C/R.

We're finally ready to talk about implementing a C/R strategy for a large, many-task problem. The problem naturally splits into a few distinct pieces:

**Step 0: Notation**

We will mimic the notation of `lapply()`:

* `X` is our data, some sort of list/vector object
* `FUN` is the function we want to apply to the data
* `...` are the additional arguments to `FUN`

Some other values we will use:

* `n` is the number of items (`length(X)`)
* `checkpoint_file` is the file we will save our checkpoint to
* `checkpoint_freq` is the frequency of checkpoint writing

If we have `checkpoint_freq=1` then every time a function evaluation completes, we write out to disk. With `checkpoint_freq=2`, then we write out every other time. This is to balance the cost of computation vs the cost of I/O. This will be somewhat application dependent.

**Step 1: Initialize or Restart**

Let's denote `start` as the index of the first element yet to be evaluated. If we have not yet started computing, then `start=1`. If we have evaluated 10 items, then `start=11`.

At the beginning of the workflow, we need to see if we are restarting or if we are just starting:

```r
if (file.exists(checkpoint_file))
  load(file=checkpoint_file)
else
{
  start = 1L
  ret = vector(length=n, mode="list")
}
```

The first block merely loads the checkpoint file if we are restarting. The second allocates space for the return object and initializes the index.

**Step 2: Evaluate**

Now for the real work. We need to iterate through the indices applying the function and checkpointing as necessary:

```r
for (i in start:n)
{
  ret[[i]] = FUN(X[i], ...)
  
  if (n %% checkpoint_freq == 0)
  {
    start = i+1L
    save(start, ret, file=checkpoint_file)
  }
}
```

If you understand the R language and have followed so far, I feel like this is fairly straightforward. If you feel like "this isn't really that complicated", then I agree and you probably get it.

As a final wrapup, you can remove the checkpoint file:

```r
file.remove(checkpoint_file)
```

The net effect of all of this is that `ret` contains the output that it would if you had run `ret = lapply(X, FUN)`. There is some minor overhead from bookkeeping. There is potentially major overhead from writing out the checkpoint depending on what is actually being stored in `ret`. Using a faster serialization method may be helpful, as mentioned before. Another option would be trying to overlap the compute and the I/O using something like `parallel::mcparallel()` and `parallel::mccollect()`. Note that this is potentially dangerous.

All of the above is encapsulated in the [crlapply package](https://github.com/wrathematics/crlapply). The package is available on the [HPCRAN](https://hpcran.org/) and can be installed via

```r
install.packages("crlapply", repos="https://hpcran.org")
```

Pilfering the example from the package README, we can see how this actually works in practice. We start with an "expensive" function. Here all it does is sleep for a bit before returning a square root:

```r
costly = function(x, waittime)
{
  Sys.sleep(waittime)
  print(paste("iteration:", x))
  
  sqrt(x)
}
```

Using the C/R version of `lapply()` available in the crlapply package, we can evaluate this like so:

```r
ret = crlapply::crlapply(1:10, costly, checkpoint_file="/tmp/cr.rdata", waittime=0.5)

unlist(ret)
```

Say we save this in the file `example.r`. We can run this in batch and kill it a few times (the printed `^C` represents Ctrl+C which kills the process).

```bash
$ Rscript example.r
[1] "iteration: 1"
[1] "iteration: 2"
[1] "iteration: 3"
[1] "iteration: 4"
^C
$ Rscript example.r
[1] "iteration: 5"
[1] "iteration: 6"
[1] "iteration: 7"
^C
$ Rscript example.r
[1] "iteration: 8"
[1] "iteration: 9"
[1] "iteration: 10"
```

Notice that indeed, each time we restart the script, it picks up right where it left off. The final line of the script, when executed, will produce the following:

```
unlist(ret)
##  [1] 1.000000 1.414214 1.732051 2.000000 2.236068 2.449490 2.645751
##  [8] 2.828427 3.000000 3.162278
```

So what about parallelism? I have gotten a suprising amount of mileage out of the [tasktools package](https://github.com/RBigData/tasktools), which uses MPI to solve this problem. The package is available on the [HPCRAN](https://hpcran.org/) and assuming you have a system installation of MPI available, it can be installed via

```r
install.packages("tasktools", repos=c("https://hpcran.org", "https://cran.rstudio.com"))
```

It is meant to be used in batch with [SPMD style programming](https://en.wikipedia.org/wiki/SPMD). I don't really want to get into what this means right now, but I may in a future post. My hope is that others will begin utilizing these strategies in their various wrappers around `lapply()` and `parallel::mclapply()`.
