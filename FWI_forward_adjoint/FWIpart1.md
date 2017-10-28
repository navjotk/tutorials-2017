---
title: Full-Waveform Inversion - Part 1``:`` forward and adjoint modeling
runninghead : Part 1, modeling for inversion
author: |
	Mathias Louboutin^1^\*, Philipp Witte^1^, Michael Lange^2^, Navjot Kurjeka^2^, Fabio Luporini^2^, Gerard Gorman^2^, and Felix J. Herrmann^1,3^\
	^1^ Seismic Laboratory for Imaging and Modeling (SLIM), The University of British Columbia \
	^2^ Imperial College London, London, UK\
	^3^ now at Georgia Institute of Technology, USA \
bibliography:
	- bib_tuto.bib
---


## Introduction

Since its re-introduction by @Pratt, Full-waveform inversion (FWI) has gained a lot of attention in geophysical exploration because of its ability to build high resolution velocity models more or less automatically in areas of complex geology. While there is an extensive and growing literature on this topic, publications focus mostly on technical aspects, making this topic inaccessible for a broader audience due to the lack of simple introductory resources for geophysical newcomers. This is part one of two tutorials that attempt to provide an introduction and software to help people getting started. We hope to accomplish this by providing a hands-on walkthrough of FWI using Devito [@lange2016dtg], a system based on domain specific languages that automatically generates code for time-domain finite differences. In this capacity, Devito provides a concise and straightforward computational framework for discretizing wave equations, which underlie all FWI frameworks. We will show that it generates verifiable executable code at run time for wave propagators associated with forward and adjoint wave equations. Devito releases the user from recurrent and time-consuming coding of performant time-stepping codes and allows to concentrate on the geophysics of the problem rather than on low-level implementation details of wave-equation simulators. This tutorial covers the conventional adjoint-state formulation of full-waveform tomography [@Tomo] that underlies most of the current methods referred to as full-waveform inversion. While other formulations have been developed to improve the convergence properties of FWI, we will concentrate on the standard formulation that relies on the combination of a forward/adjoint pair of propagators and a correlation-based gradient.

Full-waveform inversion tries to iteratively minimize the difference between data that was acquired in a seismic survey and synthetic data that is generated from a wave simulator with an estimated model of the subsurface. As such, each FWI framework essentially consists of a wave simulator for forward modeling the predicted data and an adjoint simulator for calculating a model update from the data misfit. The first part of this tutorial is dedicated to the forward modeling part and demonstrates how to discretize and implement the acoustic wave equation using Devito. This tutorial is accompanied by a Jupyter notebook - **`forward_modeling.ipynb`** -, in which we describe how to simulate synthetic data for a specified source and receiver setup and how to save the corresponding wavefields and shot records. In part two of this series, we will address how to calculate model updates, i.e. gradients of the FWI objective function, via adjoint modeling and part three will demonstrate how to use the gradient as part of an optimization framework for inverting an unknown velocity model. 

Installation instructions for Devito are detailed at the end of the paper and required to execute the notebook.

## Wave simulations for inversion

The acoustic wave equation with the squared slowness ``m``, defined as ``m(x,y)=c^{-2}(x,y)`` with ``c(x,y)`` being the unknown spatially varying wavespeed, is given by:

```math {#WE}
 m \frac{d^2 u(x,y,t)}{dt^2} - \nabla^2 u(x,y,t) + \eta(x,y) \frac{d u(x,y,t)}{dt}=q(x,y,t;x_s, y_s),
```
where ``q(x,y,t;x_s)`` is the seismic source, located at ``(x_s,y_s)`` and ``\eta(x,y)`` is a space-dependent dampening parameter for the absorbing boundary layer [@Cerjan]. As shown in Figure #model, the physical model is extended in every direction by `nbpml` grid points to mimic an infinite domain. The dampening term ``\eta \frac{d u(x,t)}{dt}`` attenuates the waves in the dampening layer [@Cerjan] and prevents waves to reflect at the model boundaries. In Devito, the discrete representations of  ``m`` and ``\eta`` are contained in a `model` object that contains all relevant information such as the origin of the coordinate system, grid spacing and size of the model---i.e., in `Python` we have

```python
	# Define a Devito object with the velocity model and grid information
	model = Model(vp=vp, origin=origin, shape=shape, spacing=spacing, nbpml=40)
```

