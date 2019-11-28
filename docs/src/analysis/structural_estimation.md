# Structural Identification and Estimation

Structural estimation is the process of automatically finding the differential
equations from a given set of data. It differs from parameter estimation which
is finding parameter values in a preset model. These techniques do not require
giving as input a known model. Instead, these methods take in data and return
the differential equation model which generated the data.

## Installation

This functionality does not come standard with DifferentialEquations.jl.
To use this functionality, you must install DataDrivenDiffEq.jl:

```julia
]add DataDrivenDiffEq
using DataDrivenDiffEq
```

## Definition of the Methods

There are various different avenues in which structural estimation can occur.
However, the main branches are: do you want to know the equations in a human
understandable manner, or is it sufficient to have a function that predicts the
derivative and generates correct timeseries? We will refer to methods which
return symbolic forms of the differential equation as *structural identification*,
while those which return functions only for prediction as *structural estimation*.

### Basis

Almost all methods require setting some form of a basis on observables or
functional forms. A `Basis` is generated via:

```julia
Basis(h, u, parameters = [])
```

where `h` is a vector of ModelingToolkit `Operation`s for the valid functional
forms, `u` are the ModelingToolkit `Variable`s used to describe the Basis, and
`parameters` are the optional ModelingToolkit `Variable`s used to describe the
parameters in the basis elements.

### Structural Identification Methods

#### SInDy

`SInDy` is the [method for generating sparse sets of equations](https://www.pnas.org/content/113/15/3932)
from a chosen basis. The function call is:

```julia
SInDy(data, dx, basis, ϵ = 1e-10)
```

where `data` is a matrix of observed values (each column is a timepoint, each
row is an observable), `dx` is the matrix of derivatives of the observables,
`basis` is a `Basis`, `ϵ` is a cutoff tolerance for exclusion from the final
sparsified equations. This function will return the coefficients for the equations
in the chosen basis, with zeros denoting forms that do not exist in the final
equations.

### Structural Estimation Methods

#### (Exact) Dynamic Mode Decomposition (DMD)

Dynamic Mode Decomposition, or Exact Dynamic Mode Decomposition, is a method for
generating an approximating linear differential equation (the Koopman operator)
from the observed data.

To construct the approximation, use:

```julia
ExactDMD(X; dt = 0.0)
```

-  `X` is a data matrix of the observations over time, where each column is
  all observables at a given time point.
- `dt` is an optional value for the spacings
  between the observation timepoints which is used in the construction of the
  continuous dynamics.

#### Extended Dynamic Mode Decomposition (eDMD)

Extended Dynamic Mode Decomposition (eDMD), is a method for
generating an approximating linear differential equation (the Koopman operator)
for the observed data in a chosen basis of observables. Thus the signature is:

```julia
ExtendedDMD(X,basis; dt = 1.0)
```

-  `X` is a data matrix of the observations over time, where each column is
  all observables at a given time point.
- `dt` is an optional value for the spacings
  between the observation timepoints which is used in the construction of the
  continuous dynamics.

#### Shared DMD Features

To get the dynamics from the DMD object, use the `dynamics` function:

```julia
dynamics(dmd, discrete=true)
```

This will build the `f` function for DifferentialEquations.jl integrators, and
defaults to building approximations for `DiscreteProblem`, but this is changed
to approximating `ODEProblem`s by setting `discrete=false`.

The linear approximation can also be analyzed using the following functions:

```julia
eigen(dmd)
eigvals(dmd)
eigvecs(dmd)
```

Additionally, DMD objects can be updated to add new measurements via:

```julia
update!(dmd, x; Δt = 0.0, threshold = 1e-3)
```

## Examples

### Structural Identification with SInDy

In this example we will showcase how to automatically recover the differential
equations from data using the SInDy method. First, let's generate data
from the pendulum model. The pendulum model looks like:

```julia
using DataDrivenDiffEq, ModelingToolkit, DifferentialEquations, LinearAlgebra,
      Plots

function pendulum(u, p, t)
    x = u[2]
    y = -9.81sin(u[1]) - 0.1u[2]
    return [x;y]
end

u0 = [0.2π; -1.0]
tspan = (0.0, 40.0)
prob = ODEProblem(pendulum, u0, tspan)
sol_full = solve(prob,Tsit5())
sol = solve(prob,Tsit5(),saveat=0.75)
data = Array(sol)

plot(sol_full)
scatter!(sol.t,data')
```

In order to perform the SInDy method, we will need to get an approximate
derivative for each observable at each time point. We will do this with the
following helper function which fits 1-dimensional splines to each observable's
time series and uses the derivative of the splines:

```julia
using Dierckx
function colloc_grad(t::T, data::D) where {T, D}
  splines = [Dierckx.Spline1D(t, data[i,:]) for i = 1:size(data)[1]]
  grad = [Dierckx.derivative(spline, t[1:end]) for spline in splines]
  grad = [[grad[1][i],grad[2][i]] for i = 1:length(grad[1])]
  grad = convert(Array, VectorOfArray(grad))
  return grad
end
DX = colloc_grad(sol.t,data)
```

Now that we have the data, we need to choose a basis to fit it to. We know it's
a differential equation in two variables, so let's define our two symbolic
variables with ModelingToolkit:

```julia
@variables u[1:2]
```

Now let's choose a basis. We do this by building an array of ModelingToolkit
`Operation`s that represent the possible terms in our equation. Let's do this
with a bunch of polynomials, but also make sure to include some trigonometric
functions (since the true solution has trigonometric functions!):

```julia
# Lots of polynomials
polys = [u[1]^0]
for i ∈ 1:3
    for j ∈ 1:3
        push!(polys, u[1]^i*u[2]^j)
    end
end

# And some other stuff
h = [1u[1];1u[2]; cos(u[1]); sin(u[1]); u[1]*u[2]; u[1]*sin(u[2]); u[2]*cos(u[2]); polys...]
```

Now we build our basis:

```julia
basis = Basis(h, u)
```

From this we perform our SInDy to recover the differential equations in this basis:

```julia
Ψ = SInDy(data, DX, basis, ϵ = 1e-10)
```

From here we can use Latexify.jl to generate the LaTeX form of the outputted
equations via:

```julia

```

Wow, we recovered the equations! However, let's assume we didn't know the
analytical solution. What we would want to do is double check how good our
is. To do this, we can generate the dynamics and simulate to see how good
the regenerated dynamics fit the original data:

```julia
estimator = ODEProblem(dynamics(Ψ), u0, tspan)
sol_ = solve(estimator, saveat = sol.t)
plot(sol_)
scatter!(data)
```

### Linear Approximation of Dynamics with eDMD
