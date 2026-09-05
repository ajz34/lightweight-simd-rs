# Functionality guide — one-file API overview

This document is a compact but complete map of everything [`lightweight-simd`](crate)
offers, written for two audiences: API users who want the big picture without
reading every module, and AI coding agents that need to know what this crate
can and cannot do before writing code against it. The rustdoc API pages remain
the source of truth for exact signatures; this file is the orientation layer.

- Design stance: portable fixed-lane vector types whose every operation is a
  plain loop that the compiler (LLVM) auto-vectorizes. **No intrinsics, no
  inline assembly, no runtime CPU-feature dispatch, no fallback code paths.**
  SIMD-level performance comes from building with `-C target-cpu=native` (or a
  fixed level such as `x86-64-v3`); correctness never depends on it.
- Everything is stable Rust; dependencies are `duplicate` (codegen), and
  `num-traits` (element trait bounds).

```rust
use lightweight_simd::{f64x4, Simd};

let a = f64x4::from([1.0, 2.0, 3.0, 4.0]);
let b = a * 2.0 + f64x4::splat(1.0);              // vector-scalar and vector-vector ops
assert_eq!(b.to_array(), [3.0, 5.0, 7.0, 9.0]);
assert_eq!(b.reduce_sum(), 24.0);                 // horizontal sum

let m = a.simd_gt(f64x4::splat(2.0));             // lane-wise compare -> mask
assert_eq!(a.mask_select(m, f64x4::zero()).to_array(), [0.0, 0.0, 3.0, 4.0]);
```

## The six vector types

There is one generic core type and five aligned siblings; they share a single
implementation, so **every operation listed in this guide exists on all six
types** (except the handful of mask-only helpers, noted below):