In the `Model` instantiation, `vp` is the velocity in ``\text{km}/\text{s}``, `origin` is the origin of the physical model in meters, `spacing` is the discrete grid spacing in meters, `shape` is the number of grid points in each dimension and `nbpml` is the number of grid points in the absorbing boundary layer. Is is important to note that `shape` is the size of the physical domain only, while the total number of grid points, including the absorbing boundary layer, will be automatically derived from `shape` and `nbpml`.

####Figure: {#model}
![Model](Figures/setup.png){width=50%}
: Representation of the computational domain and its extension, which contains the absorbing boundaries layer.


### Symbolic definition of the wave propagator

To model seismic data by solving the acoustic wave equation, the first necessary
step is to discretize our PDE, which includes discrete representations
of the velocity model and wavefields, as well as approximations of the spatial and 
temporal derivatives using finite differences (FD). However, instead of writing out 
long finite difference stencils by hand, we are going to employ the powerful symbolic
representations of Devito.

The primary design objective of Devito is to allow users to define
complex matrix-free finite difference operators from high-level
symbolic definitions, while employing automated code generation to
create highly optimized low-level C code. For this purpose Devito uses
the symbolic algebra package SymPy [@Meurer17] to facilitate the
automatic creation of derivative expressions, allowing the quick and
efficient generation of high-order wave propagators with variable
stencil orders, while automated performance optimization using
state-of-the-art compiler techniques are used to ensure
computational efficiency.

At the core of Devito's symbolic API are two symbolic types that
behave like SymPy function objects, while also managing
user data:

* `Function` objects represent a spatially varying function
  discretized on a regular cartesian grid. For example, a function
  symbol `f = Function(name='f', ..., space_order=2)`
  is denoted symbolically as `f(x, y)`. Auto-generated symbolic
  expressions for finite difference derivatives are provided by these
  objects via shorthand expressions, where the `space_order` parameter
  defines the order of the created finite difference stencil.

* `TimeFunction` objects represent a time-dependent function
  that includes leading dimension $t$, for example `g(t, x, y)`. In
  addition to spatial derivatives `TimeFunction` symbols also provide
  time derivatives `g.dt` and `g.dt2`, as well as options to store
  the entire data along the time axis.

To demonstrate Devito's symbolic capabilities, let us consider a
time-dependent function $\vd{u}(t, x, y)$ representing the discrete forward
wavefield. We can define this as a `TimeData` object in Devito:

```python
  u = TimeData(name='u', grid=model.grid, time_order=2,
               space_order=2, save=True, timd_dim=nt)
```

where the the `grid` object provided by the `model` defines the size
of the allocated memory region, `time_order` and `space_order` define
the default discretization order of any derived derivative expressions.

We can now use this symbolic representation of our wavefield to
generate simple discretized stencil expressions for finite difference
derivatives using shorthand expressions, such as `u.dx` and `u.dx2` to
denote $\frac{\partial u}{\partial x}$ and $\frac{\partial^2
u}{\partial x^2}$ respectively.

```python
  In []: u
  Out[]: u(t, x, y)

  In []: u.dx
  Out[]: -u(t, x, y)/h_x + u(t, x + h_x, y)/h_x

  In []: u.dx2
  Out[]: -2*u(t, x, y)/h_x**2 + u(t, x-h_x, y)/h_x**2 + u(t, x-h_x, y)/h_x**2
```

Using the automatic derivation of derivative expressions we can now
implement a discretized expression for Equation #WE\ without the
source term $q(x,y,t;x_s, y_s)$. The `model` object which we created
earlier, already contains the squared slowness $\vd{m}(x, y)$ and 
damping term $\vd{\eta}(x, y)$ as `Function` objects:

```python
  # Set up discretized wave equation
  m, eta = model.m, model.damp
  pde = m * u.dt2 - u.laplace + eta * u.dt
```

This expression translates to the following equation if we write out
the (second order) second time derivative `u.dt2` as shown earlier and 
for the moment ignore the damping term:

```math {#WEdis}
 \frac{\vd{m}}{s^2} \Big( \vd{u}[\text{time}-s] - 2\vd{u}[\text{time}] + \vd{u}[\text{time}+s]\Big) - \Delta \vd{u}[\text{time}] = 0, \quad \text{time}=1 \cdots n_{t-1}
```

