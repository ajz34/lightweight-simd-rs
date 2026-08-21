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

