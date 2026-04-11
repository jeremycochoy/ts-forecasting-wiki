# Head-to-Head Leaderboard

This page collects benchmark numbers *as reported* in the source
papers. Every cell cites its paper and table. The goal is not to
produce a single global ranking — there isn't one — but to surface
places where several TS foundation models reported on the same suite
so readers can judge relative performance without re-running
experiments.

Throughout this page, "zero-shot" means *the paper's own claim*. See
[methodology-caveats.md](methodology-caveats.md) for why that claim
often deserves asterisks. A dash (`—`) means the paper did not report
that number.

## 1. GIFT-Eval (zero-shot, 97 tasks over 55 datasets)

GIFT-Eval has quickly become the de-facto zero-shot yardstick for new
TS-FMs. Chronos-2 is the most recent single-paper survey on this
suite (Oct 2025); it reports both its own numbers and the leaderboard
values for the main contenders. Metrics are *skill score* (geometric
mean improvement over Seasonal Naive; higher is better) under
Weighted Quantile Loss (WQL, probabilistic) and MASE (point).

| Model | GIFT-Eval WQL skill score (%) | GIFT-Eval MASE skill score (%) | Source |
|---|---|---|---|
| Chronos-2 (120M) | 51.4 | 30.2 | Chronos-2, Table 4 (arXiv:2510.15821) |
| TimesFM-2.5 | 51.0 | 29.5 | Chronos-2, Table 4 (reporting the GIFT-Eval leaderboard) |
| TiRex | 50.2 | 27.6 | Chronos-2, Table 4 |
| Toto-1.0 | 48.6 | 25.2 | Chronos-2, Table 4 |
| Moirai-2.0 | 48.4 | 27.2 | Chronos-2, Table 4 |
| COSMIC | 44.5 | 20.8 | Chronos-2, Table 4 |
| Sundial | 44.1 | 25.0 | Chronos-2, Table 4 |
| TabPFN-TS | 43.1 | 16.6 | Chronos-2, Table 4 |
| Chronos-Bolt | 42.6 | 19.2 | Chronos-2, Table 4 |
| Seasonal Naive | 0.0 | 0.0 | Chronos-2, Table 4 |

A separate reading of GIFT-Eval comes from the Sundial paper
(Feb 2025), which uses the raw geometric-mean-relative MASE/CRPS
(lower is better) over the 23-dataset / 97-config version and quotes
the official GIFT-Eval leaderboard for its baselines:

| Model | MASE (rel., lower better) | CRPS (rel., lower better) | Rank (avg over 97 configs) | Source |
|---|---|---|---|---|
| Sundial | 0.673 | 0.472 | 9.062 | Sundial, Table 2 (arXiv:2502.00816) |
| TimesFM | 0.680 | 0.465 | 8.237 | Sundial, Table 2 |
| TabPFN-TS | 0.748 | 0.480 | 8.268 | Sundial, Table 2 |
| PatchTST (full-shot) | 0.762 | 0.496 | 10.052 | Sundial, Table 2 |
| Chronos | 0.786 | 0.551 | 14.309 | Sundial, Table 2 |
| Moirai | 0.809 | 0.515 | 10.175 | Sundial, Table 2 |
| iTransformer (full-shot) | 0.802 | 0.524 | 11.320 | Sundial, Table 2 |
| N-BEATS (full-shot) | 0.842 | 0.689 | 21.381 | Sundial, Table 2 |
| Seasonal Naive | 1.000 | 1.000 | 26.175 | Sundial, Table 2 |

Notes on comparability: the two GIFT-Eval tables above are drawn
from different paper cut-offs (Feb 2025 vs. Oct 2025) and use
different aggregation conventions (skill score vs. relative GM).
They rank their own model first; the crossing points — Sundial,
TimesFM, Moirai, Chronos — agree directionally that zero-shot
foundation models already close most of the gap to strong
task-specific deep models like PatchTST and iTransformer on
GIFT-Eval. See [methodology-caveats.md](methodology-caveats.md)
for the skill-score vs. GM-relative conversion.

