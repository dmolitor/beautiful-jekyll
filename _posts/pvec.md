---
layout: post
title: Parallel Vectorization in R
subtitle: Excerpt from Soulshaping by Jeff Brown
cover-img:
thumbnail-img: 
share-img:
tags: [R, Parallel Computing]
---

Something you learn pretty early on with R is that vectorization of functions is valuable, both for code optimization and ease-of-use. It's often easier (at least for me)
to prototype a function for a single-element use case and then vectorize after the fact. The following will discuss the importance of combining vectorization and speed
(aka parallelization). For these examples we'll be using the `dplyr` package and `future.apply` (for parallelization).

## Why/How to vectorize ##

Say we define a function that adds two values together (adding in a pause to simulate a slow process).
```r
add_slow <- function(a, b) {
  Sys.sleep(1)
  return(sum(a, b))
}

# Quick example
add_slow(1, 2)
#> [1] 3
```
A natural extension of this function is to return the pairwise sum of two vectors.
```r
# What we want the result to be
c(3, 7)
#> [1] 3 7

# What the result actually is
add_slow(c(1, 3), c(2, 4))
#> [1] 10
```
As you can see, the output of `add_slow(c(1, 3), c(2, 4))` is the same as `sum(c(1, 2, 3, 4))` which is not what we want in this case. To acheive the desired result, we need to
vectorize `add_slow`. Thankfully, base R has a convenience function `Vectorize` for this that makes things very simple. To vectorize we simply do the following
```r
# base::Vectorize will solve this problem for us
vadd_slow <- Vectorize(FUN = add_slow, vectorize.args = c("a", "b"))

# Does it still do what we want?
vadd_slow(1, 2) # Yes
#> [1] 3

# Does it return the pairwise sums?
vadd_slow(c(1, 3), c(2, 4)) # Nice!
#> [1] 3 7
```

## Practical use-case of vectorization ##

Vectorization can clearly be helpful, and the following is a toy example that exemplifies how it can be applied in an analysis setting.
```r
library(dplyr)

# Let's see a real-ish example
dat <- tibble("col1" = 1:7, "col2" = 11:17)

# Trying to mutate with non-vectorized function
dat %>% mutate(col3 = add_slow(col1, col2)) # Not what we wanted
#> # A tibble: 7 x 3
#>    col1  col2  col3
#>   <int> <int> <int>
#> 1     1    11   126
#> 2     2    12   126
#> 3     3    13   126
#> 4     4    14   126
#> 5     5    15   126
#> 6     6    16   126
#> 7     7    17   126

# Vectorized version, however, works perfectly
dat %>% mutate(col3 = vadd_slow(col1, col2)) # But this is soooo slow 
#> # A tibble: 7 x 3
#>    col1  col2  col3
#>   <int> <int> <int>
#> 1     1    11    12
#> 2     2    12    14
#> 3     3    13    16
#> 4     4    14    18
#> 5     5    15    20
#> 6     6    16    22
#> 7     7    17    24
```
As we can see, vectorization solves our problem; but the function is very slow ☹️. However, if we want the best of both worlds, we can easily implement
a version of `base::Vectorize` that runs in parallel!

## How can we vectorize base::Vectorize ##

