# rh-lean-check

Independent re-run of a **fresh Lean kernel replay and axiom audit** for the Lean formalization that Levent Alpöge announced on 2026-11-20 as a proof of the Riemann Hypothesis (and GRH for Dirichlet L-functions), found working with Claude.

This repository contains **no proof code**. It only holds two GitHub Actions workflows and the results they write back.

> **Status:** claimed proof, not yet independently confirmed. A passing run here is evidence about the formal artifact, not a mathematical review.

## What is checked

| Question | How | Where the answer lands |
|---|---|---|
| Is the headline statement the real RH? | `Main.lean` proves Mathlib's own `RiemannHypothesis` (pinned Mathlib `0df444a…`); no redefinition in the project | checked statically, see below |
| Any `sorry`, extra `axiom`, `native_decide`, `implemented_by`, `unsafe`, kernel-skipping options? | text scan of all 175,187 `.lean` files | checked statically, see below |
| Does the Lean kernel accept every declaration? | `lake env leanchecker --fresh Main` (replays all declarations in an empty environment) | `results/verify_tail.log`, `results-build-missing/build_tail.log` |
| Which axioms do the five headline theorems use? | axiom audit of `Main` (expected: `propext`, `Classical.choice`, `Quot.sound` only) | `results/axioms.txt`, `results-build-missing/axioms.txt` |

## Why this is independent even though it reuses compiled shards

The workflows call the [Omega Institute verifier](https://github.com/the-omega-institute/riemann-hypothesis) (`./verify`). It downloads their published, **SHA-256-checked** Linux x86-64 build shards plus Mathlib's official cache. The verdict, however, comes from `leanchecker --fresh` running **on this repository's own GitHub runner**: it re-checks every declaration with the pinned Lean kernel from an empty environment. A wrong or tampered `.olean` would be rejected there. Reused shards save compile time; they are not trusted for correctness.

Still trusted: the Lean 4.33.1 kernel, Mathlib at the pinned commit, `leanchecker`, and GitHub's runners.

## Workflows

- **`verify.yml`**: runs hourly. Records how many of the 8,490 required shards the Omega Institute has published (`results/status.txt`). Once all are published, it runs the full restore, fresh kernel replay and axiom audit once, and writes `results/FINAL`.
- **`build-missing.yml`**: one attempt started on 2026-11-20 without waiting: restore the published shards, **compile the unpublished rest locally** (`./verify --build-missing Main`), then replay and audit. It may hit the 6-hour job limit or run out of memory or disk; the reason is recorded in `results-build-missing/FINAL` either way.

Permissions: the jobs that run Lean code have read-only access to this repository. Only the final `record` job, which commits result files, can write.

## Static findings (2026-11-20)

Done on the unmodified mirror [CocoSgt/riemann-hypothesis-lean](https://github.com/CocoSgt/riemann-hypothesis-lean), which the Omega Institute reports matches the published tarball:

- `theorem riemannHypothesis : RiemannHypothesis := rhOfAll`, where `RiemannHypothesis` is Mathlib's definition; no declaration in the project redefines `RiemannHypothesis` or `riemannZeta`.
- 0 occurrences of `axiom` declarations, `sorry`, `admit`, `native_decide`, `implemented_by`, `@[extern]`, `unsafe`, `ofReduceBool`, `debug.skipKernelTC`.
- Metaprogramming present: ~20 tactic macros, two term-copying elaborators, one `addDecl`-based copy command (`CutCptCopy.lean`), and `run_cmd` in `BuildAid/` (library-suggestion deny list only). All of these go through the kernel; none is a soundness bypass by itself.
- Top of the proof chain: `RiemannHypothesis ⇐ EveryCharacter` (L(χ,s) ≠ 0 for Re s > ½, all Dirichlet χ) via the functional equation, ~20 lines; `EveryCharacter ⇐` Galois-representation input from a theta-lift argument (`galoisRepresentations_of_thetaLift`). The structure `OpenLiteratureStatements` is instantiated with proofs inside the project, not assumed.

Shard coverage at the first run (2026-11-20, about 20:57 UTC): 5,891 / 8,490 published.

## Limits

- A passing kernel replay shows the formal statement is derivable in Lean + Mathlib. It does not show that the argument is understood, and it is not peer review.
- This setup was prepared with the help of Claude (Anthropic). A Claude model was involved in finding the claimed proof, so treat this repository as a reproducible procedure to re-run, not as an independent judgment.

## Sources

- Announcement: https://alpo.ge/remark.pdf
- Omega Institute verifier and Comparator setup: https://github.com/the-omega-institute/riemann-hypothesis
- Unmodified mirror: https://github.com/CocoSgt/riemann-hypothesis-lean

All credit for the proof belongs to Levent Alpöge (with Claude).
