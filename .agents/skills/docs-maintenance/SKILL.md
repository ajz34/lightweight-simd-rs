---
name: docs-maintenance
description: Map of which documentation must be updated when crate functionality changes (public API, type aliases, cargo features, mul_add policy, codegen verification, ADRs, performance claims, terminology). Use while or after making such a change, before committing, so README, src/guide.md, rustdoc, ADRs, and the asm checker stay consistent.
---

## When to use

Any change that touches what the crate **offers** or **promises**: public API
surface, aliases, cargo features, numeric semantics (especially `mul_add`),
codegen/verification tooling, CI matrix, performance claims, or domain
terminology. Internal refactors with identical behavior need nothing from this
map except the verification commands at the end.

## Document inventory

| File | Audience / role |
| --- | --- |
| `README.md` | project-facing: scope/stance, usage quick-start, FMA notes, codegen-verification story, BLIS example claim |
| `CHANGELOG.md` | per-release notes; v0.1.0 hand-written, then prepended by release-plz from conventional commits (`release-plz.toml`, `.github/workflows/release-plz*.yml`) |
| `src/guide.md` | one-file API overview; users + AI agents; rendered into rustdoc as the `guide` module (cfg(doc) `include_str!` in `src/lib.rs`) |
| `src/lib.rs` crate docs | overview, trait-bound model, mul_add semantics section, usage example (doctest) |
| `src/alias.rs` | per-alias rustdoc lines (generated via `concat!` in the duplicate table) |
| `src/{aligned,ops,reduce,memory}.rs` module docs | per-module surface summaries |
| `CONTEXT.md` | glossary — normative terminology for every doc above |
| `docs/adr/0001`–`0005` | decision records (rationale lives here, not in README) |
| `Cargo.toml` | feature list + comments (feature docs of record for non-obvious flags) |
| `scripts/check_asm.py`, `scripts/asm/probes.rs` | expectation tables and probes = the codegen claims |
| `.github/workflows/` | CI matrix (lint, tests, asm rows, runners) |
| `tests/` | behavioral ground truth the guide points to |

Do not document in `tmp/` (scratch, untracked-by-convention) or agent memory.

## Change-type → update map

**New/changed public method, operator, or inherent impl** (on `Simd`/aligned
types, usually via the `_mod_` duplicate table):
1. rustdoc on the single macro-instantiated definition (covers all six types).
2. `src/guide.md`: the matching "Operation surface" subsection; the
   element-trait-bound statements; **and the "What this crate does not
   provide" list — remove the entry if it no longer holds**; agent checklist
   if the behavior is non-obvious.
3. `tests/`: coverage (the guide cites tests as ground truth).
4. Arithmetic/compute ops expected to vectorize: probe + expectation rows in
   `scripts/check_asm.py` (see README "Verifying codegen" for the workflow).

**New/changed type alias**: `src/alias.rs` table (doc attr auto-renders);
`src/guide.md` alias list (+ backing-type rule if alignment choice is novel);
README only if headline-worthy.

**Cargo feature add/change**: `Cargo.toml` comment; `src/guide.md` feature
table + affected sections; `src/lib.rs` crate docs if semantics change;
README if user-visible behavior; feature-gated tests; CI leg if the feature
needs one.

**`mul_add`/FMA policy change**: `docs/adr/0005` (or supersede with a new
ADR); README "Notes on FMA"; `src/lib.rs` "Multiply-add semantics" section;
`src/guide.md` multiply-add subsection + agent checklist items; `src/ops.rs`
region comments; `scripts/check_asm.py` FMA rows; `tests/ops.rs`
`mul_add_rounding_mode`.

**Codegen verification change** (new target arch, new probe, table edit):
`scripts/check_asm.py` (+ probes); README "Verifying codegen" paragraph
(row inventory is described there); CI workflow if a new runner/leg is
needed; `src/guide.md` performance section if the stance-level story shifts.

**New/superseded ADR**: `docs/adr/NNNN-slug.md`; update the `0001`–`0005`
range mention in `src/guide.md` "Repository pointers"; README only if it
changes a user-facing promise.

**Performance/claim change** (TFLOP/s figures, host description, CI runners):
README example section (the only place numeric claims live);
`src/guide.md` if repeated there.

**New example or dependency**: `examples/` + README mention + guide
"Repository pointers"; dependency also updates `Cargo.toml` and the guide's
intro dependency line.

**Terminology**: change `CONTEXT.md` glossary first (preferred term +
_Avoid_ list), then sweep the other docs for the avoided terms.

## Cross-file invariants

1. **README usage example ↔ `src/lib.rs` doctest are a manual mirror** —
   deliberately duplicated (not `include_str!`). Any edit to one must be
   copied verbatim into the other; verify with `cargo test --doc`.
2. **`src/guide.md` link rules**: it compiles inside a cfg(doc) module, so
   intra-doc links must resolve with default features *and*
   `--all-features`. Feature-gated aliases (`c64x2`, `f16x*`, …) are plain
   code text, never links — they don't exist in a default-feature doc build.
3. **Glossary compliance**: use `CONTEXT.md` preferred terms in all docs
   (lane-wise not element-wise, mask not boolean vector, splat not
   broadcast, fallback not soft SIMD, partial transfer not masked load).
4. **No personal or absolute paths** in tracked docs (repo-wide AI-agent
   notice in `CLAUDE.md`).

## Verify after doc edits

```sh
cargo doc --no-deps                 # expect 0 warnings
cargo doc --no-deps --all-features  # expect 0 warnings
cargo test --doc                    # README mirror + guide example run
cargo test                          # full suite
```
