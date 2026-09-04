# mul_add: fused where the target has hardware FMA

`mul_add`/`fma_from` lower to `llvm.fma`, which is a hardware FMA
instruction on FMA-capable targets (no flags needed) but eight per-vector
calls into libm's correctly-rounded software `fma` on targets without the
FMA ISA. Measured on the reference host (f64x8, dependent accumulation
chain, rustc nightly + glibc 2.43):

| codegen | fused path (`llvm.fma`) | separated `a*b+c` |
|--|--|--|
| `x86-64-v2` (no FMA ISA) | 12.8 ns/vector-op (libm calls) | 2.8 ns/vector-op |
| `x86-64-v3` (FMA) | 0.8 ns/vector-op (`vfmadd`) | 3.0 ns/vector-op |

So the libm fallback is ~4.6x slower than separating the operations, while
the fused form is also ~3.6x *faster* than separated on FMA-capable targets.
The two forms also differ numerically: single vs double rounding (e.g.
`(1 + 2^-52) * (1 - 2^-53) + 2^-53` yields `1 + 2^-52` fused but `1.0`
separated).

## Decision

The evaluation is selected at compile time by a cfg predicate:

- **default**: the fused `num_traits::MulAdd` path when the compilation
  target has hardware FMA — `target_feature = "fma"` (which follows
  `-C target-cpu`, so any `x86-64-v3+` or `native` build) or aarch64
  (scalar/vector FMA is baseline there). Otherwise — generic x86-64,
  `x86-64-v2`, and any architecture the predicate does not know — a
  separated `self * b + c` (bounds `Mul + Add`), which never degrades into
  the slow libm calls. Unknown architectures therefore fail safe: fast,
  correct, no libm.
- **`use_libm_fma` feature** (non-default): pins the fused path on every
  target, accepting the libm software `fma` where hardware is absent.
  Use it when results must be bit-identical across build targets.

Decision history: the first cut was always-fused; a second revision made
separated the unconditional default with `use_libm_fma` opting into fused.
That revision was superseded by this cfg-split because under the crate's
recommended build (`-C target-cpu=native`) it shipped the *slower and*
double-rounded form by default.

## Considered Options

- **Always `llvm.fma`**: simplest and always single-rounded, but ships a
  ~4.6x slowdown on every pre-FMA build.
- **Always separated, feature to fuse**: no libm anywhere, but a footgun
  under recommended builds (slower and double-rounded on FMA hardware
  unless the user also sets `-C llvm-args=-fp-contract=fast`).
- **Pure cfg split without the feature**: best codegen with zero
  configuration (this is the default), but no way to pin semantics across
  build targets — a library user does not control the top-level
  `RUSTFLAGS`, so results would silently vary between builds; the feature
  restores that control.
- **Runtime dispatch on hardware detection**: rejected by the crate's
  founding principle (ADR-0001).

## Consequences

- By default `mul_add`'s rounding follows the build target; cross-build
  bit-reproducibility requires `use_libm_fma`.
- The element bounds of `mul_add`/`fma_from` differ per path (`MulAdd`
  vs `Mul + Add`); tests and probes mirror the cfg predicate.
- `tests/ops.rs` checks each path against its own reference, plus a
  bit-pattern test (`0x3ff0000000000001` vs `0x3ff0000000000000`) whose
  expectation follows the same predicate — verified under plain
  `cargo test` (generic x86-64, separated), `--features use_libm_fma`
  (fused), and `RUSTFLAGS=-Ctarget-cpu=native` (fused).
- The assembly framework encodes the split: v3/v4/native rows assert
  `vfmadd` for `mul_add`/`fma_from`, v2 and baseline rows assert
  separated mul+add (baseline additionally forbids `vfmadd` as the
  negative control), the `v3c` row shows the separated expression fusing
  under `-C llvm-args=-fp-contract=fast`, and the `v2f`/`v3f` rows cover
  the feature (libm calls vs hardware `vfmadd`). On aarch64 every row
  asserts hardware `fmla` (baseline ISA there) and forbids libm `fma`
  calls — including under the feature (`genf`), where x86 without the
  FMA ISA would pay them. Plain `a*b+c`
  expressions are unaffected by all of this and stay unfused except under
  fp-contract.
