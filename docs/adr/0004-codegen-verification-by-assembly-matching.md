# Codegen verification by assembly matching

The crate's core promise — no intrinsics, no runtime dispatch, speed from
auto-vectorization under the requested `target-cpu` — is verified statically:
`scripts/check_asm.py` compiles the black-box-fenced probe functions in
`scripts/asm/probes.rs` to assembly for several `-C target-cpu` rows
(`x86-64-v2`, `-v3`, `-v4`, `native`, a generic-SSE2 baseline row used as a
negative control, and one `-C llvm-args=-fp-contract=fast` variant; aarch64:
`generic`/`native` plus the same fp-contract and `use_libm_fma` variant
rows) and asserts instruction families with width-aware minimum counts.
REQUIRED checks
fail the run; forms that cannot hold at a target level are INFORMATIONAL
notes. The baseline row asserts vector instructions are *absent*, proving the
harness can go red.

## Considered Options

- **Runtime microbenchmarks** (criterion-style): measure end-to-end effects
  but cannot attribute a regression to scalarization (cache/allocator noise),
  are flaky, and cannot check codegen for target levels the host cannot run.
- **Interactive inspection** (cargo-show-asm and equivalents): repeatable only
  by a human reading output; adds an install dependency and version drift.
- **Trust `-O` and code review**: the common portable-crate stance; silent
  vectorization regressions on toolchain upgrades.
- **Scripted family-regex matching (chosen)**: deterministic, dependency-free
  (python stdlib), works for target levels the host cannot execute (emission
  is compile-only), and its tiered expectations make "correct but
  known-scalar" results explicit instead of noisy failures.

## Consequences

- Expectations are instruction *families* (VEX/EVEX prefix optional, ps/pd
  domain alternatives, blend vs k-mask selection), so ordinary codegen-shape
  drift stays green — confirmed live when a nightly toolchain bump mid-phase
  kept the whole matrix green; width-aware counts still catch partial
  vectorization.
- Architectures are data-driven in the script (`ARCHES` maps each arch to
  its rows and expectation table). x86_64 rows walk the ISA ladder
  (baseline -> v2 -> v3 -> v4). aarch64 is populated too: NEON is the
  baseline there, so `generic`/`native` rows pin the 128-bit `v.2d`/`v.4s`
  shapes and the fp-contract (`genc`) / `use_libm_fma` (`genf`) variant
  rows, with `expr_fma`'s forbid serving as the fused-codegen negative
  control (there is no scalar-only row to forbid vectors). An arch whose
  expectations are still empty runs in informational mode — probes compile
  and report, nothing fails — until a table is authored on that arch
  (`--hist` prints per-probe instruction histograms, `--dump` shows single
  bodies). Cross-emission from another host is deliberately not wired up.
- Emission goes through a helper package (`scripts/asm/probes`,
  `cargo rustc --emit asm` with per-row `RUSTFLAGS` and a dedicated target
  dir): manual `rustc --extern` cannot pair modern cargo's split
  stub-rlib/full-rmeta artifacts, and stale rlibs from earlier toolchains
  are indistinguishable by name or mtime.
- Recorded known-scalar results (x86_64): `mul_add` at v2 under the
  `use_libm_fma` feature (no FMA ISA → scalar libm `fma` calls; the default
  separated mode is a hard mul+add check there), i64 shifts before AVX-512
  (GPR `shlq`/`sarq`, semantics verified numerically), `f16` arithmetic via
  `half` (soft), strict-order reductions at v2/v3 (scalar chain),
  `store_slice_partial` (runtime tail length). Recorded known-scalar results
  (aarch64): comparisons/mask select (branch-free per-lane `fcmp`/`fcsel`,
  not `fcmgt`+`bsl`), `f16` via `half` (soft-float GPR work at `generic`,
  per-lane scalar `fadd h` at `native`), strict-order reductions (scalar
  chain), `store_slice_partial` (runtime tail length).
- Probe-design constraint: masks must be consumed by a vector operation —
  comparisons feeding only reductions get scalarized by LLVM.
- Adding or changing an operation should update the probe table in the same
  change. The script runs manually (`python3 scripts/check_asm.py`,
  `--arch` to select, `--dump`/`--hist` to inspect) and in CI on every push
  to main and PR: x86_64 on an ubuntu runner, aarch64 on a GitHub-provided
  Apple Silicon macOS runner (the workflows pin the runner arch, since the
  cross-emission guard exits 0 and would otherwise mask a skipped matrix).
