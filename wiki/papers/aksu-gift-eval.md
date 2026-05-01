# GIFT-Eval: A Benchmark for General Time Series Forecasting Model Evaluation

> **Short name:** `aksu-gift-eval` · **arXiv:** [2410.10393](https://arxiv.org/abs/2410.10393) · **PDF:** [local](../../papers/aksu-gift-eval_2410.10393.pdf) · **Date:** 2024-10 (v2 2024-11) · **Venue:** preprint (Salesforce AI Research)

**Authors:** Taha Aksu, Gerald Woo, Juncheng Liu et al. (Salesforce AI Research; Juncheng Liu also at NUS)

## Abstract
GIFT-Eval is a zero-shot forecasting benchmark designed as a foundation-model leaderboard rather than a forecasting archive. The test split contains 23 datasets / 144,000 series / ~177M observations across 7 domains and 10 frequencies, with prediction lengths spanning short / medium / long horizons and both univariate and multivariate settings. The paper ships a paired non-leaking pretraining split (`GiftEvalPretrain` on HuggingFace, 88 datasets / ~240B observations) carved out of LOTSA so that fair comparisons against `Moirai` are possible, and reports a comprehensive 17-baseline evaluation including five statistical models, eight deep-learning models, and four foundation models (`TimesFM`, `Chronos` S/B/L, `Moirai` S/B/L, `VisionTS`). The authors retrain the entire `Moirai` family from scratch on `GiftEvalPretrain` to remove the partial leakage in LOTSA against the test split, and document the leakage delta in §F.2 by also reporting the original LOTSA-trained checkpoints under the alias `Moirai-Leakage`. PatchTST emerges as the most consistent generalist; `Moirai-Large` (retrained on `GiftEvalPretrain`) is the best foundation model in aggregate Rank.

## Key contributions
- **Benchmark design.** A 23-dataset / 7-domain / 10-frequency / 97-configuration test suite explicitly framed as a zero-shot foundation-model leaderboard, with fixed splits, prediction lengths, context lengths, and metric definitions so that papers from different teams can be compared without re-running each other's experiments.
- **Paired non-leaking pretraining corpus.** `Salesforce/GiftEvalPretrain` (88 datasets, 71 univariate + 17 multivariate, ~240B observations, ~4.5M series) is carved out of LOTSA with the test datasets removed. It is the only large-scale TS pretraining corpus that comes with an explicit "clean against this benchmark" guarantee (cf. [LOTSA](../datasets-benchmarks/lotsa.md), [Time-300B](../datasets-benchmarks/time-300b.md), [Time Series Pile](../datasets-benchmarks/time-series-pile.md), where leakage audits are partial or absent).
- **Two-metric protocol.** All scores are reported as MAPE (point) and CRPS (probabilistic), each normalized against Seasonal Naive on the same series. CRPS is approximated as mean weighted quantile loss across nine equispaced quantiles — the same quantity later papers call WQL. A Rank metric (average ranking across all 97 configs by CRPS) accompanies the two relative scores.
- **17-baseline empirical study.** The first cross-paradigm leaderboard that puts statistical (Naive, Seasonal Naive, Auto_ARIMA, Auto_Theta, Auto_ETS), deep-learning (DeepAR, TFT, TiDE, N-BEATS, PatchTST, iTransformer, DLinear, Crossformer), and foundation models (`TimesFM`, `Chronos` S/B/L, `Moirai` S/B/L, `VisionTS`) on a single shared protocol.
- **Leakage audit (§F.2).** The headline `Moirai_S/B/L` numbers in the main tables are retrained from scratch on `GiftEvalPretrain` only — making this the first published TS-FM trained purely on a leakage-free GIFT-Eval-aligned corpus. Table 23 contrasts these with the LOTSA-trained `Moirai-Leakage` checkpoints on the eight datasets where LOTSA and GIFT-Eval test overlap, demonstrating that leakage benefit grows with prediction length and is concentrated on a few datasets (`loop_seattle`, `m_dense`) rather than uniformly distributed.

## Methodology at a glance
Each test dataset is split with the last 10% of every series held out. Foundation models are run zero-shot from public checkpoints (HF repos `Salesforce/moirai-1.1-R-{small,base,large}`, `amazon/chronos-t5-{tiny,small,base}`, `google/timesfm-1.0-200m`); deep-learning baselines train one instance per dataset under a hyperparameter search defined in Appendix A; statistical baselines are fit per series with `gluonts`. The evaluation harness uses `gluonts` for both metrics. Multivariate datasets are forecast natively only by `Moirai` (the only foundation model in the lineup that supports it); the other three foundation models marginalize per-channel. Eight A100 GPUs are used for the deep-learning grid; foundation-model inference is single-GPU. All `Moirai` retraining runs follow the public `Moirai` recipe with GIFT-Eval-aligned context lengths (1000–8000 search range).

### Sizes

Benchmark paper — **no new model architecture introduced.** The retrained `Moirai-S/B/L` checkpoints follow the [Moirai](./moirai.md) recipe (Woo et al. 2024 Table 4) verbatim with only the pretraining corpus changed (LOTSA → `GiftEvalPretrain`); the `(L, d_model, d_ff, heads, d_kv, params, patch, context)` tuple for those checkpoints is documented on [moirai.md](./moirai.md). The 17-baseline lineup (statistical / deep-learning / foundation-model) follows the architectures of the cited works without modification.

### Headline aggregate (Table 10, all 97 configs, lower is better)
| Model | MAPE | CRPS | Rank |
|---|---|---|---|
| PatchTST (full-shot) | 0.860 | 0.563 | **5.72** |
| Moirai-Large (`GiftEvalPretrain` only) | 0.864 | 0.593 | 5.99 |
| iTransformer (full-shot) | 0.985 | 0.626 | 6.37 |
| TFT (full-shot) | 1.12 | 0.596 | 6.76 |
| Moirai-Base (`GiftEvalPretrain` only) | 1.01 | 0.648 | 7.57 |
| Moirai-Small (`GiftEvalPretrain` only) | 0.882 | 0.642 | 7.81 |
| Chronos-Large | 0.930 | 0.629 | 8.31 |
| Chronos-Base | 0.940 | 0.637 | 8.32 |
| TimesFM | 1.25 | 0.683 | 8.55 |
| Chronos-Small | 0.928 | 0.638 | 9.16 |
| N-BEATS | 1.08 | 0.773 | 13.26 |
| Seasonal Naive | 1.00 | 1.00 | 16.64 |

PatchTST and Moirai-Large (retrained) are statistically tied at the top of Rank; the foundation models with leakage in their pretraining (`Chronos`, `TimesFM`) sit in the middle.

## Why it matters
GIFT-Eval is the first benchmark in TS forecasting to ship its own paired non-leaking pretraining corpus. That single design choice closes the credibility gap that plagued earlier zero-shot claims on Monash and LTSF, where every TS-FM paper had to add an asterisk for partial leakage. The retrained-`Moirai` reference numbers also give the field its first clean answer to the question *"what does training on `GiftEvalPretrain` alone get you?"* — Moirai-Large at MAPE 0.864 / CRPS 0.593, sitting essentially tied with full-shot PatchTST. Every modern leaderboard ([leaderboard.md §1](../benchmarks/leaderboard.md#1-gift-eval-zero-shot-97-tasks-over-55-datasets)) is now expressed in GIFT-Eval skill scores derived from this protocol.

## Strengths
- The pretrain–test disjointness is constructed by exclusion from LOTSA rather than asserted post-hoc, which removes the dispute about which datasets count as "in-distribution."
- Reporting the LOTSA-trained `Moirai-Leakage` alongside the retrained `Moirai` in §F.2 + Table 23 is the only published per-dataset quantification of TS-FM leakage in the literature.
- Wide methodological lattice: domain, frequency, prediction length, and number-of-variates are each given an aggregate table (Tables 6–9), so the benchmark surfaces *where* a model wins rather than just whether it wins on average.
- Open artifacts: `Salesforce/GiftEvalPretrain` on HuggingFace, the Moirai training code in `uni2ts`, the leaderboard at `https://github.com/SalesforceAIResearch/gift-eval`. Re-running the entire benchmark is operationally tractable for a small team.

## Limitations and open critiques
- The protocol uses **MAPE** (not MASE) for point accuracy. This diverges from the public GIFT-Eval leaderboard (which reports MASE skill score) and from Monash / Chronos Benchmark II (which use MASE). Cross-paper readers must convert MAPE↔MASE manually or refer to the leaderboard form, and there is no formal map between the two.
- The CRPS used here is mean weighted quantile loss across nine equispaced quantiles — i.e. what `Chronos-2` and successors call **WQL**. Different aggregation choices (per-series scaling for `fev-bench`'s SQL; mean pinball with non-uniform quantile grid for older Chronos) make cross-paper comparison subtle (see [methodology-caveats.md §2](../benchmarks/methodology-caveats.md)).
- Only `Moirai` is retrained from scratch on the clean corpus. The headline numbers for `TimesFM` and `Chronos` are reported with the original (partially leaked) pretraining checkpoints, since neither team published a training pipeline that the GIFT-Eval authors could re-run end-to-end. So the leakage delta is quantified only for `Moirai`.
- The benchmark predates the second wave of TS-FMs (`Sundial`, `Chronos-2`, `Moirai-2.0`, `Toto`, `TiRex`, `Timer-S1`, `TabPFN-TS`). Its public leaderboard absorbs those papers' self-reported numbers, but the paper-internal evaluation is locked to the 17-baseline set above.
- `Moirai-Base` is *worse* than `Moirai-Small` in aggregate Rank (7.57 vs 7.81 reverses for MAPE: 1.01 vs 0.882) — a documented inverse-scaling artifact in the original `Moirai`-1 family even after retraining on the clean corpus, foreshadowing the same pattern in [Moirai 2.0](./moirai-2.md) (cf. [scaling-laws.md](../concepts/scaling-laws.md)).

## Follow-up work and dialogue
GIFT-Eval has become the canonical SOTA yardstick for every TS-FM paper since late 2024. [Moirai-MoE](./moirai-moe.md) (Oct 2024) reports GIFT-Eval improvements over the dense `Moirai` baseline; [Sundial](./sundial.md) (Feb 2025) reports GIFT-Eval as one of two primary suites; [Moirai 2.0](./moirai-2.md) (Nov 2025) evaluates exclusively on GIFT-Eval and uses `GiftEvalPretrain` as the public-data backbone of its 36M-series mixture; [Chronos-2](./chronos-2.md) (Oct 2025) tabulates the GIFT-Eval public leaderboard as its main reference table; [Timer-S1](./timer-s1.md) (Mar 2026) explicitly scrubs GIFT-Eval leakage from TimeBench and reports MASE/CRPS on the public leaderboard. The benchmark's "carve a clean pretraining split out of a leaky big corpus" template was replicated by `Timer-S1`'s TimeBench-curation pipeline.

## Reproducibility
- **Open artifacts:** code + leaderboard at `github.com/SalesforceAIResearch/gift-eval`; pretraining corpus at `huggingface.co/datasets/Salesforce/GiftEvalPretrain`.
- **Code:** public Python harness using `gluonts` for metrics and the `uni2ts` codebase for `Moirai` retraining.
- **Data:** test split is downloaded via the harness; pretraining corpus is HuggingFace-hosted (~140 GB Parquet).
- **Compute:** eight A100 GPUs for the deep-learning hyperparameter grid; `Moirai` retraining runs (small/base/large) inherit the public `Moirai` recipe.
- **Deployment footprint:** —

## When to cite this paper
Cite this paper as the canonical reference for: (a) the GIFT-Eval benchmark protocol (23 test datasets, 97 configs, MAPE+CRPS relative to Seasonal Naive); (b) the paired non-leaking `GiftEvalPretrain` corpus; (c) the first published `Moirai` family retrained on a leakage-free corpus, including its Table 10 headline numbers; (d) the §F.2 / Table 23 leakage audit, which is the only per-dataset quantification of LOTSA→GIFT-Eval leakage in the literature.

## In the knowledge graph
- **Cluster:** Pre/parallel-FM benchmark methodology (not part of the eight-cluster TS-FM taxonomy).
- **Methodology hub:** [GIFT-Eval](../datasets-benchmarks/gift-eval.md), [evaluation-benchmarks](../datasets-benchmarks/evaluation-benchmarks.md), [leakage-map](../datasets-benchmarks/leakage-map.md).
- **Related concepts:** [zero-shot forecasting](../concepts/zero-shot-forecasting.md), [scaling laws](../concepts/scaling-laws.md) (the `Moirai_S < Moirai_B` inversion is one of the published negative-scaling data points).
- **See also:** [Moirai](./moirai.md) (the model retrained for the headline leakage-free numbers), [Moirai 2.0](./moirai-2.md) (the modern Moirai successor evaluated exclusively on this benchmark), [Chronos-2](./chronos-2.md) (most recent leaderboard tabulation), [Timer-S1](./timer-s1.md) (current pre-trained-model SOTA on GIFT-Eval).

## Related wiki pages
- [datasets-benchmarks/gift-eval.md](../datasets-benchmarks/gift-eval.md)
- [datasets-benchmarks/leakage-map.md](../datasets-benchmarks/leakage-map.md)
- [benchmarks/leaderboard.md](../benchmarks/leaderboard.md)
- [benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md)
- [benchmarks/training-a-small-model.md](../benchmarks/training-a-small-model.md)
- [evaluation/protocols.md](../evaluation/protocols.md)
