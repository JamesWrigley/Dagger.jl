# Use Case: Parallel Nested Loops

One of the many applications of Dagger is that it can be used as a drop-in
replacement for nested multi-threaded loops that would otherwise be written
with `Threads.@threads`.

Consider a simplified scenario where you want to calculate the maximum mean
values of random samples of various lengths that have been generated by several
distributions provided by the Distributions.jl package. The results should be
collected into a DataFrame. We have the following function:

```julia
using Dagger, Random, Distributions, StatsBase, DataFrames

function f(dist, len, reps, σ)
    v = Vector{Float64}(undef, len) # avoiding allocations
    maximum(mean(rand!(dist, v)) for _ in 1:reps)/σ
end
```

Let us consider the following probability distributions for numerical
experiments, all of which have expected values equal to zero, and the following
lengths of vectors:

```julia
dists =  [Cosine, Epanechnikov, Laplace, Logistic, Normal, NormalCanon, PGeneralizedGaussian, SkewNormal, SkewedExponentialPower, SymTriangularDist]
lens = [10, 20, 50, 100, 200, 500]
```

Using `Threads.@threads` those experiments could be parallelized as:

```julia
function experiments_threads(dists, lens, K=1000)
    res = DataFrame()
    lck = ReentrantLock()
    Threads.@threads for T in dists
        dist = T()
        σ = std(dist)
        for L in lens
            z = f(dist, L, K, σ)
            Threads.lock(lck) do
                push!(res, (;T, σ, L, z))
            end
        end
    end
    res
end
```

Note that `DataFrames.push!` is not a thread safe operation and hence we need
to utilize a locking mechanism in order to avoid two threads appending the
DataFrame at the same time.

The same code could be rewritten in Dagger as:

```julia
function experiments_dagger(dists, lens, K=1000)
    res = DataFrame()
    @sync for T in dists
        dist = T()
        σ = Dagger.@spawn std(dist)
        for L in lens
            z = Dagger.@spawn f(dist, L, K, σ)
            push!(res, (;T, σ, L, z))
        end
    end
    res.z = fetch.(res.z)
    res.σ = fetch.(res.σ)
    res
end
```

In this code we have job interdependence. Firstly, we are calculating the
standard deviation `σ` and than we are using that value in the function `f`.
Since `Dagger.@spawn` yields an `EagerThunk` rather than actual values, we need
to use the `fetch` function to obtain those values. In this example, the value
fetching is perfomed once all computations are completed (note that `@sync`
preceding the loop forces the loop to wait for all jobs to complete). Also,
note that contrary to the previous example, we do not need to implement locking
as we are just pushing the `EagerThunk` results of `Dagger.@spawn` serially
into the DataFrame (which is fast since `Dagger.@spawn` doesn't block).

The above use case scenario has been tested by running `julia -t 8` (or with
`JULIA_NUM_THREADS=8` as environment variable). The `Threads.@threads` code
takes 1.8 seconds to run, while the Dagger code, which is also one line
shorter, runs around 0.9 seconds, resulting in a 2x speedup.