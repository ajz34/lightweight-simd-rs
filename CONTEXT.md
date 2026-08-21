# Lightweight SIMD

A context for fixed-length vector types whose operations stay portable across all
architectures, with performance delegated to the compiler rather than hand-written
intrinsics. This file is the glossary for that context; scopes and rationale live in
`README.md`.

## Language

**Vector**:
A fixed-length group of lanes treated as a single value; the crate's central concept.
_Avoid_: SIMD register, packed type, fixed array type

**Lane**:
One scalar slot of a vector, addressed by position.
_Avoid_: slot, channel, component

**Element**:
The scalar type that a vector's lanes hold.
_Avoid_: scalar type, base type

**Lane-wise**:
Acting independently on corresponding lanes of one or more vectors.
_Avoid_: element-wise, component-wise

**Reduction**:
An operation that combines all lanes of a vector into a single scalar.
_Avoid_: horizontal operation, cross-lane accumulation

**Mask**:
A vector whose lanes are boolean, used to choose between two vectors lane by lane.
_Avoid_: predicate vector, boolean vector

**Splat**:
Constructing a vector whose lanes all hold the same scalar.
_Avoid_: broadcast, fill

**Partial transfer**:
Moving data between a vector and a run of scalars shorter than the lane count: reading
zero-fills the unused lanes, writing truncates to the run's length.
_Avoid_: masked load/store, padded load

**Tail lanes**:
The lanes beyond the valid data of a partial transfer.
_Avoid_: padding lanes, remainder lanes

**Fallback**:
The portable implementation strategy of this crate: no explicit intrinsics, no runtime
instruction selection; speed comes from compiler auto-vectorization under the requested
target CPU.
_Avoid_: soft SIMD, scalar path, backend

**Aligned alias** (planned):
A named vector type that adds enforced memory alignment on top of the generic vector.
_Avoid_: aligned type, aligned wrapper

**Block**:
An application-level grouping of several vectors for cache-blocking; explicitly outside
this crate's scope.
_Avoid_: tile, batch