## 2. Chronos Benchmark II (27 zero-shot tasks, mostly short context)

Chronos Benchmark II is the original zero-shot evaluation introduced
alongside Chronos (Mar 2024). It is "pure" zero-shot in the sense
that the datasets were explicitly held out during Chronos and
Chronos-2 pretraining.

| Model | Chronos-II WQL skill score (%) | Chronos-II MASE skill score (%) | Source |
|---|---|---|---|
| Chronos-2 (120M) | 46.6 | 26.5 | Chronos-2, Table 5 (arXiv:2510.15821) |
| TimesFM-2.5 | 42.4 | 23.3 | Chronos-2, Table 5 |
| TiRex | 41.7 | 22.2 | Chronos-2, Table 5 |
| Toto-1.0 | 41.9 | 22.3 | Chronos-2, Table 5 |
| Moirai-2.0 | 40.9 | 19.8 | Chronos-2, Table 5 |
| Chronos-Bolt | 39.3 | 20.4 | Chronos-2, Table 5 |
| COSMIC | 36.7 | 18.1 | Chronos-2, Table 5 |
| TabPFN-TS | 32.6 | 10.5 | Chronos-2, Table 5 |
| Sundial | 24.1 | 9.5 | Chronos-2, Table 5 |
| Seasonal Naive | 0.0 | 0.0 | Chronos-2, Table 5 |

## 3. fev-bench (98 tasks: 32 univariate / 26 multivariate / 42 covariates)

fev-bench (Shchur et al. 2025) is the newest of the three suites and
the first to stress multivariate and covariate-informed forecasting
at scale. Chronos-2 reports the full table. Metric is skill score
under Scaled Quantile Loss (SQL), higher better.

| Model | fev-bench SQL skill score (%) | Median runtime (s) | Source |
|---|---|---|---|
| Chronos-2 (120M) | 47.3 | 3.6 | Chronos-2, Table 3 (arXiv:2510.15821) |
| TiRex | 42.6 | 1.4 | Chronos-2, Table 3 |
| TimesFM-2.5 | 42.3 | 16.9 | Chronos-2, Table 3 |
| Toto-1.0 | 40.7 | 90.7 | Chronos-2, Table 3 |
| TabPFN-TS | 39.6 | 305.5 | Chronos-2, Table 3 |
| Moirai-2.0 | 39.3 | 2.5 | Chronos-2, Table 3 |
| COSMIC | 39.0 | 34.4 | Chronos-2, Table 3 |
| Chronos-Bolt | 38.9 | 1.0 | Chronos-2, Table 3 |
| Sundial | 33.4 | 35.6 | Chronos-2, Table 3 |
| Stat. Ensemble | 20.2 | 690.6 | Chronos-2, Table 3 |
| AutoARIMA | 20.6 | 186.8 | Chronos-2, Table 3 |
| Seasonal Naive | 0.0 | 2.3 | Chronos-2, Table 3 |

The Chronos-2 paper further splits fev-bench into univariate,
multivariate and covariates subsets. On the covariates subset the
gap opens: Chronos-2 reaches 47.0 SQL skill score while TabPFN-TS
(the next best covariate-aware model) reaches 40.0 and TiRex
(covariate-blind) is at 39.9 (Chronos-2, Figure 4b). This gap is
the clearest modern demonstration that native covariate support
matters on real tasks.

## 4. Monash zero-shot (29-dataset in-distribution aggregated MAE)

The Monash archive is older and has been reported on by most early
TS-FMs. Moirai-MoE (Oct 2024) gives a clean aggregate (lower = better;
geometric-mean MAE normalized by Seasonal Naive).