| Type | Alignment | Role |
| --- | --- | --- |
| [`Simd<T, N>`](crate::Simd) | natural (element's) | the core type; public field `.0` of type `[T; N]` |
| [`Aligned4<T, N>`](crate::Aligned4) | 4 B | enforced-alignment siblings, |
| [`Aligned8<T, N>`](crate::Aligned8) | 8 B | structurally identical |
| [`Aligned16<T, N>`](crate::Aligned16) | 16 B | (`#[repr(align(n))]` over |
| [`Aligned32<T, N>`](crate::Aligned32) | 32 B | a public `[T; N]`) |
| [`Aligned64<T, N>`](crate::Aligned64) | 64 B | |

- The field `.0` is public on every type: zero-friction interop with plain
  array code, and `.0` / [`From`] conversions move between plain and aligned
  forms losslessly ([`Aligned64::to_simd`](crate::Aligned64::to_simd) too).
- Masks — [`Simd<bool, N>`](crate::Simd) — are never aligned. Comparisons on
  any vector type return plain `Simd<bool, N>`; masks are register-resident
  temporaries.

### Concrete type aliases

Defined in the crate root (`alias` module). The aligned
backing type is chosen as byte size rounded up to a power of two, **capped at
64 bytes** (so e.g. `f64x16` is 128 B of data with 64 B alignment):

- `f64`: [`f64x2`](crate::f64x2) [`f64x4`](crate::f64x4) [`f64x8`](crate::f64x8) [`f64x16`](crate::f64x16)
- `f32`: [`f32x2`](crate::f32x2) [`f32x4`](crate::f32x4) [`f32x8`](crate::f32x8) [`f32x16`](crate::f32x16)
- `i64`: [`i64x2`](crate::i64x2) [`i64x4`](crate::i64x4) [`i64x8`](crate::i64x8) [`i64x16`](crate::i64x16)
- `i32`: [`i32x4`](crate::i32x4) [`i32x8`](crate::i32x8) [`i32x16`](crate::i32x16)
- `i16`: [`i16x4`](crate::i16x4) [`i16x8`](crate::i16x8) [`i16x16`](crate::i16x16) [`i16x32`](crate::i16x32)
- `i8`:  [`i8x8`](crate::i8x8) [`i8x16`](crate::i8x16) [`i8x32`](crate::i8x32) [`i8x64`](crate::i8x64)
- masks: [`mask4`](crate::mask4) [`mask8`](crate::mask8) [`mask16`](crate::mask16) [`mask32`](crate::mask32)
  (= `Simd<bool, N>`; other `Simd<bool, N>` widths work directly too)

Feature-gated aliases (off by default):

- feature `complex` (dep `num-complex`): `c64x2` `c64x4` `c64x8` `c32x4` `c32x8`
  `c32x16` over `num_complex::Complex<f64/f32>`.
- feature `half` (dep `half`): `f16x2` … `f16x64` and `bf16x2` … `bf16x64`.
  Note: `f16`/`bf16`
  arithmetic goes through the `half` crate's scalar impls and stays scalar in
  the emitted code on x86 (documented in the asm checker's informational notes).

The lane count `N` is an arbitrary `const` generic on [`Simd`]/aligned types —
powers of two that fill whole registers are the sweet spot, not a requirement.
Non-builtin element types work wherever their trait bounds fit (`bool`
supports splat/bitwise/compare/select; `Complex<T>` supports arithmetic,
compare, and `mul_add` behind the `complex` feature; `half::f16` supports the
float functions).

## Operation surface

All methods below are inherent methods, `#[inline(always)]`, identical across
the six types. `Simd` is written below for brevity; read it as "any of the six".

### Construction, access, conversion

- [`Simd::from_fn`](crate::Simd::from_fn)(f) — lane `i` is `f(i)`.
- [`Simd::splat`](crate::Simd::splat)(val) — all lanes `val` (`const fn`).
- [`Simd::fill`](crate::Simd::fill)(&mut self, val) — overwrite all lanes.
- [`Simd::zero`](crate::Simd::zero)() / [`Simd::one`](crate::Simd::one)() — via `num_traits`.
- `From<[T; N]>` on every type; plain↔aligned via `From`, `.0`, or
  `to_simd`. **No `From<&[T]>`/`Vec` conversions** — use `from_slice`.
- [`Simd::as_array`](crate::Simd::as_array) / `as_array_mut` / [`Simd::to_array`](crate::Simd::to_array) —
  borrow/move the backing array (`as_array` is `const fn`). There is **no
  iterator over lanes**; use `.as_array().iter()`.
- `Index<usize>` / `IndexMut<usize>` — per-lane access.
- Derived traits on every type: `Clone`, `Copy`, `Debug`, `PartialEq`, `Eq`,
  `Hash`, `Default`; `num_traits::Zero`/`One` where the element admits them.
  **`PartialOrd`/`Ord` are deliberately not implemented** — comparisons go
  through the `simd_*` methods below so they always produce masks.

### Arithmetic

- `Add`/`Sub`/`Mul`/`Div` as vector`op`vector **and** vector`op`scalar; plus
  the four `*Assign` forms in both shapes. Requires `T: num_traits::Num`
  (`NumAssignOps` for the assign forms).
- `Neg` (element's `Neg`).
- No saturating/wrapping/checked variants, no widening/narrowing, no
  `Rem`.

### Bitwise and shifts

- `BitAnd`/`BitOr`/`BitXor` (+ assign forms) and `Not`, by the element's own
  `core::ops` impls. On `Simd<bool, N>` these are the lane-wise logical
  operators — combine masks with `&`, `|`, `^`, `!`.
- `Shl<usize>`/`Shr<usize>`: shift **every lane by one scalar `usize`**.
  There is no lane-wise/varying shift.

### Multiply-add — the one target-dependent area

[`Simd::mul_add`](crate::Simd::mul_add)(b, c) → `self * b + c`, and
[`Simd::fma_from`](crate::Simd::fma_from)(b, c) → `*self = b * c + *self`
(accumulator is the receiver — note the operand-order difference; it matches
`self += b * c` for micro-kernels). Evaluation follows the build target
(policy and measurements: `docs/adr/0005-mul-add-fallback-policy.md`):

1. **Hardware-FMA targets** (x86 with `-C target-cpu` ≥ `x86-64-v3`,
   including `native`; all aarch64): the element's `num_traits::MulAdd` —
   one rounding per lane, one FMA instruction per register.
2. **Targets without** (generic x86-64, `x86-64-v2`, unknown arches):
   separated multiply + add, two roundings — deliberately never the slow
   software libm `fma` (~4.6× slower measured). Fuses back to a hardware FMA
   under `-C llvm-args=-fp-contract=fast`.
3. Feature **`use_libm_fma`** pins case 1 everywhere, accepting slow libm
   calls where hardware is absent — the only way to get bit-identical
   `mul_add` results across build targets.

Floats always have `mul_add` available (case 1 or 2 by target). Agents:
never assume `a.mul_add(b, c)` equals `a * b + c` bit-for-bit; it does only on
FMA targets (single vs double rounding).

### Float functions (`T: num_traits::Float`)

- Unary, lane-wise: `abs`, `sqrt`, `recip`, `floor`, `ceil`, `round`, `trunc`,
  `fract`.
- Binary: `min`, `max` (NaN handling follows the scalar `Float` methods),
  `copysign`.
- `round_ties_even` exists only for `f32`/`f64` vectors (not reachable
  generically through `num_traits::Float`).
- **No transcendental functions** (exp/log/sin/…): use the escape hatch
  [`Simd::map`](crate::Simd::map), e.g. `v.map(f64::exp)` — it vectorizes the
  same way everything else does.

### Comparisons and masks

- `simd_lt`/`simd_le`/`simd_gt`/`simd_ge` (need `PartialOrd`) and
  `simd_eq`/`simd_ne` (need `PartialEq`) return `Simd<bool, N>` masks.
- Mask-side (`Simd<bool, N>` only): `select(if_true, if_false)` — lane-wise
  pick; `any_true()` / `all_true()`.
- Value-side (any of the six types): `mask_select(mask, other)` — keep
  `self` where `mask` is true, `other` where false.

### Reductions

- [`Simd::reduce_sum`](crate::Simd::reduce_sum) / [`Simd::reduce_product`](crate::Simd::reduce_product) —
  all lanes, **accumulated in plain lane order** (sequential fold, not a
  pairwise/tree reduction; float results depend on that order).
- `reduce_sum_n(n)` / `reduce_product_n(n)` — first `n` lanes; panic if
  `n > N`. Useful for masked-out tails.
- No horizontal min/max/and/or reductions.

### Slice load/store

- [`Simd::from_slice`](crate::Simd::from_slice)(src) — first `N` elements;
  **panics** if `src.len() < N`.
- `from_slice_pad(src)` — same, zero-filling lanes past `src.len()` (tail-safe
  load).
- `store_slice(&self, dst)` — writes `N` lanes; **panics** if `dst.len() < N`.
- `store_slice_partial(&self, dst)` — writes `min(dst.len(), N)` lanes
  (tail-safe store).

### Unsafe constructor

- [`Simd::uninit`](crate::Simd::uninit)() — `MaybeUninit`-backed scratch
  vector; **caller must fully initialize every lane before reading**; element
  must admit arbitrary bit patterns (floats/integers yes, types with niches
  no). Intended for hot loops that write-then-read, e.g. accumulate-then-reduce.

## Cargo features

| Feature | Default | Effect |
| --- | --- | --- |
| `complex` | off | `num-complex` dep + `c64x*`/`c32x*` aliases |
| `half` | off | `half` dep + `f16x*`/`bf16x*` aliases |
| `use_libm_fma` | off | pins fused `mul_add` on every target (see above) |

## Performance model

- Build with `-C target-cpu=native` or a fixed level (`x86-64-v3`/`-v4`);
  without it the code is still correct but scalar-ish. A single binary is not
  expected to be fast on multiple micro-architectures — that is the accepted
  trade (see README "Scopes" and `docs/adr/0001-no-dispatch-autovectorization-fallback.md`).
- This crate sets `[profile.dev] opt-level = 2`, so tests and dev runs already
  vectorize without `--release`.
- Codegen is statically verified by `scripts/check_asm.py`: it compiles
  probe functions at several target levels and asserts expected vector
  instruction families (details and CI wiring in README; rationale in
  `docs/adr/0004-codegen-verification-by-assembly-matching.md`).
- Evidence the approach works end-to-end: `examples/blis_matmul.rs` reaches
  OpenBLAS-parity (~1.2 TFLOP/s) GEMM on an AVX-512 host using only these
  portable types.

## What this crate does not provide

So agents do not hallucinate APIs: no intrinsics or inline asm, no runtime
CPU dispatch, no shuffles/permutes/interleaves/transposes/swizzles/rotates,
no lane insert/extract beyond `Index`, no horizontal min/max, no
saturating/wrapping arithmetic, no gather/scatter or masked load/store, no
`core::simd` (`std::simd`) interop, no serde, no lane iterator, no
`FromIterator`, no transcendental math, no `PartialOrd` operators.

## Non-obvious points (agent checklist)

1. `<`/`>`/`==` operators do not exist on vectors — comparisons are the
   `simd_*` methods returning masks; branch on them with `any_true`/`all_true`.
2. `select` lives on the mask; `mask_select` lives on the value. Same
   operation, two receivers.
3. `fma_from`'s receiver is the accumulator (`self += b * c` shape), unlike
   `mul_add`.
4. `mul_add` results differ across build targets unless `use_libm_fma` is on.
5. `reduce_sum` folds in lane order — not pairwise; do not assume
   associativity-exact float sums.
6. Only `splat` and `as_array` are `const fn`; everything else is
   runtime (though always `#[inline(always)]`).
7. Alignment is capped at 64 bytes even for >64-byte vectors (`f64x16`).
8. `bool` lanes: arithmetic traits do not apply — use bitwise/logical ops and
   `select`/`mask_select`.
9. Tail handling: `from_slice_pad` + `reduce_sum_n` + `store_slice_partial`
   are the intended ragged-edge toolkit.
10. Custom element types need: `Copy` + `num_traits::Num` for arithmetic,
    `num_traits::Float` for float math, the element's own `core::ops` impls
    for bitwise.

## Repository pointers

- `README.md` — project scope, FMA notes, codegen verification, usage example.
- `examples/blis_matmul.rs` — BLIS-style GEMM built only on these types.
- `tests/basic.rs`, `tests/ops.rs`, `tests/aligned.rs` — behavioral ground
  truth, including target-dependent `mul_add` tests and feature-gated
  complex/`half` tests.
- `docs/adr/0001`–`0005` — architecture decisions: no-dispatch rationale,
  single generic type, aligned aliases, assembly-based codegen verification,
  mul-add fallback policy.
- `scripts/check_asm.py` (+ `scripts/asm/probes.rs`) — static codegen checker;
  `--dump`/`--hist` for inspecting emitted assembly.
- `src/lib.rs` — `Simd` and shared impls; `src/aligned.rs`, `src/alias.rs`,
  `src/ops.rs`, `src/reduce.rs`, `src/memory.rs` — one module each for the
  surfaces described above.