The first question here is how `base::Vectorize` acheives its vectorization behavior. To understand that let's take a look at the source code for the function, and you'll
see the following line near the bottom of the function
```r
# Take a look at base::Vectorize source code
base::Vectorize
#> ...
#>         do.call("mapply", c(FUN = FUN, args[dovec], MoreArgs = list(args[!dovec]), 
#>             SIMPLIFY = SIMPLIFY, USE.NAMES = USE.NAMES))
#> ...
```
Basically, it is taking your arguments as vectors, and it is using the `base::mapply` function to iterate `FUN` across each 'row' of your arguments. To see how this
works, let's look at the following examples.
```r
# Create two vectors like our example above
a <- c(1, 3)
b <- c(2, 4)

# Now map base::mapply across these vectors
mapply(sum, a, b)
#> [1] 3 7
```
As you can see, this acheives the behavior that applying the `Vectorize` function acheived. This is because `Vectorize` is basically a user-friendly wrapper
around `base::mapply`. Thankfully, the [future.apply](https://cran.r-project.org/web/packages/future.apply/vignettes/future.apply-1-overview.html) package has 
implemented parallel versions of all the base-R \*apply() functions, so all we need to do is replace the `base::mapply` call with `future.apply::future_mapply`!
```r
library(future.apply)

# Parallel version of base::vectorize
pvec <- function(FUN, 
                 vectorize.args, 
                 SIMPLIFY = TRUE, 
                 USE.NAMES = TRUE) {
  arg.names <- as.list(formals(FUN))
  arg.names[["..."]] <- NULL
  arg.names <- names(arg.names)
  vectorize.args <- as.character(vectorize.args)
  if (!length(vectorize.args))
    return(FUN)
  if (!all(vectorize.args %in% arg.names))
    stop("must specify names of formal arguments for 'vectorize'")
  collisions <- arg.names %in% c("FUN", "SIMPLIFY",
                                 "USE.NAMES", "vectorize.args")
  if (any(collisions))
    stop(sQuote("FUN"), " may not have argument(s) named ",
         paste(sQuote(arg.names[collisions]), collapse = ", "))
  FUNV <- function() {
    args <- lapply(as.list(match.call())[-1L], eval, parent.frame())
    names <- if (is.null(names(args)))
      character(length(args))
    else names(args)
    dovec <- names %in% vectorize.args
    do.call("future_mapply", # Now using `future.apply::future_mapply` as opposed to `base::mapply`
            c(FUN = FUN, 
              args[dovec], 
              MoreArgs = list(args[!dovec]),
              SIMPLIFY = SIMPLIFY, 
              USE.NAMES = USE.NAMES))
  }
  formals(FUNV) <- formals(FUN)
  FUNV
}
```
## Does this actually speed things up? ##

To answer that, let's revert back to the example we looked at above. First let's vectorize our `add_slow` function in parallel.
```r
# Make parallel vectorized version
pvadd_slow <- pvec(FUN = add_slow, vectorize.args = c("a", "b"))
```
Next, initialize our parallel session.
```
# Initialize a parallel session
# To learn more, go here: https://cran.r-project.org/web/packages/future/vignettes/future-1-overview.html
# My machine has 8 cores -- so using 8 background sessions in parallel
plan(multisession)
```
Now, let's compare the time it takes our non-parallel vectorized function to compute the output compared to the parallel vectorized function
```r
# How fast is the first one approximately?
start_time <- Sys.time()
dat %>% mutate(col3 = vadd_slow(col1, col2))
#> # A tibble: 7 x 3
#>    col1  col2  col3
#>   <int> <int> <int>
#> 1     1    11    12
#> 2     2    12    14
#> 3     3    13    16
#> 4     4    14    18
#> 5     5    15    20
#> 6     6    16    22
#> 7     7    17    24
Sys.time() - start_time
#> Time difference of 7.07175 secs

# How fast is the parallel one approximately?
start_time <- Sys.time()
dat %>% mutate(col3 = pvadd_slow(col1, col2))
#> # A tibble: 7 x 3
#>    col1  col2  col3
#>   <int> <int> <int>
#> 1     1    11    12
#> 2     2    12    14
#> 3     3    13    16
#> 4     4    14    18
#> 5     5    15    20
#> 6     6    16    22
#> 7     7    17    24
Sys.time() - start_time
#> Time difference of 1.410486 secs
```
The speed up was almost seven-fold! Although these same results can be acheived many different ways, this is a quick and easy way to acheive optimization both through
the act of vectorization itself, and by doing it in parallel.

