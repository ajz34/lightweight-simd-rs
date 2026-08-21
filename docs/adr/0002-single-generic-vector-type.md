# Single generic vector type with trait-bounded operation groups

There is exactly one vector struct, `Simd<T, const N: usize>` over `[T; N]`; element
categories (float, integer, complex, bool) are expressed as impl-block bounds
(`num_traits::Float`, `Num`, `MulAdd`, `core::ops` bit ops, …), not as separate struct
families (`FpSimd`/`IntSimd`/…) and not as a custom ops-trait surface.

## Considered Options

- **Struct family per element category**: rejected — conversions between family
  members, duplicated impls, and friction for mixed use (`Complex<f64>` vectors × real
  scalars).
- **Ops-trait surface** (fearless_simd's `SimdBase`/pulp's `Ops`): rejected for now —
  those traits exist mainly to serve multiple dispatch backends, which ADR 1 removed.
  Extension traits can be introduced later without breaking changes if kernels generic
  over *aligned-ness* (step 2) demand them.

## Consequences

- `bool` is just an element, so `Simd<bool, N>` doubles as the mask type.
- `Simd<Complex<f64>, N>` works through the same generic bounds with no special casing.
- Lane count `N` is arbitrary; no operation may assume a fixed lane count.
