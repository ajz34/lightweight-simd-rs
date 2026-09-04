# Lightweight SIMD (rust)

This project will implement lightweight, generic fixed-array types in rust.

## Scopes

- Array types will be similar to SIMD types (f64x8, f32x16, etc). Generic may allow non-standard types (like bf16) to be also be representable.
- **WILL NOT** use SIMD intrinsics/asm (unless extremely necessary). No runtime SIMD instruction dispatch by CPU flags.
- All architectures should work correctly. To enable SIMD-level efficiency, we depends on rust's compiler backend (usually LLVM) with `target-cpu=native`.
- Cleaner code is better. "Fallback" scheme in other SIMD crates is actually fast if `target-cpu=native`. We do not expect a single binary to be efficient on multiple CPU micro-architectures.
- We are expecting to implement some common traits (like `num`) for scientific computation.
- We mostly stick on stable rust.

## Notes

This project is somehow inspired by crate fearless_simd, pulp. However, the aim of this project is completely different to those projects (dispatchable SIMD).

This project will probably extensively use AI code agent for development.

## More detailed scopes

- We will try to check if this lightweight, non-intrinsic code can emit correct SIMD assembly with `target-cpu` supported intrinsics. We will try to make code clean and easy, but extensive testing to make sure effectiveness.
- If intrinsics are not avoidable (default rust compiler does not correctly optimize), then a cargo feature (compile-time instead of runtime-dispatch) `with-intrinsics` will perform optimize by `cfg` attribute with `target-cpu`.
- A cargo feature `with-full-intrinsics` will always introduce intrinsics for optimization, but this is not the best/intended way of using this crate.

## Notes on FMA

- `mul_add`/`fma_from` evaluation modes (see `docs/adr/0005`):
  - default: separated mul+add — two roundings per lane, never calls libm. On FMA-capable targets it still fuses into one hardware FMA instruction when compiled with `-C llvm-args=-fp-contract=fast` (recommended for heavy multiply-add workloads; the fused form is also faster there).
  - `use_libm_fma` (non-default feature): the element's fused `MulAdd` — single rounding per lane; a hardware FMA instruction with no extra flags; a slow correctly-rounded libm `fma` on pre-FMA targets (measured ~4.6x slower than separated there).

## Verifying codegen

A static checker (no runtime benchmarking) verifies the auto-vectorization
promise: it compiles black-box-fenced probe functions for several
`-C target-cpu` levels (`x86-64-v2`, `-v3`, `-v4`, `native`, plus a
generic-SSE2 negative-control row) and asserts the expected vector
instruction families with width-aware counts.

```sh
python3 scripts/check_asm.py                                  # full matrix; exit 1 on any miss
python3 scripts/check_asm.py --dump probe_mul_add_f64x8 v3    # inspect one body
```

Checks that cannot hold at a target level are informational notes rather than
failures (e.g. `f16` arithmetic via the `half` crate stays scalar on x86). The
`mul_add` policy of `docs/adr/0005` is encoded too: default rows assert
separated mul+add, the `v3c` row asserts fusion under
`-C llvm-args=-fp-contract=fast`, and the `v2f`/`v3f` rows cover the
`use_libm_fma` feature (libm calls vs hardware `vfmadd`). Emitted assembly
lands in `target/asm/` (gitignored). See
`docs/adr/0004-codegen-verification-by-assembly-matching.md`.

Architectures other than x86-64 are scaffolded in the same script: each
architecture is one entry (compile rows + expectation table), and an
architecture without expectations yet runs in informational mode — probes
compile and report, nothing fails. `--hist` prints per-probe instruction
histograms and `--dump` single bodies, which is how a table for a new
architecture gets authored on that architecture's host.


