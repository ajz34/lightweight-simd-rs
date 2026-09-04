# mul_add fallback: separated mul+add by default, libm-fused optional

`mul_add`/`fma_from` lower to `llvm.fma`, which is a hardware FMA
instruction on FMA-capable targets (no flags needed) but eight per-vector
calls into libm's correctly-rounded software `fma` on targets without the
FMA ISA. Measured on this host (f64x8, dependent accumulation chain,
rustc nightly + glibc 2.43):

| codegen | fused path (`llvm.fma`) | separated `a*b+c` |
|--|--|--|
| `x86-64-v2` (no FMA ISA) | 12.8 ns/vector-op (libm calls) | 2.8 ns/vector-op |
| `x86-64-v3` (FMA) | 0.8 ns/vector-op (`vfmadd`) | 3.0 ns/vector-op |

So the libm fallback is ~4.6x slower than separating the operations, while
the fused form is also ~3.6x *faster* than separated on FMA-capable targets.
The two forms also differ numerically: single vs double rounding (e.g.
`(1 + 2^-52) * (1 - 2^-53) + 2^-53` yields `1 + 2^-52` fused but `1.0`
separated).

Since `-C target-cpu` is not visible to `cfg` (only `-C target-feature`
is), the strategy cannot be selected per-target inside the crate; it is a
compile-time policy. The non-default cargo feature `use_libm_fma` selects
it:

- **default (feature off)**: `self * b + c` as separate multiply and add.
  Bounds `Mul + Add`. Never calls libm. Fuses back to a hardware FMA
  instruction on capable targets when the user compiles with
  `-C llvm-args=-fp-contract=fast`.
- **`use_libm_fma` (feature on)**: the element's `num_traits::MulAdd`.
  Bounds `MulAdd`. Single rounding; hardware FMA with no extra flags; slow
  correctly-rounded libm fallback on pre-FMA targets.

## Considered Options

- **Always `llvm.fma` (the previous behavior)**: simplest and always
  single-rounded, but ships a ~4.6x slowdown on every pre-FMA build.
- **Always separated**: fast everywhere, but silently loses the fused
  contract `mul_add` nominally promises, with no recourse.
- **`cfg(target_feature = "fma")` split**: impossible — `-C target-cpu`
  (the usual way users enable FMA) does not set `cfg(target_feature)`.
- **Feature-gated policy (chosen)**: one global switch, matching the
  crate's no-runtime-dispatch principle; both modes verified by the
  assembly framework (rows `v2f`/`v3f` for the feature, `v3c` for the
  fp-contract recovery path).

## Consequences

- Default builds: `mul_add` has two roundings per lane and — without
  `fp-contract=fast` — runs as separate mul/add even on FMA hardware,
  where the fused form would also be faster. The crate docs recommend the
  flag or the feature when rounding or heavy multiply-add throughput
  matters.
- The element bounds of `mul_add`/`fma_from` differ per mode (`Mul + Add`
  vs `MulAdd`); the methods are available for any `Num`-like element in
  the default mode.
- `tests/ops.rs` checks each mode against its own reference, plus a
  bit-pattern test (`0x3ff0000000000001` vs `0x3ff0000000000000`) proving
  the feature genuinely switches rounding.
- The assembly framework encodes all of it: default rows assert
  mul+add counts (including v2, now a hard check instead of a fallback
  note), `v3c` asserts fusion under the flag, `v3f` asserts `vfmadd` under
  the feature, `v2f` reports the 8/16 libm calls, and plain `a*b+c`
  expressions stay unfused in every mode except `v3c`.
