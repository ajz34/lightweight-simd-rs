# No runtime dispatch — rely on auto-vectorization

We ship only the portable, fallback-style implementation and get SIMD performance from
the compiler's auto-vectorization under the requested target CPU (`target-cpu=native`
or equivalent), with no runtime CPU-feature detection and no multiversioned kernels.

## Considered Options

- **Runtime dispatch** (fearless_simd/pulp-style `Level`/token architectures): rejected
  — we do not expect a single binary to run efficiently across multiple CPU
  micro-architectures, so the dispatch machinery, its token types, and the binary-size
  cost buy nothing here.
- **Compile-time `#[target_feature]` multiversioning**: rejected for the same
  single-micro-architecture reason; a non-default `with-intrinsics` cargo feature
  remains the escape hatch for cases where the compiler fails to vectorize.

## Consequences

- Correctness on all architectures is free; speed requires the user to set
  `target-cpu` at build time (release alone is not enough).
- There is no `Level`, arch token, or `dispatch!` in the public API — do not add them.