| Model | Aggregated MAE (Monash, rel. Seasonal Naive, ↓) | Source |
|---|---|---|
| Moirai-MoE-Base | 0.63 | Moirai-MoE, Figure 3 (arXiv:2410.10469) |
| Moirai-MoE-Small | 0.65 | Moirai-MoE, Figure 3 |
| Chronos-Large (*not ZS) | 0.70 | Moirai-MoE, Figure 3 |
| Chronos-Base (*not ZS) | 0.71 | Moirai-MoE, Figure 3 |
| TimesFM (*not ZS) | 0.77 | Moirai-MoE, Figure 3 |
| Chronos-Small (*not ZS) | 0.78 | Moirai-MoE, Figure 3 |
| Moirai-Large | 0.89 | Moirai-MoE, Figure 3 |
| Moirai-Base | 0.90 | Moirai-MoE, Figure 3 |
| Moirai-Small | 0.94 | Moirai-MoE, Figure 3 |
| WaveNet (task-specific) | 0.91 | Moirai-MoE, Figure 3 |
| FFNN (task-specific) | 0.89 | Moirai-MoE, Figure 3 |
| DeepAR (task-specific) | 0.94 | Moirai-MoE, Figure 3 |
| Seasonal Naive | 1.00 | Moirai-MoE, Figure 3 |

`*not ZS` = the paper marked these models as having the Monash
evaluation datasets in their pretraining corpus, so "zero-shot" on
Monash is not strictly honest for Chronos and TimesFM — Moirai-MoE
flags this explicitly with an asterisk in its Figure 3 caption. This
is the single biggest methodological footnote on the entire Monash
leaderboard.

## 5. Long-horizon LTSF benchmark (ETT / Weather / ECL, avg over {96,192,336,720})

The LTSF suite (ETTh1/2, ETTm1/2, Electricity, Weather) is the old
"long-sequence forecasting" leaderboard from the 2021-2023 era. It
still matters because most TS-FM papers report it. Metrics are MSE /
MAE averaged over the four canonical horizons. Lower is better.

| Model | ETTh1 MSE | ETTh2 MSE | ETTm1 MSE | ETTm2 MSE | Weather MSE | ECL MSE | Source |
|---|---|---|---|---|---|---|---|
| Sundial-Large (zero-shot) | 0.395 | 0.334 | 0.331 | 0.254 | 0.238 | 0.166 | Sundial, Table 1 (arXiv:2502.00816) |
| Sundial-Base (zero-shot) | 0.411 | 0.333 | 0.336 | 0.258 | 0.234 | 0.169 | Sundial, Table 1 |
| Time-MoE-Ultra (zero-shot) | 0.412 | 0.371 | 0.356 | 0.277 | 0.250 | — | Time-MoE, Table 3 (arXiv:2409.16040) |
| Time-MoE-Large (zero-shot) | 0.394 | 0.405 | 0.376 | 0.284 | 0.234 | — | Time-MoE, Table 3 |
| Timer-XL (zero-shot) | 0.404 | 0.347 | 0.373 | 0.273 | 0.256 | 0.174 | Sundial, Table 1 (quoting Timer-XL) |
| Moirai-Large (zero-shot) | 0.480 | 0.367 | 0.422 | 0.329 | 0.264 | 0.186 | Sundial, Table 1 |
| Moirai-Base (zero-shot) | 0.417 | 0.362 | 0.406 | 0.311 | 0.287 | 0.187 | Sundial, Table 1 |
| Chronos-Large (zero-shot) | 0.588 | 0.455 | 0.555 | 0.295 | 0.279 | 0.204 | Sundial, Table 1 |
| Chronos-Base (zero-shot) | 0.591 | 0.405 | 0.645 | 0.310 | 0.292 | 0.214 | Sundial, Table 1 |
| TimesFM (zero-shot) | 0.473 | 0.392 | 0.433 | 0.328 | — | — | Sundial, Table 1 |
| TTM-A (5M, zero-shot) | 0.400 | 0.333 | 0.362 | 0.252 | 0.231 | 0.192 | TTM, Table 1 (arXiv:2401.03955) |

The same LTSF suite reported by MOIRAI in its own paper (zero-shot)
gives numbers broadly consistent with the above for MOIRAI, e.g.
ETTh1 MSE 0.434 base / 0.510 large, ETTm1 MSE 0.381 base / 0.390
large, ECL 0.188 base (MOIRAI, Table 6, arXiv:2402.02592).

