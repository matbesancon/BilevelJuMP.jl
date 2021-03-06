# BilevelJuMP.jl

A bilevel optimization extension of the [JuMP](https://github.com/JuMP-dev/JuMP.jl) package.

| **Build Status** |
|:----------------:|
| [![Build Status][build-img]][build-url] [![Codecov branch][codecov-img]][codecov-url] |


[build-img]: https://travis-ci.org/joaquimg/BilevelJuMP.jl.svg?branch=master
[build-url]: https://travis-ci.org/joaquimg/BilevelJuMP.jl
[codecov-img]: http://codecov.io/github/joaquimg/BilevelJuMP.jl/coverage.svg?branch=master
[codecov-url]: http://codecov.io/github/joaquimg/BilevelJuMP.jl?branch=master

## Introduction

BilevelJuMP is a package for modeling and solving bilevel optimization problems in Julia. As an extension of the JuMP package, BilevelJuMP allows users to employ the usual JuMP syntax with minor modifications to describe the problem and query solutions.

BilevelJuMP is built on top of [MathOptInterface](https://github.com/JuMP-dev/MathOptInterface.jl) and makes strong use of its features to reformulate the problem as a single level problem and solve it with available MIP, NLP, and other solvers.

The currently available methods are based on re-writing the problem using the KKT conditions of the lower level. For that we make strong use of [Dualization.jl](https://github.com/JuMP-dev/Dualization.jl)

## Example

```julia
using JuMP, BilevelJuMP, Cbc

model = BilevelModel(Cbc.Optimizer, mode = BilevelJuMP.SOS1Mode())

@variable(Lower(model), x)
@variable(Upper(model), y)

@objective(Upper(model), Min, 3x + y)
@constraints(Upper(model), begin
    x <= 5
    y <= 8
    y >= 0
end)

@objective(Lower(model), Min, -x)
@constraints(Lower(model), begin
     x +  y <= 8
    4x +  y >= 8
    2x +  y <= 13
    2x - 7y <= 0
end)

optimize!(model)

objective_value(model) # = 3 * (3.5 * 8/15) + 8/15 # = 6.13...
value(x) # = 3.5 * 8/15 # = 1.86...
value(y) # = 8/15 # = 0.53...
```

The option `BilevelJuMP.SOS1Mode()` indicates that the solution method used
will be a KKT reformulation emplying SOS1 to model complementarity constraints
and solve the problem with MIP solvers (Cbc, Xpress, Gurobi, CPLEX, SCIP).

Alternatively, the option `BilevelJuMP.IndicatorMode()` is almost equivalent to
the previous. The main difference is that it relies on Indicator constraints
instead. This kind of constraints is available in some MIP solvers.

A third and classic option it the `BilevelJuMP.FortunyAmatMcCarlMode()`, which
relies on the Fortuny-Amat and McCarl big-M method that requires a MIP solver
with very basic functionality, i.e., just binary variables are needed.
The main drawback of this method is that one must provide bounds for all primal
and dual variables. However, if the bounds are good, this method can be more
efficient than the previous. Bound hints to compute the big-Ms can be passed
with the methods: `set_primal_(upper\lower)_bound_hint(variable, bound)`, for primals;
and `set_dual_(upper\lower)_bound(constraint, bound)` for duals.
We can also call `FortunyAmatMcCarlMode(primal_big_M = vp, dual_big_M = vd)`,
where `vp` and `vd` are, repspectively, the big M fallback values for primal
and dual variables, these are used when some variables have no given bounds,
otherwise the given bounds are used instead.

Another option is `BilevelJuMP.ProductMode()` that reformulates the
complementarity constraints as products so that the problem can be solved by
NLP (Ipopt, KNITRO) solvers or even MIP solvers with the aid of binary
expansions
(see [QuadraticToBinary.jl](https://github.com/joaquimg/QuadraticToBinary.jl)).
Note that binary expansions require varibles to have upper and lower bounds.


## Advanced Features

### Lower level dual variables

Suppose you have a constraint `b` in the lower level:

```julia
@constraint(Lower(model), b, ...)
```

It is possible to access the dual variable of `b` to use it in the upper level:

```julia
@variable(Upper(model), lambda, DualOf(b))
```

### Conic lower level

BilevelJuMP allows users to write conic models in the lower level. However,
solving this kind of problems is much harder and requires complex solution
methods. Mosek's Conic MIP can be used with the aid of
[QuadraticToBinary.jl](https://github.com/joaquimg/QuadraticToBinary.jl).