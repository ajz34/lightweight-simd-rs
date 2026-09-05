# Lightweight SIMD (rust)

This project will implement lightweight, generic fixed-array types in rust.

## Scopes

- Array types will be similar to SIMD types (f64x8, f32x16, etc). Generic may allow non-standard types (like bf16) to be also be representable.
- **WILL NOT** use SIMD intrinsics/asm (unless extremely necessary). No runtime SIMD instruction dispatch by CPU flags.
- All architectures should work correctly. To enable SIMD-level efficiency, we depends on rust's compiler backend (usually LLVM) with `target-cpu=native`.
- Cleaner code is better. "Fallback" scheme in other SIMD crates is actually fast if `target-cpu=native`. We do not expect a single binary to be efficient on multiple CPU micro-architectures.
- We are expecting to implement some common traits (like `num`) for scientific computation.
- We mostly stick on stable rust.

## Usage

Add the dependency, and build with a `target-cpu` setting so the compiler
auto-vectorizes (correctness does not depend on it):

```toml
[dependencies]
lightweight-simd = "0.1"
```

```sh
RUSTFLAGS="-C target-cpu=native" cargo build --release
```

```rust
use lightweight_simd::{f64x8, Simd};

// typed aliases (enforced alignment) or the generic Simd<T, N>
let a = f64x8::from([1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0]);
let b = f64x8::splat(2.0);

let c = a.mul_add(b, f64x8::one()); // a*b + 1, fused on FMA targets
assert_eq!(c.reduce_sum(), 80.0);

let mask = a.simd_gt(f64x8::splat(4.0)); // comparisons -> masks
assert_eq!(a.mask_select(mask, b).to_array(), [2.0, 2.0, 2.0, 2.0, 5.0, 6.0, 7.0, 8.0]);

let mut buf = [0.0; 8];
a.store_slice(&mut buf); // slice load/store

// any element type and lane count
let u = Simd::from([10u32, 20, 30]);
assert_eq!((u * 3).to_array(), [30, 60, 90]);
```

Lane-wise `+ - * /` (vector-vector or vector-scalar), float functions (`abs`,
`sqrt`, `floor`, `min`, ...), comparisons (`simd_lt`, `simd_gt`, ...) with mask
selection, reductions (`reduce_sum`, `reduce_product`), and slice loads
(`from_slice`, `from_slice_pad`) are shared by `Simd` and the aligned alias
types; transcendental functions go through `map` (e.g. `v.map(f64::exp)`).
The `complex` and `half` features add the `c64x*`/`c32x*` and `f16x*`/`bf16x*`
aliases. Full API: `cargo doc --open`.

## Notes

This project is somehow inspired by crate fearless_simd, pulp. However, the aim of this project is completely different to those projects (dispatchable SIMD).

This project will probably extensively use AI code agent for development.

## More detailed scopes

- We will try to check if this lightweight, non-intrinsic code can emit correct SIMD assembly with `target-cpu` supported intrinsics. We will try to make code clean and easy, but extensive testing to make sure effectiveness.
- The originally planned intrinsics escape hatch (cargo features `with-intrinsics` / `with-full-intrinsics`) is dropped: the assembly-verification framework shows the portable code vectorizes everywhere it matters (see "Verifying codegen"). If a genuine vectorization gap ever appears on an operation users care about, the right-sized fix is a per-op `cfg(target_feature)` patch (note: `cfg(target_feature)` follows `-C target-cpu`), not a general intrinsics layer.

## Notes on FMA

- `mul_add`/`fma_from` evaluation (see `docs/adr/0005`):
  - default: fused `MulAdd` — single rounding, one hardware FMA instruction per register, no flags — where the compilation target has hardware FMA (x86 with `-C target-cpu` of `x86-64-v3`+ or `native`; all aarch64). On targets without (generic x86-64, `x86-64-v2`, unknown architectures): separated mul+add — two roundings per lane, never the slow libm software `fma` (measured ~4.6x slower than separated there), and fuses back to a hardware FMA instruction under `-C llvm-args=-fp-contract=fast`.
  - `use_libm_fma` (non-default feature): pins the fused form on every target — single-rounding results that are bit-identical across build targets, at the cost of libm software `fma` calls where hardware FMA is absent.

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
`mul_add` policy of `docs/adr/0005` is encoded too. On x86-64: v3+ rows assert
hardware `vfmadd`, v2/baseline rows assert separated mul+add (baseline also
forbids `vfmadd` as a negative control), the `v3c` row shows the separated
expression fusing under `-C llvm-args=-fp-contract=fast`, and the `v2f`/`v3f`
rows cover the `use_libm_fma` feature (libm calls vs hardware `vfmadd`). On
aarch64 — where NEON's 128-bit vectors and `fmla` are baseline at every
`-C target-cpu` — the `generic`/`native` rows assert the `v.2d`/`v.4s` shapes,
and the `genc`/`genf` rows carry the same two variants (expressions fuse under
fp-contract; `use_libm_fma` stays hardware `fmla`, never libm calls). Emitted
assembly lands in `target/asm/` (gitignored). See
`docs/adr/0004-codegen-verification-by-assembly-matching.md`.

Architectures are data-driven in the same script: each architecture is one
entry (compile rows + expectation table); x86-64 and aarch64 tables are
populated, and an architecture without expectations yet runs in informational
mode — probes compile and report, nothing fails. `--hist` prints per-probe
instruction histograms and `--dump` single bodies, which is how a table for a
new architecture gets authored on that architecture's host.

Both populated tables run in CI (`.github/workflows/`): the x86-64 matrix on
an ubuntu runner and the aarch64 matrix on a GitHub-provided Apple Silicon
macOS runner, alongside fmt/clippy and the test suite.

## Example: BLIS-style GEMM

`examples/blis_matmul.rs` demonstrates that the portable types compose into a
cache-blocked BLIS-style matrix multiplication: fixed 4096×4096×4096 `f64`,
row-major, non-transposed, with operand packing, an 8×16 register-blocked
micro-kernel over `f64x8`, a 2D rayon task grid (a dev-dependency, example
usage only), and one shared pre-packed copy of `B` — still without any
intrinsics. Blocking targets x86-64 AVX-512F (asserted at startup) and is
measured on the development host (AMD Ryzen 9 9955HX, 16 Zen 5 cores, DDR5,
balance power mode), where it reaches ~1.2 TFLOP/s — on par with OpenBLAS
on the same machine:

```sh
RUSTFLAGS="-C target-cpu=native" cargo run --release --example blis_matmul
```


