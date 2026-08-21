# Aligned aliases; masks stay plain

The concrete width aliases (`f64x8`, `f32x16`, …) are enforced-alignment types
`AlignedK<T, N>` with `K` = the vector's byte size rounded up to a power of two,
capped at 64 bytes — the pre-1.0 alias redefinition accepted when planning this
step. `AlignedK<T, N>` is an array-backed sibling of `Simd<T, N>` (also `pub
[T; N]`), not a wrapper: every impl block is written once and instantiated for
`Simd` and the five aligned types together via `duplicate_item` rows, so all six
types share one implementation source and cannot drift apart.

## Considered Options

- **Wrapper + delegation** (`AlignedK(Simd<T, N>)` delegating each op): the
  first step-2 implementation; superseded because it duplicates the whole op
  surface per wrapper and must be manually kept in sync with `Simd`.
- **Blanket impls over a conversion trait**: illegal for operators — foreign
  traits (`core::ops::Add`) cannot be implemented for a bare type parameter
  (E0210), which is why per-type impl blocks exist at all.
- **Parallel alias names** (`f64x8` plain + `f64x8a` aligned): rejected — two
  names for one concept, and the common name would carry the slower layout.
- **Alignment on the generic type**: impossible — `#[repr(align)]` requires
  literal alignment values.

## Consequences

- Comparisons on any vector type return the plain mask `Simd<bool, N>`, and
  selection is `value.mask_select(mask, other)` (value-side, available on every
  vector type): a mask-side `select` cannot name the operand alignment
  generically (it depends on `T` and `N`). Masks stay unaligned by design —
  they are register-resident temporaries.
- Pre-0.1 code holding `f64x8 = Simd<f64, 8>` must switch to `Simd<f64, 8>`
  explicitly or convert (`From`/`.0`/`to_simd`); `AlignedK<T, N>` pads small
  vectors up to the alignment in memory.
