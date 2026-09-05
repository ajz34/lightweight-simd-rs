# Changelog

Initial release notes are hand-written; from v0.1.0 onward this file is
maintained by release-plz (`.github/workflows/release-plz*.yml`), which
prepends a section per published version from conventional commits.

## v0.1.0 -- 2026-09-05

Initial release: lightweight, generic fixed-lane vector types whose
operations are plain portable loops that the compiler auto-vectorizes —
no intrinsics, no inline assembly, no runtime CPU dispatch. SIMD-level
efficiency comes from building with `-C target-cpu=native` (or a fixed
level such as `x86-64-v3`); correctness never depends on it.

Enhancement

- `Simd<T, N>` over `[T; N]` with arbitrary const-generic lane counts:
  lane-wise arithmetic (vector/vector and vector/scalar, plus assign
  forms), bitwise and shift operators, `mul_add`/`fma_from`,
  float functions (`abs`/`sqrt`/`min`/`max`/rounding/...), comparisons
  producing `Simd<bool, N>` masks, mask selection, horizontal reductions
  (`reduce_sum`, `reduce_product`, `_n` variants), and slice load/store
  including partial-transfer tail forms.
- Enforced-alignment siblings (`Aligned4`–`Aligned64`) sharing the full
  operation surface, with concrete aliases `f64x8`, `f32x16`, `i8x64`,
  mask aliases, and feature-gated `complex` (`c64x*`/`c32x*`) and `half`
  (`f16x*`/`bf16x*`) alias sets.
- `mul_add` evaluation follows the compilation target: fused `MulAdd`
  on hardware-FMA targets (x86-64-v3+, all aarch64), separated mul+add
  elsewhere; the `use_libm_fma` feature pins the fused form for
  bit-identical results across targets (see `docs/adr/0005`).
- Static codegen verification (`scripts/check_asm.py`): compiles probe
  functions at several `-C target-cpu` levels and asserts expected
  vector instruction families, with populated x86-64 and aarch64
  expectation tables; emitted assembly lands in `target/asm/`.
- CI workflows: fmt/clippy lint, test suite, and both asm matrices
  (x86-64 on ubuntu, aarch64 on Apple Silicon runners).
- BLIS-style GEMM example (`examples/blis_matmul.rs`): cache-blocked
  4096³ `f64` multiplication over `f64x8` reaching ~1.2 TFLOP/s
  (OpenBLAS parity on the development host), still without intrinsics.
- Documentation: README with usage example, and a one-file API overview
  (`src/guide.md`) rendered into rustdoc as the `guide` module, written
  for users and AI coding agents.

Scope notes

- The originally planned intrinsics escape hatch
  (`with-intrinsics`/`with-full-intrinsics` features) is dropped: the
  assembly-verification framework shows the portable code vectorizes
  everywhere it matters (see `docs/adr/0002`, README "Verifying codegen").
