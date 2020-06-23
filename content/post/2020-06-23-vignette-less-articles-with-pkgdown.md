---
title: "Vignette-less Articles With pkgdown"
date: 2020-06-23
tags:
  - r
---


## Long Boring Background You Can Skip

I don't like the R vignette system. At its core, it's a good idea. Having long-form package documentation is a good thing, and having the ability to put in source code listing that get automatically executed during rebuilds is great. But the way it works in practice is, to me, extremely annoying. Whenever you build/test your package, the vignettes will automatically be rebuilt, even if that's not what you want. The underlying assumptions here are that:

* rebuilding vignettes is cheap
* vignette code can run on the machine that is building/checking the package

Both of these assumptions regularly fail for me, and likely anyone else who writes HPC packages for R. Imagine you're benchmarking something that takes tens of minutes or more. You probably don't want to re-run those very often. You might only even want to run them on a completely different system, say one with more RAM or CPU cores, or even one with a fancy GPU that your dev box doesn't have.

I also don't like that if you want to have html documents, you need to list a very fairly package, [knitr](https://cran.r-project.org/web/packages/knitr/index.html), in your package DESCRIPTION's Suggests field. On Linux (the only platform I run on), this is doubly annoying because everything has to be built from source. You can of course install a package without suggested packages by modifying the `dependencies` argument of `install.packages()`. But even as annoyed as I get by a giant wall of suggested packages needlessly cluttering my R library, I usually don't bother because by the time I realize the problem, it's already downloading and installing them.

But html or pdf, if you want to re-build your vignettes infrequently (having qasi-static vignettes), there are a few workarounds I'm aware of. One obvious way is to simply not have automatically executed code in your vignettes. I have done this in many packages, and if I'm being honest, there is probably some dead/non-executing code lying around in those vignettes somewhere. If you restrict yourself to pdf outputs, you can also make a "shell" vignette that "includes" your static vignette. But I find in practice that this can be cumbersome, and even then, you're stuck with a pdf.



## Using pkgdown

I've recently started using the [pkgdown package](https://pkgdown.r-lib.org/) to generate html files for the manpage-style help that R packages have. But it can also handle vignettes in its "article" system. This gives you the ability to have vignettes without even having to bother with R's vignette system.

Although the setup is a bit tedious, I find this approach fairly inoffensive, and the results make me very happy. It goes something like this:

1. Add the line `vignettes/` to your package `.Rbuildignore` file.
2. Set up your vignettes.
    - Put the source Rmd files in, say, `vignettes/src/`.
    - Manually build them using `rmarkdown::render()` with outputs in `vignettes/`, and with output format `md_document()` and file extension `.Rmd`.
3. Build the pkgdown "articles" (vignettes).

This gives you complete control over when your documents are rebuilt. I use exactly this workflow in the [fmlr package](https://github.com/fml-fam/fmlr). You can see the generated documentation [here](https://fml-fam.github.io/fmlr/html/index.html).

I will have more to say about fmlr in a later post. For now, I'll walk through these pkgdown vignette steps in more depth.



## 1. Ignoring the Vignettes

The whole point of this is to not use R's vignette system. So you need to make sure your `.Rbuildignore` file contains a line like:

```
vignettes/
```

so that when you `R CMD build` your package, it won't include any of the vignettes. Your articles still have to go in the `vignettes/` folder, because that's where pkgdown will look for them, and I have no idea how to override this if you wanted to put them somewhere else.

The generated vignette files (output of `rmarkdown::render()`) will go in `vignettes/`. The raw/source vignettes that contain yet-to-be-executed source code listings should go somewhere else. I use `vignettes/src/`.



## 2. Setting Up Your Vignette Builder

You need some kind of script that will call `rmarkdown::render()` on your source Rmd files. It can be a simple Makefile or a more complicated R script. The format to pass to `rmarkdown::render()` should look something like this:

```r
fmt = rmarkdown::md_document(
  variant="gfm",
  preserve_yaml=TRUE,
  ext=".Rmd"
)
```

The goal here is to have an Rmd output that has executed the source code listings of your input file. These generated Rmd files are pretending to be the source vignettes for pkgdown.

Here's the builder script that I use:

```r
#!/usr/bin/env Rscript
library(rmarkdown)

rmf = function(f)
{
  if (file.exists(f))
    file.remove(f)
}

clean = function()
{
  files = dir(pattern="*.Rmd", recursive=FALSE)
  for (f in files)
    rmf(f)
}

set_path = function()
{
  while (!file.exists("DESCRIPTION"))
  {
    setwd("..")
    if (getwd() == "/home")
      stop("couldn't find package!")
  }
  
  setwd("vignettes")
}

build_vignette = function(f)
{
  f_Rmd = basename(f)
  of = sub(f_Rmd, pattern="^_", replacement="")
  rmf(of)
  
  fmt = rmarkdown::md_document(
    variant="gfm",
    preserve_yaml=TRUE,
    ext=".Rmd"
  )
  
  rmarkdown::render(
    f,
    output_file=of,
    output_dir=getwd(),
    output_format=fmt
  )
  
  invisible(TRUE)
}



# -----------------------------------------------------------------------
set_path()

clean()
build_vignette("./src/_source_vignette_1.Rmd")
build_vignette("./src/_source_vignette_2.Rmd")
```

I preface my raw source files with an underscore so that they're immediately visually distinct to me in my editor. If you want to do something else, you may need to slightly modify the logic a bit. Also, because I may not want to rebuild every vignette every time, I just comment out `clean()` or `build_vignette()` calls as necessary for whatever I'm doing. You could easily come up with something more sophisticated, but this works fine for me.



## 3. Building the pkgdown Articles

Here's a sample Makefile for pkgdown. I like to build my documentation in `docs/html/`, so I put the Makefile in `docs/`. I also like to serve the documentation on GitHub pages (there's an automatic `docs/` option in the repo settings), so I have an `index.html` re-directer in `docs/` as well. Set your override arguments appropriately if you do something different.

```
all: docs

docs:
	( Rscript -e "pkgdown::build_site('..', override=list(destination='docs/html'), install=FALSE)" )

articles: vignettes
	( Rscript -e "pkgdown::build_articles('..', override=list(destination='docs/html'))" )

vignettes:
	( cd ../vignettes && Rscript ./rebuild.r )

clean:
	rm -rf html
```

Now all you have to do is (re-)build your vignettes with `make vignettes`, and `make docs` to re-generate the pkgdown html files. If you are making a lot of changes to the vignettes and only want to generate those (skipping the manpage documentation, which can take a while to generate), you can just do `make articles`.