The Moirai-Large *rises* relative to Moirai-Base on ETTh1
(0.434 → 0.510, MOIRAI Table 6; Sundial Table 1 reproduces a
similar inversion at 0.480) — an unusual scaling footprint that is
discussed in [methodology-caveats.md](methodology-caveats.md).

## 6. Moirai-MoE zero-shot CRPS table (10 held-out datasets)

The Moirai-MoE paper contains one of the cleanest single-table
comparisons because it evaluates Chronos, TimesFM, Moirai and
Moirai-MoE on the *same* 10 datasets with the *same* metrics. Lower
is better. Asterisks on Chronos and TimesFM columns in the source
table mark datasets these models saw during pretraining.

Aggregated across all 10 datasets (geometric mean relative to
Seasonal Naive):

| Model                            | Average CRPS (rel.) | Average MASE (rel.) | Source              |
| -------------------------------- | ------------------- | ------------------- | ------------------- |
| Moirai-MoE-Base                  | 0.478               | 0.651               | Moirai-MoE, Table 2 |
| Moirai-MoE-Small                 | 0.497               | 0.670               | Moirai-MoE, Table 2 |
| TimesFM (*partial leakage)       | 0.488               | 0.689               | Moirai-MoE, Table 2 |
| Chronos-Base (*partial leakage)  | 0.499               | 0.656               | Moirai-MoE, Table 2 |
| Chronos-Large (*partial leakage) | 0.500               | 0.660               | Moirai-MoE, Table 2 |
| Chronos-Small (*partial leakage) | 0.543               | 0.694               | Moirai-MoE, Table 2 |
| Moirai-Large                     | 0.514               | 0.729               | Moirai-MoE, Table 2 |
| Moirai-Base                      | 0.520               | 0.736               | Moirai-MoE, Table 2 |
| Moirai-Small                     | 0.578               | 0.798               | Moirai-MoE, Table 2 |
| iTransformer (full-shot)         | 0.540               | 0.767               | Moirai-MoE, Table 2 |
| PatchTST (full-shot)             | 0.549               | 0.808               | Moirai-MoE, Table 2 |
| Seasonal Naive                   | 1.000               | 1.000               | Moirai-MoE, Table 2 |

## 7. MOIRAI probabilistic zero-shot CRPS (paper's own table)

MOIRAI Table 5 reports CRPS on six held-out datasets against strong
tuned baselines (PatchTST, TiDE, TFT, DeepAR). Selected values:

| Dataset | MOIRAI-Large CRPS (zero-shot) | PatchTST CRPS (full-shot) | DeepAR CRPS (full-shot) | Source |
|---|---|---|---|---|
| Electricity | 0.050 | 0.052 | 0.065 | MOIRAI, Table 5 (arXiv:2402.02592) |
| Solar | 0.406 | 0.518 | 0.431 | MOIRAI, Table 5 |
| Walmart | 0.098 | 0.082 | 0.121 | MOIRAI, Table 5 |
| Weather | 0.051 | 0.059 | 0.132 | MOIRAI, Table 5 |
| Istanbul Traffic | 0.112 | 0.112 | 0.108 | MOIRAI, Table 5 |
| Turkey Power | 0.036 | 0.054 | 0.066 | MOIRAI, Table 5 |

On five of six datasets MOIRAI-Large ties or beats the best
full-shot baseline while being strictly zero-shot. Walmart is the
single clean loss.

## Related wiki pages

- [state-of-the-art.md](state-of-the-art.md) — interprets these
  tables into per-use-case recommendations.
- [methodology-caveats.md](methodology-caveats.md) — the asterisks
  and footnotes these tables hide.
- [efficiency-and-cost.md](efficiency-and-cost.md) — the complementary
  "is it cheap to run?" axis.
- [../datasets-benchmarks/gift-eval.md](../datasets-benchmarks/gift-eval.md)
- [../datasets-benchmarks/monash-archive.md](../datasets-benchmarks/monash-archive.md)
