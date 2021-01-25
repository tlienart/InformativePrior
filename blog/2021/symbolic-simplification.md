+++
title = "Symbolic Simplication"
hascode = true
date = Date(2021,1,25)
rss = "A short description of the page which would serve as blurb in a RSS feed; you can use basic markdown here but the whole description string must be a single line (not a multiline string). Like this one for instance. Keep in mind that styling is minimal in RSS so for instance don't expect maths or fancy styling to work; images should be ok though:"
+++
@def tags=["julia","probprog"]

# Symbolic Simplication
_January 25, 2021_

_Upcoming features in [Soss.jl](https://github.com/cscherrer/Soss.jl) include static model simplification. After a one-time compilation cost, posterior log-densities for many models become "constant cost", independent of the number of observations. Bayesian analysis for such models can easily scale to big data._

_The symbolic representation of the posterior log-density can also be useful for pedagogical purposes._


\toc

## First, a Model

Let's start with something simple, say a linear regression with 10,000 observations.

```julia:model
using Soss

N = 10000

m = @model x, λ begin
    σ ~ Exponential(λ)
    β ~ Normal(0,1) 
    y ~ For(1:N) do j
        Normal(x[j] * β, σ)
    end
    return y
end
```

Soss models are _generative_, so we can use one to generate data. This is useful to help make sure the model assumptions are reasonable (fake data should look similar to real data), and it's also handy for little examples like this.

To generate data from `m`, we need to have values for the _input arguments_ `x` and `λ`:

```julia:ex4
x = randn(N)
λ = 1.0
```

Given this, we could just call `rand(m(x=x,λ=λ))`. But this would "forget" the values of `σ` and `β`. To instead keep track of these, we can do

```julia:ex5
trace = simulate(m(x=x, λ=1.0)).trace
y = trace.y
```

```julia:ex6
using Statistics #hide
@show trace.σ #hide
@show trace.β #hide
@show mean(y) #hide
@show std(y)  #hide
```

As a result, we now have
     
\show{ex6}

## Evaluating the Log-Density (The Punchline)

In Bayesian modeling, we're usually in a situation of having observed `y`, and wanting to infer what we can about latent parameters like `σ` and `β`. So let's construct the posterior distribution. We could write this as `post = m(x=x, λ=λ) | (y=y,)`, or using the shorthand

```julia:ex7
post = m(; x, λ) | (; y)
```

To evaluate this, say on the `trace` from the original sample, we just call

```julia:ex8
logdensity(post, trace)
```

which in this case gives 

\show{ex8}

Let's see how fast it is.

```julia:ex9
using BenchmarkTools
slow = @benchmark $logdensity($post, $trace)
```

\show{ex9}

Ok, clearly some things can be improved, for example we should be able to do this without allocation.

But rather than dwell too much on that, let's jump to the punchline:

```julia:ex10
ℓ = symlogdensity(post)
f = codegen(post; ℓ=ℓ)
fast = @benchmark $f($(;x,λ), $(;y), $trace)
```

\show{ex10}


```julia:ex11
speedup = round(Int,minimum(slow).time / minimum(fast).time) #hide
print("**$speedup× speedup**") #hide
```

Note the units; we started out in milliseconds, but now we're in nanoseconds. A quick `minimum(slow).time / minimum(fast).time` shows we've gotten a \textoutput{ex11}.

## How it Works

`ℓ` is a symbolic expression from [`SymbolicUtils.jl`](https://juliasymbolics.github.io/SymbolicUtils.jl/). To build it, we first generate a trace, as we did above. Now we have all of the types, so we can evaluate the log-density again, this time replacing the values with symbolic variables of the appropriate type.

At this point, we have a symbolic expression, and we also have a dictionary of fixed values from the arguments and observed values. So we expand the expression and then walk the result, looking for subexpressions that evaluate to scalars with no free variables. When we find one, we use the dictionary to rewrite it.

Next, we do [common subexpression elimination](https://en.wikipedia.org/wiki/Common_subexpression_elimination), which just makes sure we don't evaluate the same thing twice.

```julia:ex12
Soss.cse(ℓ)
```

\show{ex12}

It's a very short step to get from that to a Julia `Expr` that does what we want:

```julia:ex13
sourceCodegen(post)
```

\show{ex13}

Finally we add some code to tell us where to get the variables, and use [`GeneralizedGenerated.jl`](https://github.com/JuliaStaging/GeneralizedGenerated.jl) to make it callable:

```julia:ex14
codegen(post; ℓ=ℓ)
```

\show{ex14}

## Code Generation

```julia:ex12
ℓλ = symlogdensity(post; noinline=(:λ,))
```

\show{ex12}



```julia
fast = @benchmark $f($(;x,λ), $(;y), $trace)
```


```julia
@time dynamicHMC(post; ℓ = ℓ)
```