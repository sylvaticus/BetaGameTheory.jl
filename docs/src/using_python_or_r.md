# [Using StrategicGames.jl from other programming languages](@id using_other_languages)

In this section we learn how to use `StrategicGames` in Python or R is almost as simple as to use a native library, with objects converted automatically between Julia and Python, and hence no specific package wrappers are needed. For Python we will show two separate "Julia from Python" interfaces, [PyJulia](https://github.com/JuliaPy/pyjulia) and [JuliaCall](https://github.com/cjdoris/PythonCall.jl) with the second one being the most recent one, while for R we will show the [JuliaCall](https://github.com/Non-Contradiction/JuliaCall) R package (no relations with the homonymous Python package).

### Use StrategicGames in Python

#### With the classical `pyjulia` package

[PyJulia](https://github.com/JuliaPy/pyjulia) is a relativelly old method to use Julia code and libraries in Python. It works great but it requires that you already have a Julia working installation on your PC, so we need first to download and install the Julia binaries for our operating system from [JuliaLang.org](https://julialang.org/). Be sure that Julia is working by opening the Julia terminal and e.g. typing `println("hello world")`

Install `PyJulia` with: 

```
$ python3 -m pip install --user julia   # the name of the package in `pip` is `julia`, not `PyJulia`
```

We can now open a Python terminal and, to obtain an interface to Julia, just run:

```python
>>> import julia
>>> julia.install() # Only once to set-up in julia the julia packages required by PyJulia
>>> jl = julia.Julia(compiled_modules=False)
```
If we have multiple Julia versions, we can specify the one to use in Python passing `julia="/path/to/julia/binary/executable"` (e.g. `julia = "/home/myUser/lib/julia-1.8.0/bin/julia"`) to the `install()` function.

The `compiled_module=False` in the Julia constructor is a workaround to the common situation when the Python interpreter is statically linked to `libpython`, but it will slow down the interactive experience, as it will disable Julia packages pre-compilation, and every time we will use a module for the first time, this will need to be compiled first.
Other, more efficient but also more complicate, workarounds are given in the package documentation, under the https://pyjulia.readthedocs.io/en/stable/troubleshooting.html[Troubleshooting section].

Let's now add to Julia the StrategicGames package. We can surely do it from within Julia, but we can also do it while remaining in Python:

```python
>>> jl.eval('using Pkg; Pkg.add("StrategicGames")') # Only once to install StrategicGames
```

While `jl.eval('some Julia code')` evaluates any arbitrary Julia code (see below), most of the time we can use Julia in a more direct way. Let's start by importing the StrategicGames Julia package as a submodule of the Python Julia module:

```python
>>> from julia import StrategicGames
>>> jl.seval("using StrategicGames")
```

As you can see, it is no different than importing any other Python module.

For the data, let's load the payoff matrix "Python side" using Numpy:

```python
>>> import numpy as np
>>> payoff = np.array([[[-1,-1],[-3,0]],[[0,-3],[-2,-2]]]) # prisoner's dilemma
```

We can now call StrategicGames functions as we would do for any other Python library functions. In particular, we can pass to the functions (and retrieve) complex data types without worrying too much about the conversion between Python and Julia types, as these are converted automatically:

```python
>>> eq = StrategicGames.nash_cp(payoff)
>>> # Note that array indexing in Julia start at 1
>>> eq
<PyCall.jlwrap (status = MathOptInterface.LOCALLY_SOLVED, equilibrium_strategies = [[0.0, 0.9999999887780999], [0.0, 0.9999999887780999]], expected_payoffs = [-1.9999999807790678, -1.9999999807790678])
>>> StrategicGames.is_nash(payoff,eq.equilibrium_strategies)
True
>>> StrategicGames.dominated_strategies(payoff)
[array([1], dtype=int64), array([1], dtype=int64)]
```

Note: If we are using the `jl.eval()` interface, the objects we use must be already known to julia. To pass objects from Python to Julia, import the julia `Main` module (the root module in julia) and assign the needed variables, e.g.

```python
>>> X_python = [1,2,3,2,4]
>>> from julia import Main
>>> Main.X_julia = X_python
>>> jl.eval('sum(X_julia)')
12
```

Another alternative is to "eval" only the function name and pass the (python) objects in the function call:

```python
>>> jl.eval('sum')(X_python)
12
```

#### With the newer `JuliaCall` python package

[JuliaCall](https://github.com/cjdoris/PythonCall.jl) is a newer way to use Julia in Python that doesn't require separate installation of Julia.

Istall it in Python using `pip` as well:

```
$ python3 -m pip install --user juliacall
```

We can now open a Python terminal and, to obtain an interface to Julia, just run:

```python
>>> from juliacall import Main as jl
```
If you have `julia` on PATH, it will use that version, otherwise it will automatically download and install a private version for `JuliaCall`

If we have multiple Julia versions, we can specify the one to use in Python passing `julia="/path/to/julia/binary/executable"` (e.g. `julia = "/home/myUser/lib/julia-1.8.0/bin/julia"`) to the `install()` function.

To add `StrategicGames` to the JuliaCall private version we evaluate the julia package manager `add` function:

```python
>>> jl.seval('using Pkg; Pkg.add("StrategicGames")') # Only once to install StrategicGames
```

As with `PyJulia` we can evaluate arbitrary Julia code either using `jl.seval('some Julia code')` and by direct call, but let's first import `StrategicGames`:

```python
>>> jl.seval("using StrategicGames")
>>> sg = jl.StrategicGames
```

For the data, we reuse the `payoff` Numpy arrays we created earlier.

We can now call StrategicGames functions as we would do for any other Python library functions. In particular, we can pass to the functions (and retrieve) complex data types without worrying too much about the conversion between Python and Julia types, as these are converted automatically:


```python
>>> eq = sg.nash_cp(payoff)
>>> eq._jl_display() # force a "Julian" way of displaing of Julia objects
(status = MathOptInterface.LOCALLY_SOLVED, equilibrium_strategies = [[0.0, 0.9999999887780999], [0.0, 0.9999999887780999]], expected_payoffs = [-1.9999999807790678, -1.9999999807790678])
>>> sg.is_nash(payoff,eq.equilibrium_strategies)
True
>>> sg.dominated_strategies(payoff)
<jl [[1], [1]]>

```

Note: If we are using the `jl.eval()` interface, the objects we use must be already known to julia. To pass objects from Python to Julia, we can write a small Julia _macro_:

```python
>>> X_python = [1,2,3,2,4]
>>> jlstore = jl.seval("(k, v) -> (@eval $(Symbol(k)) = $v; return)")
>>> jlstore("X_julia",X_python)
>>> jl.seval("sum(X_julia)")
12
```

Another alternative is to "eval" only the function name and pass the (python) objects in the function call:

```python
>>> X_python = [1,2,3,2,4]
>>> jl.seval('sum')(X_python)
12
```

#### Conclusions about using StrategicGames in Python

Using either the direct call or the `eval` function, wheter in `Pyjulia` or `JuliaCall`, we should be able to use all the StrategicGames functionalities directly from Python. If you run into problems using StrategicGames from Python, [open an issue](https://github.com/sylvaticus/StrategicGames.jl/issues/new) specifying your set-up.


### Use StrategicGames in R (TODO)

To use `StrategicGames` in R we start by installing the [`JuliaCall`]() R package:

```{r}
> install.packages("JuliaCall")
> library(JuliaCall)
> julia_setup(installJulia = FALSE) # use installJulia = FALSE to let R download and install a private copy of julia
```

Note that, differently than `PyJulia`, the "setup" function needs to be called every time we start a new R section, not just when we install the `JuliaCall` package.
If we don't have `julia` in the path of our system, or if we have multiple versions and we want to specify the one to work with, we can pass the `JULIA_HOME = "/path/to/julia/binary/executable/directory"` (e.g. `JULIA_HOME = "/home/myUser/lib/julia-1.1.0/bin"`) parameter to the `julia_setup` call. Or just let `JuliaCall` automatically download and install a private copy of julia.

`JuliaCall` depends for some things (like object conversion between Julia and R) from the Julia `RCall` package. If we don't already have it installed in Julia, it will try to install it automatically.

As in Python, let's start from the data loaded from R and do some work with them in Julia:

```{r}
> library(datasets)
> X     <- as.matrix(sapply(iris[,1:4], as.numeric))
> y     <- sapply(iris[,5], as.integer)
> xsize <- dim(X)
```

Let's install StrategicGames. As we did in Python, we can install a Julia package from Julia itself or from within R:

```{r}
> julia_eval('using Pkg; Pkg.add("StrategicGames")')
```

We can now "import" the StrategicGames julia package (in julia a "Package" is basically a module plus some metadata that facilitate its discovery and integration with other packages, like the reuired set) and call its functions with the `julia_call("juliaFunction",args)` R function:

```{r}
> julia_eval("using StrategicGames")
> shuffled <- julia_call("consistent_shuffle",list(X,y))
> Xs       <- matrix(sapply(shuffled[1],as.numeric), nrow=xsize[1])
> ys       <- as.vector(sapply(shuffled[2], as.integer))
> m        <- julia_eval('KMeansClusterer(n_classes=3)')
> yhat     <- julia_call("fit_ex",m,Xs)
> acc      <- julia_call("accuracy",yhat,ys,ignorelabels=TRUE)
> acc
[1] 0.8933333
```

As alternative, we can embed Julia code directly in R using the `julia_eval()` function:

```{r}
kMeansR  <- julia_eval('
  function accFromKmeans(x,k,y)
    m    = KMeansClusterer(n_classes=Int(k))
    yhat = fit!(m,x)
    acc  = accuracy(yhat,y,ignorelabels=true)
    return acc
  end
')
```

We can then call the above function in R in one of the following three ways:
1. `kMeansR(Xs,3,ys)`
2. `julia_assign("Xs_julia", Xs); julia_assign("ys_julia", ys); julia_eval("accFromKmeans(Xs_julia,3,ys_julia)")`
3. `julia_call("accFromKmeans",Xs,3,ys)`

While other "convenience" functions are provided by the package, using  `julia_call`, or `julia_assign` followed by `julia_eval`, should suffix to use `StrategicGames` from R. If you run into problems using StrategicGames from R, [open an issue](https://github.com/sylvaticus/StrategicGames.jl/issues/new) specifying your set-up.

## [Dealing with stochasticity and reproducibility](@id dealing_with_stochasticity)

Machine Learning workflows include stochastic components in several steps: in the data sampling, in the model initialisation and often in the models's own algorithms (and sometimes also in the prediction step).
All StrategicGames models with a stochastic components support a `rng` parameter, standing for _Random Number Generator_. A RNG is a "machine" that streams a flow of random numbers. The flow itself however is deterministically determined for each "seed" (an integer number) that the RNG has been told to use.
Normally this seed changes at each running of the script/model, so that stochastic models are indeed stochastic and their output differs at each run.

If we want to obtain reproductible results we can fix the seed at the very beginning of our model with `Random.seed!([AnInteger])`. Now our model or script will pick up a specific flow of random numbers, but this flow will always be the same, so that its results will always be the same.

However the default Julia RNG guarantee to provide the same flow of random numbers, conditional to the seed, only within minor versions of Julia. If we want to "guarantee" reproducibility of the results with different versions of Julia, or "fix" only some parts of our script, we can call the individual functions passing [`FIXEDRNG`](@ref), an instance of `StableRNG(FIXEDSEED)` provided by `StrategicGames`, to the `rng` parameter. Use it with:


- `MyModel(;rng=FIXEDRNG)`               : always produce the same sequence of results on each run of the script ("pulling" from the same rng object on different calls)
- `MyModel(;rng=StableRNG(SOMEINTEGER))` : always produce the same result (new identical rng object on each call)

This is very convenient expecially during model development, as a model that use `(...,rng=StableRNG(an_integer))` will provides stochastic results that are isolated (i.e. they don't depend from the consumption of the random stream from other parts of the model).

In particular, use `rng=StableRNG(FIXEDSEED)` or `rng=copy(FIXEDRNG)` with [`FIXEDSEED`](@ref)  to retrieve the exact output as in the documentation or in the unit tests.

Most of the stochasticity appears in _training_ a model. However in few cases (e.g. decision trees with missing values) some stochasticity appears also in _predicting_ new data using a trained model. In such cases the model doesn't restrict the random seed, so that you can choose at _predict_ time to use a fixed or a variable random seed.

Finally, if you plan to use multiple threads and want to provide the same stochastic output independent to the number of threads used, have a look at [`generate_parallel_rngs`](@ref).

"Reproducible stochasticity" is only one of the elements needed for a reproductible output. The other two are (a) the inputs the workflow uses and (b) the code that is evaluated.
Concerning the second point Julia has a very modern package system that guarantee reproducible code evaluation (with a few exception linked to using external libraries, but StrategicGames models are all implemented in Julia itself). Without going in detail, you can use a pattern like this at the beginning of your machine learning workflows:

```        
using Pkg  
cd(@__DIR__)            
Pkg.activate(".")  # Activate a "local" environment, specific to this folder
Pkg.instantiate()  # Download and install the required packages if not already available 
```

This will tell Julia to load the exact version of dependent packages, and recursively of their dependencies, from a `Manifest.toml` file that is automatically created in the script's folder, and automatically updated, when you add or update a package in your workflow.
Note that these locals "environments" are very "cheap" (packages are not actually copied to each environment on your system, only referenced) and the environment doen't need to be in the same script folder as in this example, can be any folder you want to "activate".

## Saving and loading trained models

Trained models can be saved on disk using the [`model_save`](@ref) function, and retrieved with [`model_load`](@ref).
The advantage over the serialization functionality in Julia core is that the two functions are actually wrappers around equivalent [JLD2](https://juliaio.github.io/JLD2.jl/stable/) package functions, and should maintain compatibility across different Julia versions. 
