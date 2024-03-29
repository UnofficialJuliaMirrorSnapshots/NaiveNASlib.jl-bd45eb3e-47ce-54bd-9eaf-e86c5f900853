# NaiveNASlib

[![Build Status](https://travis-ci.com/DrChainsaw/NaiveNASlib.jl.svg?branch=master)](https://travis-ci.com/DrChainsaw/NaiveNASlib.jl)
[![Build Status](https://ci.appveyor.com/api/projects/status/github/DrChainsaw/NaiveNASlib.jl?svg=true)](https://ci.appveyor.com/project/DrChainsaw/NaiveNASlib-jl)
[![Codecov](https://codecov.io/gh/DrChainsaw/NaiveNASlib.jl/branch/master/graph/badge.svg)](https://codecov.io/gh/DrChainsaw/NaiveNASlib.jl)

NaiveNASlib is a library of functions for mutating computation graphs. It is designed with Neural Architecture Search (NAS) in mind, but can be used for any purpose where doing changes to a model architecture is desired.

It is "batteries excluded" in the sense that it is independent of both neural network implementation and search policy implementation. If you need batteries, check out https://github.com/DrChainsaw/NaiveNASflux.jl.

Its only contribution to this world is some help with the sometimes annoyingly complex procedure of changing an existing neural network into a new, similar yet different, neural network.

## Basic usage

```julia
]add NaiveNASlib
```

Main supported operations:
* Change the input/output size of vertices
* Parameter pruning/insertion (policy excluded)
* Add vertices to the graph
* Remove vertices from the graph
* Add edges to a vertex
* Remove edges to a vertex

For each of the above operations, NaiveNASlib makes the necessary changes to neighboring vertices to ensure that the computation graph is consistent w.r.t dimensions of the activations.

The price one has to pay is that the computation graph must be explicitly defined in the "language" of this library, similar to what some older frameworks using less modern programming languages used to do. In its defense, the sole reason anyone would use this library to begin with is to not have to create computation graphs themselves.

Just to get started, lets create a simple graph for the summation of two numbers:
```julia
using NaiveNASlib
using Test
in1 = inputvertex("in1", 1)
in2 = inputvertex("in2", 1)

# Create a new vertex which computes the sum of in1 and in2
computation = in1 + in2
@test typeof(computation) == MutationVertex

# CompGraph helps evaluating the whole graph as a function
graph = CompGraph([in1, in2], computation);

# Evaluate the function represented by graph
@test graph(2,3) == 5
```

Now for a more to the point example.

```julia
# First we need something to mutate. Batteries excluded, remember?
mutable struct SimpleLayer
    W
    SimpleLayer(W) = new(W)
    SimpleLayer(nin, nout) = new(ones(Int, nin,nout))
end
(l::SimpleLayer)(x) = x * l.W

# Helper function which creates a mutable layer.
layer(in, outsize) = absorbvertex(SimpleLayer(nout(in), outsize), outsize, in, mutation=IoSize)

invertex = inputvertex("input", 3)
layer1 = layer(invertex, 4);
layer2 = layer(layer1, 5);

@test [nout(layer1)] == nin(layer2) == [4]

# Lets change the output size of layer1:
Δnout(layer1, -2);

@test [nout(layer1)] == nin(layer2) == [2]
````

As can be seen above, the consequence of changing the output size of `layer1` was that the input size of `layer2` also was changed.

Besides the very simple graph, this mutation was trivial because both layers are of the type `SizeAbsorb`, meaning that a change in number of inputs/outputs does not propagate further in the graph.

Lets do a non-trivial example where changes propagate:

```julia
# First a few "normal" layers
invertex = inputvertex("input", 6);
start = layer(invertex, 6);
split = layer(start, div(nout(invertex) , 3));

# When multiplying with a scalar, the output size is the same as the input size.
# This vertex type is said to be SizeInvariant (in lack of better words).
scalarmult(v, f::Integer) = invariantvertex(x -> x .* f, v)

# Concatenation is a third type of vertex, called SizeStack where the output size is the sum of input sizes
joined = conc(scalarmult(split, 2), scalarmult(split,3), scalarmult(split,5), dims=2);
@test trait(joined) == SizeStack()

# Elementwise addition is SizeInvariant
out = start + joined;
@test trait(out) == SizeInvariant()

@test [nout(invertex)] == nin(start) == nin(split) == [3 * nout(split)] == [sum(nin(joined))] == [nout(out)] == [6]
@test [nout(start), nout(joined)] == nin(out) == [6, 6]

graph = CompGraph(invertex, out)
@test graph((ones(Int, 1,6))) == [78  78  114  114  186  186]

# Ok, lets try to change the size of the vertex "out".
# First we need to realize that we can only change it by integer multiples of 3
# This is because it is connected to "split" through three paths which require nin==nout
# Therefore, any size change to nout of "split" will result in 3 times the change of nin of "out".
# Equivalently, nout of "split" is nin of "out" divided by 3 and nin/nout must be integers.

# We need this information from the layer. Some layers have other requirements
NaiveNASlib.minΔnoutfactor(::SimpleLayer) = 1
NaiveNASlib.minΔninfactor(::SimpleLayer) = 1

@test minΔnoutfactor(out) == minΔninfactor(out) == 3

# Next, we need to define how to mutate our SimpleLayer
NaiveNASlib.mutate_inputs(l::SimpleLayer, newInSize) = l.W = ones(Int, newInSize, size(l.W,2))
NaiveNASlib.mutate_outputs(l::SimpleLayer, newOutSize) = l.W = ones(Int, size(l.W,1), newOutSize)

# In some cases it is useful to hold on to the old graph before mutating
# To do so, we need to define the clone operation for our SimpleLayer
NaiveNASlib.clone(l::SimpleLayer) = SimpleLayer(l.W)
parentgraph = copy(graph)

Δnout(out, 3)

# We didn't touch the input when mutating...
@test [nout(invertex)] == nin(start) == [6]
# Start and joined must have the same size due to elementwise op.
# All three scalarmult vertices are transparent and propagate the size change to split
@test [nout(start)] == nin(split) == [3 * nout(split)] == [sum(nin(joined))] == [nout(out)] == [9]
@test [nout(start), nout(joined)] == nin(out) == [9, 9]

# However, this only updated the mutation metadata, not the actual layer.
# Some reasons for this are shown in the pruning example below
@test graph((ones(Int, 1,6))) == [78  78  114  114  186  186]

# To mutate the graph, we need to apply the mutation:
apply_mutation(graph);

@test graph((ones(Int, 1,6))) == [114  114  114  168  168  168  276  276  276]

# Copy is still intact
@test parentgraph((ones(Int, 1,6))) == [78  78  114  114  186  186]

```

As seen above, things get a little bit out of hand when using:

* Layers which require nin==nout, such as batch normalization and pooling
* Element wise operations
* Concatenation of activations  

The core idea of NaiveNASlib is basically to annotate the type of vertex in the graph so that functions know what is the proper way to deal with the neighboring vertices when mutating a vertex.

This is done through labeling vertices into three major types:
* `SizeAbsorb`: Assumes `nout(v)` and `nin(v)` may change independently. This means that size changes are absorbed by this vertex in the sense they don't propagate further.

* `SizeStack`: Assumes `nout(v) == sum(nin(v))`. This means that size changes propagate forwards (i.e. input -> output and output -> input).

* `SizeInvariant`: Assumes `[nout(v)] == unique(nin(v))`. This means that size changes propagate both forwards and backwards as changing any input size or the output size means all others must change as well.

To use this library to mutate architectures for some neural network library basically means annotating up the above type for each layer type.  

Lets just do a few quick examples of the other use cases.

Prune (and insert) neurons:
```julia
# Some mockup 'batteries' for this example

# First, how to select or add rows or columns to a matrix
# Negative values in selected indicate rows/cols insertion at that index
function select_params(W, selected, dim)
    Wsize = collect(size(W))
    indskeep = repeat(Any[Colon()], 2)
    newmap = repeat(Any[Colon()], 2)

    # The selected indices
    indskeep[dim] = filter(ind -> ind > 0, selected)
    # Where they are 'placed', others will be zero
    newmap[dim] = selected .> 0
    Wsize[dim] = length(newmap[dim])

    newmat = zeros(Int64, Wsize...)
    newmat[newmap...] = W[indskeep...]
    return newmat
end

NaiveNASlib.mutate_inputs(l::SimpleLayer, selected::Vector{<:Integer}) = l.W = select_params(l.W, selected, 1)
NaiveNASlib.mutate_outputs(l::SimpleLayer, selected::Vector{<:Integer}) = l.W = select_params(l.W, selected, 2)

# Return layer just so we can easiliy look at it
function prunablelayer(in, outsize)
    l = SimpleLayer(reshape(1: nout(in) * outsize, nout(in), :))
    return absorbvertex(l, outsize, in), l
end

# Ok, now lets get down to business!
invertices = inputvertex.(["in1", "in2"], [3,4])
v1, l1 = prunablelayer(invertices[1], 4)
v2, l2 = prunablelayer(invertices[2], 3)
merged = conc(v1, v2, dims=2)
v3, l3 = prunablelayer(merged, 2)
graph = CompGraph(invertices, v3)

# These weights are of course complete nonsense from a neural network perspective.
# They are just to make it easier to spot what has changed after pruning below.
@test l1.W ==
[ 1  4  7  10 ;
  2  5  8  11 ;
  3  6  9  12 ]

@test l2.W ==
[ 1  5   9 ;
  2  6  10 ;
  3  7  11 ;
  4  8  12 ]

@test l3.W ==
[ 1   8 ;
  2   9 ;
  3  10 ;
  4  11 ;
  5  12 ;
  6  13 ;
  7  14 ]

# A limitation in current implementation is that one must change the size before pruning
# See https://github.com/DrChainsaw/NaiveNASlib.jl/issues/40
Δnin(v3, -3)

# What did that do?
@test nout(v1) == 2
@test nout(v2) == 2

# Doing this however makes it possible to do several mutations without throwing away
# more information than needed.
# For example, if we had first applied the previous mutation we would have thrown away
# weights for v2 which would then just be replaced by 0s when doing this:
Δnout(v2, 2)

# What did that do?
@test nout(v1) == 2
@test nout(v2) == 4
@test nin(v3) == [6]
# Net result is that v1 shall decrease output size by 1 and v2 shall increase its output size by 1

# Now, we need a utility/value metric per neuron in order to determine which neurons to keep
# Give high utility to neurons 1 and 3 of v1, same for all others...
utility(v) = v == v1 ? [10, 1, 10, 1] : ones(nout_org(v))
# Then select the neurons.
Δoutputs(graph, utility)
# And apply it to the actual weights
apply_mutation(graph)

@test l1.W ==
[ 1  7 ;
  2  8 ;
  3  9 ]

# Note how column 3 was not replaced by zeros: We increased the target size before pruning
@test l2.W ==
[ 1  5   9  0;
  2  6  10  0;
  3  7  11  0;
  4  8  12  0]

@test l3.W ==
[ 1   8 ;
  3  10 ;
  5  12 ;
  6  13 ;
  7  14 ;
  0   0]
```

Add a vertex to a graph:
```julia
invertex = inputvertex("input", 3)
layer1 = layer(invertex, 5)
graph = CompGraph(invertex, layer1)

@test nv(graph) == 2
@test graph(ones(Int, 1, 3)) == [3 3 3 3 3]

# Insert a layer between invertex and layer1
insert!(invertex, vertex -> layer(vertex, nout(vertex)))

@test nv(graph) == 3
@test graph(ones(Int, 1, 3)) == [9 9 9 9 9]
```

Remove a vertex from a graph:
```julia
invertex = inputvertex("input", 3)
layer1 = layer(invertex, 5)
layer2 = layer(layer1, 4)
graph = CompGraph(invertex, layer2)

@test nv(graph) == 3
@test graph(ones(Int, 1, 3)) == [15 15 15 15]

# Remove layer1 and change nin of layer2 from 5 to 3
# Would perhaps have been better to increase nout of invertex, but it is immutable
remove!(layer1)
apply_mutation(graph)

@test nv(graph) == 2
@test graph(ones(Int, 1, 3)) == [3 3 3 3]
```

Add an edge to a vertex:
```julia
invertices = inputvertex.(["input1", "input2"], [3, 2])
layer1 = layer(invertices[1], 4)
layer2 = layer(invertices[2], 4)
add = layer1 + layer2
out = layer(add, 5)
graph = CompGraph(invertices, out)

@test nin(add) == [4, 4]
# Two inputs this time, remember?
@test graph(ones(Int, 1, 3), ones(Int, 1, 2)) == [20 20 20 20 20]

# This graph is not interesting enough for there to be a good showcase for adding a new edge.
# Lets create a new layer which has a different output size just to see how things change
# The only vertex which support more than one input is add
layer3 = layer(invertices[2], 6)
create_edge!(layer3, add)
apply_mutation(graph)

# By default, NaiveNASlib will try to increase the size in case of a mismatch
@test nin(add) == [6, 6, 6]
@test graph(ones(Int, 1, 3), ones(Int, 1, 2)) == [42 42 42 42 42]
```

Remove an edge from a vertex:
```julia
invertex = inputvertex("input", 4)
layer1 = layer(invertex, 3)
layer2 = layer(invertex, 5)
merged = conc(layer1, layer2, layer1, dims=2)
out = layer(merged, 3)
graph = CompGraph(invertex, out)

@test nin(merged) == [3, 5, 3]
@test graph(ones(Int, 1, 4)) == [44 44 44]

remove_edge!(layer1, merged)
apply_mutation(graph)

@test nin(merged) == [5, 3]
@test graph(ones(Int, 1, 4)) == [32 32 32]
```

## Advanced usage

The previous examples have been focused on giving an overview of the purpose of this library. For more advanced usage, there are many of ways to customize the behavior and in other ways alter or hook in to the functionality. Here are a few of the most important.

### Strategies

For more or less all operations which mutate the graph, it is possible achieve fine grained control of the operation through selecting a strategy.

Here is an example of strategies for changing the size:

```julia
# A simple graph where one vertex has a constraint for changing the size.
invertex = inputvertex("in", 3)
layer1 = layer(invertex, 4)
joined = conc(scalarmult(layer1, 2), scalarmult(layer1, 3), dims=2)

# joined can only change in steps of 2
@test minΔnoutfactor(joined) == 2

# all_in_graph finds all vertices in the same graph as the given vertex
verts = all_in_graph(joined)

# Strategy to try to change it by one and throw an error when not successful
exactOrFail = ΔNout{Exact}(joined, 1, ΔSizeFailError("Size change failed!!"))

# Note that we now call Δsize instead of Δnout as the direction is given by the strategy
@test_throws ErrorException Δsize(exactOrFail, verts)

# No change was made
@test nout(joined) == 2*nout(layer1) == 8

# Try to change by one and fail silently when not successful
exactOrNoop = ΔNout{Exact}(joined, 1, ΔSizeFailNoOp())

Δsize(exactOrNoop, verts)

# No change was made
@test nout(joined) == 2*nout(layer1) == 8

# In many cases it is ok to not get the exact change which was requested
relaxedOrFail = ΔNout{Relaxed}(joined, 1, ΔSizeFailError("This should not happen!!"))

Δsize(relaxedOrFail, verts)

# Changed by two as this was the smallest possible change
@test nout(joined) == 2*nout(layer1) == 10

# Logging when fallback is applied is also possible
using Logging
# Yeah, this is not easy on the eyes, but it gets the job done...
exactOrLogThenRelax = ΔNout{Exact}(joined, 1, LogΔSizeExec(Logging.Info, "Exact failed, relaxing", ΔNout{Relaxed}(joined, 1, ΔSizeFailError("This should not happen!!"))))

@test_logs (:info, "Exact failed, relaxing") Δsize(exactOrLogThenRelax, verts)

@test nout(joined) == 2*nout(layer1) == 12
```
A similar pattern is used for most other mutating operations. Use the built-in documentation to explore the options until I find the energy and time to write proper documentation. As I could not let go of the OO habit of having abstract base types for everything, the existing strategies can be discovered using `subtypes` as a stop-gap solution.

### Traits

A variant (bastardization?) of the [holy trait](https://docs.julialang.org/en/v1/manual/methods/#Trait-based-dispatch-1) pattern is used to annotate the type of a vertex. In the examples above the three 'core' types `SizeAbsorb`, `SizeStack` and `SizeInvariant` are shown, but it is also possible to attach other information and behaviors by freeriding on this mechanism.

This is done by adding the argument `traitdecoration` when creating a vertex and supplying a function which takes a trait and return a new trait (which typically wraps the input).

Some examples:

```julia
noname = layer(inputvertex("in", 2), 2)
@test name(noname) == "MutationVertex::SizeAbsorb"

# Naming vertices is so useful for logging and debugging I almost made it mandatory
named = absorbvertex(SimpleLayer(2, 3), 3, inputvertex("in", 2), traitdecoration = t -> NamedTrait(t, "named layer"))
@test name(named) == "named layer"

# Speaking of logging...
layer1 = absorbvertex(SimpleLayer(2, 3), 3, inputvertex("in", 2), traitdecoration = t -> SizeChangeLogger(NamedTrait(t, "layer1")))

# What info is shown can be controlled by supplying an extra argument to SizeChangeLogger
nameonly = NameInfoStr()
layer2 = absorbvertex(SimpleLayer(nout(layer1), 4), 4, layer1, traitdecoration = t -> SizeChangeLogger(nameonly, NamedTrait(t, "layer2")))

@test_logs(
(:info, "Change nout of layer1, inputs=[in], outputs=[layer2], nin=[2], nout=[3], SizeAbsorb() by 1"),
(:info, "Change nin of layer2 by 1"), # Note: less verbose compared to layer1 due to NameInfoStr
 Δnout(layer1, 1))

 # traitdecoration works exactly the same for conc and invariantvertex as well, no need for an example

 # Use the >> operator when creating SizeInvariant vertices using arithmetic operators:
 add = "addvertex" >> inputvertex("in1", 1) + inputvertex("in2", 1)
 @test name(add) == "addvertex"

 # For more elaborate traits one can use traitconf
 add2 = traitconf(t -> SizeChangeLogger(NamedTrait(t, "layer1 + layer2"))) >> layer1 + layer2
 @test name(add2) == "layer1 + layer2"

 @test_logs(
 (:info, "Change nout of layer1, inputs=[in], outputs=[layer2, layer1 + layer2], nin=[2], nout=[4], SizeAbsorb() by 1"),
 (:info, "Change nin of layer2 by 1"),
 (:info, "Change nout of layer2 by 1"),
 (:info, "Change nin of layer1 + layer2, inputs=[layer1, layer2], outputs=[], nin=[4, 4], nout=[4], SizeInvariant() by 1, 1"),
 (:info, "Change nout of layer1 + layer2, inputs=[layer1, layer2], outputs=[], nin=[5, 5], nout=[4], SizeInvariant() by 1"),
  Δnout(layer1, 1))

  # When creating own trait wrappers, remember to subtype DecoratingTrait or else there will be pain!

  # Wrong!! Not a subtype of DecoratingTrait
  struct PainfulTrait{T<:MutationTrait} <: MutationTrait
      base::T
  end
  painlayer = absorbvertex(SimpleLayer(2, 3), 3, inputvertex("in", 2), traitdecoration = PainfulTrait)

  # Now one must implement a lot of methods for PainfulTrait...
  @test_throws MethodError Δnout(painlayer, 1)

  # Right! Is a subtype of DecoratingTrait
  struct SmoothSailingTrait{T<:MutationTrait} <: DecoratingTrait
      base::T
  end
  # Just implement base and all will be fine
  NaiveNASlib.base(t::SmoothSailingTrait) = t.base

  smoothlayer = absorbvertex(SimpleLayer(2, 3), 3, inputvertex("in", 2), traitdecoration = SmoothSailingTrait)

  Δnout(smoothlayer, 1)
  @test nout(smoothlayer) == 4

```

### Graph instrumentation and modification

In many cases it is desirable to change things like traits of an existing graph. This can be achieved by supplying an extra argument when copying the graph. The extra argument is a function which determines how each individual component of the graph shall be copied.

Depending on what one wants to achieve, it can be more or less messy. Here is a pretty messy example:

```julia
invertex = inputvertex("in", 2)
layer1 = layer(invertex, 3)
layer2 = layer(layer1, 4)

graph = CompGraph(invertex, layer2)

@test name.(vertices(graph)) == ["in", "MutationVertex::SizeAbsorb", "MutationVertex::SizeAbsorb"]

# Ok, lets add names to layer1 and layer2 and change the name of invertex

# Lets first define the default: Fallback to "clone"
# clone is the built-in function to copy things in this manner as I did not want to override Base.copy
copyfun(args...;cf) = clone(args...;cf=cf) # Keyword argument cf is the function to use for copying all fields of the input

# Add a name to layer1 and layer2
function copyfun(v::MutationVertex,args...;cf)
    # This is probably not practical to do in a real graph, so make sure you have names when first creating it...
    name = v == layer1 ? "layer1" : "layer2"
    addname(args...;cf) = clone(args...;cf=cf)
    addname(t::SizeAbsorb;cf) = NamedTrait(t, name) # SizeAbsorb has no fields, otherwise we would have had to run cf for each one of them...
    clone(v, args...;cf=addname)
end

# Change name of invertex
# Here we can assume that invertex name is unique in the whole graph or else we would have had to use the above way
copyfun(s::String; cf) = s == name(invertex) ? "in changed" : s

# Now supply copyfun when copying the graph.
# I must admit that thinking about what this does makes me a bit dizzy...
namedgraph = copy(graph, copyfun)

@test name.(vertices(namedgraph)) == ["in changed", "layer1", "layer2"]
```

## Contributing

All contributions are welcome. Please file an issue before creating a PR.
