# rh-lean-check

Independent re-run of a **fresh Lean kernel replay and axiom audit** for the Lean formalization that Levent Alpöge announced on 2026-11-20 as a proof of the Riemann Hypothesis (and GRH for Dirichlet L-functions), found working with Claude.

**Map of the proof:** <https://raissaramaruchti.github.io/rh-lean-check/>: import layers, the proof spine from `riemannHypothesis` down to the theta-lift input, where the 17.4 M lines sit, and a searchable catalogue of the 561 `GlobalTheory.Published.*` literature statements.

This repository contains **no proof code**. It only holds two GitHub Actions workflows and the results they write back.

> **Status (24 Nov 2026):** independently verified as a formal proof by Andrew Sutherland (MIT): all modules compile, `#print axioms` on the five headline theorems shows only `propext`, `Classical.choice`, `Quot.sound`, and `leanchecker` plus the independent checkers nanoda, con-leche, con-ron and lean4lean accept. Lean's default kernel (22 Nov) and paranoid kernel (23 Nov, 72 h on one thread) also accept the full export, and the standalone Comparator v4.33.0 reports "Your solution is okay!" (23 Nov; its build step was satisfied by a stub and the challenge module was Sutherland's own compile). The full `lake comparator` v4.36.0 run has passed its paranoid-kernel and nanoda stages; its last stage, Lean's default kernel (about 48 h single-threaded), is still running. A sandboxed rebuild (no network, read-only source) finished with zero errors and produced an export byte-for-byte identical to the published one (sha256 `5ea74b91…244e`); 175,186 of 175,187 `.olean` files match the manifest; the one exception compiles non-deterministically, which Sutherland traced to a harmless 38-byte difference in a bound-variable name (a known Lean issue fixed in v4.36.0; the export is unaffected). Report: <https://math.mit.edu/~drew/grh/>. Michael Rubinstein (Waterloo) reports a further independent check (compile, axioms, kernel replay, nanoda) with no errors: <https://www.math.uwaterloo.ca/~mrubinst/alpoge_claude_grh/>. On 21 Nov 2026 a joint statement of the Lean and Mathlib communities, endorsed by the Lean FRO and the Mathlib Initiative, certified the statement and proof as correct: <https://leanprover-community.github.io/blog/posts/riemann-hypothesis/>. Rubinstein has also posted a working draft, written by AI models at his request, explaining the argument for the zeta function (v1, 164 pp., 21 Nov: <https://www.math.uwaterloo.ca/~mrubinst/alpoge_claude_grh/zeta-paper.pdf>; v2, 265 pp., 22 Nov, on his page). It does not yet prove the key inequality (M); v2 reduces (M), and with it RH, to thirteen numbered hypotheses. Alpöge has not yet published a human-readable account. On 24 Nov 2026 the Clay Mathematics Institute moved the Riemann Hypothesis from “Unsolved” to “Active” on its Millennium Problems page; that means under consideration, not solved (Clay requires publication in a qualifying outlet, two years of scrutiny and broad acceptance): <https://www.claymath.org/millennium/riemann-hypothesis/>. Runs in this repository are further independent replications. A passing kernel check is evidence about the formal artifact, not a mathematical review.

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