with time being the current time step and $dt$ being the time stepping interval.
As we can see, the Laplacian $\Delta \vd{u}$ is simply expressed with Devito by
the shorthand expression `u.laplace`, where the order of the derivative stencil is
defined by the `space_order` parameter used to create the symbol `u(t, x, y, z)`. 
However, for solving the wave equation, Equation (#WEdis) needs to be rearranged 
so that we obtain an expression for the wavefield $\vd{u}(\text{time}+s)$
at the next time step (again, ignoring the damping term for now):

```math {#WEstencil}
 \vd{u}[\text{time}+s] = 2\vd{u}[\text{time}] - \vd{u}[\text{time}-s] + \frac{s^2}{\vd{m}} \Delta \vd{u}[\text{time}], \quad \text{time}=1 \cdots n_{t-1}
```

In Python, we can rearrange our ```pde``` expression automatically using 
the SymPy utility function `solve`, to create a stencil expression that defines
the update of the wavefield for the new time step $\vd{u}(\text{time}+s)$, which
is expressed as `u.forward` in Devito:

```python
  # Rearrange PDE to obtain new wavefield at next time step
  stencil = Eq(u.forward, solve(pde, u.forward)[0])
```

This `stencil` expression now represents the finite difference scheme from Equation #WEstencil,
including the FD approximation of the Laplacian and the damping term. The `stencil` expression
defines the update for a single time step only, but since the wavefield `u` is a `TimeFunction`
object, Devito knows that we are solving a time-dependent problem over a number of time steps.


### Setting up the acquisition geometry

In Devito, we model monopole sources/receivers with the object `PointData`, which includes methods that interpolate between the computational grid on which the wave equation is discretized and possibly off-the-grid source/receiver locations.
We showed previously the stencil obtained without a source. In presence of a source the stencil is

```math {#WEdisa}
 \vd{u}[\text{time}+s] = 2\vd{u}[\text{time}] - \vd{u}[\text{time}-s] + \frac{s^2}{\vd{m}} \Big(\Delta \vd{u}[\text{time}] + \vd{q}[\text{time}]\Big), \quad \text{time}=1 \cdots  n_t-1_.
```


and we see that at time step `i` we need to add the source term corresponding to this time-step into the updated wavefield `u[i+1]` (`u.forward` in Devito) with the discretization weight  ``\frac{\Delta t^2}{\vd{m}}`` as the source is inside the physical domain (``\eta = 0``).

The code that implements the definition of the receiver and sources, with locations collected in the arrays `rec_coords` and `src_coords`, reads in `Python` as 

```	
  # Define source injection array for given a source wavelet, coordinates and frequency
  src = RickerSource(name='src', grid=model.grid, f0=f0, time=time, coordinates=src_coords)
  src_term = src.inject(field=u.forward, expr=src * dt**2 / m, offset=model.nbpml)
```

The parameter `offset` is the size of the absorbing layer as shown in Figure #model (source position shifted by `offset`).

On the other side, the receiver is only a read of the wavefield at a specific position at time `i` and does not require any weight.

```python
  # Create receiver array from receiver coordinates
  rec = Receiver(name='rec', grid=model.grid, npoint=101, ntime=nt, coordinates=rec_coords)
  rec_term = rec.interpolate(u, offset=model.nbpml)
```

and the `offset` parameter also correct for the origin shift from the model extension.

### Forward simulation 

With the source/receiver geometry set and the wave-equation stencil
generated, we can now define our forward propagator by adding the
source and receiver terms to our previously defined `stencil`
object. The resulting list is used to generate a `Operator` object
that performs automated generation and optimization of low-level C
code according to the symbolic expressions provided:

```python
  # Create forward propagator
  op_fwd = Operator([stencil] + src_term + rec_term)
```

The symbolic expressions used to create `Operator` contain sufficient
meta-information for Devito to create a fully functional computational
kernel. The dimension symbols contained in the symbolic function
object (`t, x, y, z`) define the loop structure of the created code,
while allowing Devito to automatically optimize the underlying loop
structure to increase performance.

The size of the loops and spacing between grid points is inferred from
the symbolic `Function` objects and associated `model.grid` object at
run-time. As a result we can invoke the generated kernel through a
simple Python function call by supplying the number of timesteps
`time` and the timestep size `dt`. The user data associated with each
`Function` is updated in-place during operator execution, allowing us
to extract the final wavefield and shotrecord directly from the
symbolic function objects.

```python	
  # Generate wavefield snapshots and a shot record
  op_fwd(time=n_timesteps, dt=model.critical_dt)

  # Access the wavefield and shot record at the end of the propagation.
  wavefield = u.data
  shotrecord = rec.data
```

Although Devito `Operator` objects are fully self-contained Python,
the underlying C code can easily be accessed via `op_fw.ccode`, as
shown in Figure #Forward.

####Figure:{#Cgen}
![Generated C code](Figures/ccode-crop.png)

In Figure #Forward, we show the resulting shot record. A movie of snapshots of the forward wavefield can be generated by executing the last cell of **`forward_modeling.ipynb`**.

####Figure: {#Forward}
![Two layer shot record](Figures/shotrecord.pdf){width=45%}
![Marmousi shot record](Figures/shotrecord_marmou.pdf){width=45%}
: Shot record on a two layer and marmousimodel for a single source and split-spread receiver geometry from **`modeling.ipynb`**.

####Figure: {#Snaps}
![T=..33s](Figures/snap1.pdf){width=30%}
![T=.5s](Figures/snap2.pdf){width=30%}
![T=..67](Figures/snap3.pdf){width=30%}
: Snapshots of the wavefield in a two layer model for a source in the middle of the x axis **`modeling.ipynb`**.

## Conclusions

In this first part of the tutorial, we have demonstrated how to set up discretized forward wave equations, their associated propagators with at runtime code generation. In the following part, we will show how to calculate a valid gradient of the FWI objective using the adjoint state method. In part three, we will demonstrate how to set up a complete matrix-free and scalable optimization framework for acoustic FWI.

### Installation

This tutorial and the coming second part are based on Devito version 3.1. It also require to install the full software with examples, not only the code generation API. To install devito

```
	git clone -b v3.1.0 https://github.com/opesci/devito
	cd devito
	conda env create -f environment.yml
	source activate devito
	pip install -e .
```
 
### Usefull links

- [Devito documentation](http://www.opesci.org/)
- [Devito source code and examples](https://github.com/opesci/Devito)
- [Tutorial notebooks with latest Devito/master](https://github.com/opesci/Devito/examples/seismic/tutorials)


### Need to add acknow
##references

``` math_def
\def\argmin{\mathop{\rm arg\,min}}
\def\vec{\mbox{``\mathrm{vec}``}}
\def\ivec{\mbox{``\mathrm{vec}^{-1}``}}
\newcommand{\m}{{\mathsf{m}}}
\newcommand{\PsDO}{\mbox{PsDO\,}}
\newcommand{\Id}{\mbox{``\tensor{I}\,``}}
\newcommand{\R}{\mbox{``\mathbb{R}``}}
\newcommand{\Z}{\mbox{``\mathbb{Z}``}}
\newcommand{\DE}{:=}
\newcommand{\Order}{\mbox{``{\cal O}``}} \def\bindex#1{{\mathcal{#1}}}
\def\pector#1{\mathrm{\mathbf{#1}}} 
\def\cector#1{#1} 
\def\censor#1{#1} 
\def\vd#1{\mathbf{#1}}
\def\fvector#1{{\widehat{\vd{#1}}}}
\def\evector#1{{\widetilde{\vd{#1}}}}
\def\pvector#1{{\breve{\vd{#1}}}}
\def\pector#1{\mathrm{#1}}
\def\ctensor#1{\bm{\mathcal{#1}}}
\def\tensorm#1{\bm{#1}}
\def\tensor#1{\vd{#1}}
\def\hensor#1{\tensor{#1}}
\def\uensor#1{\underline{\bm{#1}}}
\def\hector#1{\vd{#1}}
\def\ftensor#1{{\widehat{\tensor{#1}}}}
\def\calsor#1{{\boldsymbol{\mathcal{#1}}}}
\def\optensor#1{{\boldsymbol{\mathcal{#1}}}}
\def\hvector#1{\hat{\boldsymbol{\mathbf{#1}}}}
\def\minim{\mathop{\hbox{minimize}}}
\newcommand{\norm}[1]{\left\lVert#1\right\rVert_2^2}
\newcommand{\overbar}[1]{\mkern 1.5mu\overline{\mkern-1.5mu#1\mkern-1.5mu}\mkern 1.5mu}
```