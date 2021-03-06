[![Build Status](https://travis-ci.org/Yurlungur/pyballd.svg?branch=master)](https://travis-ci.org/Yurlungur/pyballd)

pyballd
=======

Author: Jonah Miller (jonah.maxwell.miller@gmail.com)

A Pseudospectral Elliptic Solver for Axisymmetric Problems Implemented
in Python

# Installation

Simply clone the repository and use

```bash
python setup.py install
```

# Pseudospectral Derivatives

Pyballd uses Chebyshev pseudospectral derivatives to attain very high
accuracy with fairly low resolution. For example, if we numerically
take second-order derivatives of this function:

![analytic function](figs/test_function.png)

and vary the number of points (or alternatively the maximum order of
polynomial used for differentiation), we find that our error decays
exponentially with the number of points. This is called "spectral" or
"evanescent" convergence:

![evanescent convergence](figs/orthopoly_errors.png)

# Domain

The appropriate domain for an axisymmetric problem is

![axisymmetric domain base](eqns/domain_base.gif)

where *r<sub>h</sub>* is some minimum radius. Infinite domains are
difficult to handle. Therefore, we define

![definition of x for most boundaries](eqns/def_x_dirichlet.gif)

for most boundary situations or (following the work of [1])

![definition of x](eqns/def_x_bh.gif)

when *r<sub>h</sub>* is a black hole event horizon. Then, following
Boyd [2], we then define

![definition of X](eqns/def_X_new.gif)

for some characteristic length scale *L* so that *X* is defined on the
domain *[-1,1]*. We perform our differentiation on *X*, which has no
effect on the original PDE system except the introduction of Jacobian
terms of the form

![jacobian terms](eqns/X_Jacobian.gif)

in a few places. Since one may want to assume additional (or
different!) symmetry in the longitudinal direction, we do not impose
any restriction there.

## Jacobian for the Compactified Domain

When *x* is defined as 

![definition of x for most boundaries](eqns/def_x_dirichlet.gif)

the Jacobian for the coordinate transformation looks like

![Jacobian for the coordinate transformation](figs/domain_dXdr.png)

As we shall see, this spectral method can efficiently represent both
exponential and algebraic decay with spectral accuracy.

## Convergence on the Compactified Domain

The convergence of the scheme depends senitively on the characteristic
length scale *L*. Generically it will be *spectral* but
*subgeometric*, meaning it's not quite as fast as in the non-compact
case.

### Functions that Decay Rationally

Consider this function

![test function on compact domain](figs/domain_test_function_alg_2d.png)

which has this derivative

![derivative of test function on compact domain](figs/deriv_domain_test_function_alg_2d.png)

On the compact domain (on the equator), this function becomes

![test function on equator](figs/domain_test_function_alg.png)

with derivative

![derivative of test function on equator](figs/deriv_domain_test_function_alg.png)

We achieve best convergence for this function with *L=1*:

![errors on compactified domain](figs/domain_l1_errors_alg.png)

### Functions that decay exponentially

On the other hand, consider this function:

![test function on compact domain](figs/domain_test_function_exp_2d.png)

which has this derivative

![derivative of test function on compact domain](figs/deriv_domain_test_function_exp_2d.png)

On the compact domain (on the equator), this function becomes

![test function on equator](figs/domain_test_function_exp.png)

with derivative

![derivative of test function on equator](figs/deriv_domain_test_function_exp.png)

We achieve best convergence for this function with

![Lexp](eqns/Lexp.gif)

as shown here:

![errors on compactified domain](figs/domain_l1_errors_exp.png)

# Solving an Elliptic PDE

In Pyballd, an elliptic system is defined via a *residual.* A residual

![residual](eqns/residual.gif)

acts on a state vector *u* and its first and second derivatives in (in
our case, axisymmetry) *r* and &#952;. If

![residual vanishes](eqns/residual_vanishes.gif)

then *u(r,&#952;)* is a solution to the PDE system.

An elliptic PDE is not well-posed without the addition of boundary
conditions, which select for the particular solution. At infinity
(*X=1*), we automatically demand that the solution must vanish. (In
other words, we demand that all solutions are square-integrable.)

Dirichlet, Neumann, or Robin boundary conditions can be imposed on
the inner radius (*r_h*), the position of minimum &#952; (often the
axis of symmetry), and the position of maximum &#952; (often the axis
of symmetry or the equator).

In the case of black-hole like compactifications, where

![definition of x](eqns/def_x_bh.gif)

we automatically impose Von-Neumann boundary conditions at the inner
boundary such that

![neumann regularity condition](eqns/von-neumann-regularity-condition.gif)

which is a regularity condition we need to impose on solutions in
these coordinates.

## Pyballd's API

Pyballd simply requires that the user pass in functions that vanish
when the residual and boundary conditions are satisfied. Along with
information about the domain, such as the value of *r<sub>h</sub>*,
this is sufficient information to construct a solution. 

## An Example: Poisson's Equation

Poisson's equation is

![poissons equation generic](eqns/poisson_generic.gif),

or, in spherical coordinates it is

![poisson in spherical coordinates](eqns/poisson_spherical_coordinates.gif).

If we assume axisymmetry so that the azimuthal derivatives vanish and
multiply both sides of the equation by the appropriate factors, we
attain

![poisson in axisymmetry](eqns/poisson_axisymmetry.gif).

For simplicity, we further restrict ourselves to the source-free case and attain

![lapplace eqn in axisymmetry](eqns/lapplace_axisymmetry.gif).

We would like to solve this problem using pyballd.

We begin by importing `pyballd` and `numpy`.

```python
import pyballd
import numpy as np
```

`pyballd` expects a residual, which defines the PDE system. The PDE is
satisfied when the residual vanishes. We define ours as

```python
def residual(r,theta,u,d):
    u = u[0]
    out = (2*np.sin(theta)*r*d(u,1,0)
           + r*r*np.sin(theta)*d(u,2,0)
           + np.cos(theta)*d(u,0,1)
           + np.sin(theta)*d(u,0,2))
    out = out.reshape(tuple([1]) + out.shape)
    return out
```

Here `d` is a derivative operator. So `d(u,2,0)` corresponds to two
derivatives with respect to *r* of *u*, while `d(u,0,2)` corresponds
to two derivatives with respect to &#952;.

The weird array reshaping is an artifact of the fact that `pyballd` is
designed for vector and tensor equations, not just scalar
equations. Therefore, scalars must be treated as vectors of length 1.

`pyballd` allso expects a boundary condition for the inner boundary. We
choose a Dirichlet boundary condition and define it as:

```python
k = 4
a = 2
def bdry_X_inner(theta,u,d):
	u = u[0]
	out = u - a*np.cos(k*theta)
	out = out.reshape(tuple([1]) + out.shape)
	return out
```

This works a lot like `residual` defined above. However, it will only
be evaluated at the inner boundary. The boundary condition is
satisfied when `bdry_X_inner` vanishes.

Nonlinear elliptic systems are not necessarily unique. (And even
linear ones may be difficult to uniquely solve numerically.)
Therefore, we must feed the solver with an initial guess for the
solution. We define ours as

```python
def initial_guess(r,theta):
	out = 1./r
	out = out.reshape(tuple([1]) + out.shape)
	return out
```

which is a simple square integrable function. It's not a solution that
matches the boundary conditions. However, it should be sufficiently
close to the true solution to allow for convergence.

Finally, we ask `pyballd` to generate a solution by calling
`pyballd.pde_solve_once`. Because it's a spectral method, you don't
need many nodes! For example:

```python
SOLN,s = pyballd.pde_solve_once(residual,
                                r_h = 1.0,
                                order_X = 24,
                                order_theta = 6
                                theta_max = np.pi/2,
                                bdry_X_inner = bdry_X_inner,
                                initial_guess = initial_guess)
```

`pyballd.pde_solve_once` returns the solution on the product grid of
colocation points and a *discretization object* `s`, which can
differentiate a function defined on the colocation points, or
interpolate it to a uniform grid. For example:

```python
SOLN = SOLN[0]
r = np.linspace(r_h,4,200)
theta = np.linspace(0,np.pi/2,200)
R,THETA = np.meshgrid(r,theta,indexing='ij')
interpolator = s.get_interpolator_of_r(SOLN)
soln_interp = interpolator(R,THETA)
x,z = R*np.sin(THETA), R*np.cos(THETA)
plt.pcolor(mx,mz,soln_interp)
plt.xlim(0,2)
plt.ylim(0,2)
```

will produce a beautiful plot of the solution interpolated to a finer
grid, such as this one:

![solution to the Poisson equation](figs/poisson_solution.png)

Both `pde_solve_once` and discretization objects, called
`PyballdStencil`s in the code, have a large number of options and
defaults. For a full description of of these, see the help strings
associated with them.

# References

[1] Herderio, Radu, Runarrson. "Kerr black holes with Proca
hair." *Classical and Quantum Gravity* **33-15** (2016).

[2] Boyd, John P. "Orthogonal rational functions on a semi-infinite
interval." *Journal of Computational Physics* **70.1** (1987): 63-88.
