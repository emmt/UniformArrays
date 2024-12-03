# Structured arrays for Julia

[![License][license-img]][license-url]
[![Build Status][github-ci-img]][github-ci-url]
[![Build Status][appveyor-img]][appveyor-url]
[![Coverage][codecov-img]][codecov-url]

`StructuredArrays` is a [Julia][julia-url] package which provides multi-dimensional arrays
behaving like regular arrays but whose elements have the same given value or are lazily
computed by applying a given function to their indices. The main advantage of such arrays
is that they are very light in terms of memory: their storage requirement is `O(1)`
whatever their size instead of `O(n)` for an ordinary array of `n` elements.

Note that `StructuredArrays` has a different purpose than
[`StructArrays`](https://github.com/JuliaArrays/StructArrays.jl) which is designed for
arrays whose elements are `struct`.


## Installation

To install the latest stable release with Julia's package manager:

``` julia
] add StructuredArrays
```


## Uniform arrays

All elements of a uniform array have the same value. A uniform array thus require to only
store this value and the dimensions (or the axes) of the array. In addition, some
operations (e.g., `minimum`, `maximum`, `extrema`, `all`, `any`, `sum`, `prod`, `count`,
`findmin`, `findmax`, `reverse`, or `unique`) may be implemented so as to be very fast for
uniform arrays.

To build a uniform array, call:

```julia
A = UniformArray(val, args...)
```

which yields an array `A` behaving as a read-only array whose values are all `val` and
whose axes are defined by `args...`. Each of `args...` define an array axis and can be an
integer for a 1-based index or an integer-valued unit step range. It is thus possible to
have offset axes as in the
[`OffsetArrays`](https://github.com/JuliaArrays/OffsetArrays.jl) package.

Uniform arrays implement conventional linear indexing: `A[i]` yields `val` for all linear
indices `i` in the range `1:length(A)`.

The statement `A[i] = x` is however not implemented because uniform arrays are considered
as read-only. You may call `MutableUniformArray(val,dims)` to create a mutable uniform
array. For example:

```julia
B = MutableUniformArray(val, args...)
```

For `B`, the statement `B[i] = x` is allowed to change the value of all the elements of
`B` provided index `i` represents all possible indices in `B`. Typically `B[:] = x` or
`B[1:end] = x` are accepted but not `B[1] = x`, unless `B` has a single element.

Apart from all values being the same, uniform arrays behave like ordinary Julia arrays.

When calling a uniform array constructor, the element type `T` and the number of
dimensions `N` may be specified. This is most useful for `T` to enforce a given element
type. By default, `T = typeof(val)` is assumed. For example:

```julia
A = UniformArray{T}(val, args...)
B = MutableUniformArray{T,N}(val, args...)
```


## Fast uniform arrays

A fast uniform array is like an immutable uniform array but with the value of all elements
being part of the signature so that this value is known at compile time. To build such an
array, call one of:

```julia
A = FastUniformArray(val, args...)
A = FastUniformArray{T}(val, args...)
A = FastUniformArray{T,N}(val, args...)
```


## Structured arrays

The values of the elements of a structured array are computed on the fly as a function of
their indices. To build such an array, call:

```julia
A = StructuredArray(func, args...)
```

which yields a read-only array `A` whose values at index `i` are computed as `func(i)` and
whose axes are defined by `args...`. In other words, `A[i]` yields `func(i)`.

An optional leading argument `S` may be used to specify another index style than the
default `IndexCartesian`:

```julia
A = StructuredArray(S, func, args...)
```

where `S` may be a sub-type of `IndexStyle` or an instance of such a sub-type. If `S` is
`IndexCartesian` (the default), the function `func` will be called with `N` integer
arguments, a `Vararg{Int,N}`, `N` being the number of dimensions. If `S` is `IndexLinear`,
the function `func` will be called with a single integer argument, an `Int`.

A structured array can be used to specify the location of structural non-zeros in a sparse
matrix. For instance, the structure of a lower triangular matrix of size `m×n` could be
given by:

```julia
StructuredArray((i,j) -> (i ≥ j), m, n)
```

but with a constant small storage requirement whatever the size of the matrix.

Although the callable object `func` may not be a *pure function*, its return type shall be
stable and structured arrays are considered as immutable in the sense that a statement
like `A[i] = x` is not implemented. The type, say, `T` of the elements of structured array
is inferred by applying `func` to the unit index or may be explicitly specified:

```julia
StructuredArray{T}(S, func, dims)
```

where, if omitted, `S = IndexCartesian` is assumed. The `StructuredArray` constructor also
supports the number of dimensions `N` and the indexing style `S` as optional type
parameters. The two following examples are equivalent:

```julia
A = StructuredArray{T,N}(S, func, args...)
A = StructuredArray{T,N,S}(func, args...)
```


## Cartesian meshes

As implemented in `StructuredArrays`, Cartesian `N`-dimensional meshes have equally spaced
nodes with given *step* and *origin*. The mesh *step* is the spacing between contiguous
nodes, it may be different (in length and units) along the different dimensions of the
mesh. The mesh *origin* is the index of the node whose coordinates are all equal to zero,
this index has no units and may be fractional.

Assuming the `step` and `origin` of a mesh are both specified as `N`-tuples, the
coordinates of the node at Cartesian index `(i1,i2,...)` are the `N`-tuple:

```julia
(step[1]*(i1 - origin[1]), step[2]*(i2 - origin[2]), ...)
```

If `step` and/or `origin` are scalars, they are assumed to be the same for all dimensions.
The `origin` may also be `nothing` (the default), to assume that the origin of the mesh is
at index `(0,0,...)`. In the implementation, the exact formula used to compute the
coordinates of the nodes is optimized for the different possible cases. As a consequence,
specifying `origin` as `0` or as a `N`-tuple of `0`s yields a mesh with the same
coordinates but computed with more overheads than with `origin = nothing`.

The values of `step` and `origin` stored by a mesh object `A` may be retrieved by calling
`step(A)` or `origin(A)` Call `origin(Tuple,A)` to retrieve a `N`-dimensional (possibly
fractional) index in all cases.


### Cartesian mesh as a function

To create a Cartesian mesh as a function, call:

```julia
using StructuredArrays.Meshes
mesh = CartesianMesh{N}(step, origin = nothing)
```

to build a callable object such that `mesh(i1,i2,...)` generates the coordinates of the
nodes according to the above formula. If any of `step` or `origin` is a `N`-tuple, the
parameter `N` may be omitted. The node indices may also be specified as a `N`-tuple or as
a `CartesianIndex`.

The values of `step` and `origin` stored by a mesh may be converted to reduce the number
of operations when computing coordinates. This optimization assumes that all
indices are specified as `Int`s but fractional indices (i.e. reals) are also accepted.


### Cartesian mesh as an array

A typical usage is to wrap a Cartesian mesh function in a `StructuredArray` to build an
abstract array whose values are the coordinates of the nodes of a finite size Cartesian
mesh. For example:

``` julia
using StructuredArrays.Meshes
 A = StructuredArray(CartesianMesh{N}(step, origin), inds...)
```

with `inds...` the *shape* (indices and/or dimensions) of the mesh. This can also be done
by calling:


``` julia
using StructuredArrays
A = CartesianMeshArray(inds...; step, origin=nothing)
```

Then the syntax `A[i1,i2,...]` yields the coordinates of the node at index `(i1,i2,...)`.
Node indices may also be specified as a tuple of `Int`s or as a `CartesianIndex`. The
coordinates are lazily computed and the storage is `O(1)`.


[license-url]: ./LICENSE.md
[license-img]: http://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat

[github-ci-img]: https://github.com/emmt/StructuredArrays.jl/actions/workflows/CI.yml/badge.svg?branch=master
[github-ci-url]: https://github.com/emmt/StructuredArrays.jl/actions/workflows/CI.yml?query=branch%3Amaster

[appveyor-img]: https://ci.appveyor.com/api/projects/status/github/emmt/StructuredArrays.jl?branch=master
[appveyor-url]: https://ci.appveyor.com/project/emmt/StructuredArrays-jl/branch/master

[codecov-img]: https://codecov.io/github/emmt/StructuredArrays.jl/graph/badge.svg?token=QhmKO7PmN1
[codecov-url]: https://codecov.io/github/emmt/StructuredArrays.jl

[julia-url]: https://julialang.org/
[julia-pkgs-url]: https://pkg.julialang.org/
