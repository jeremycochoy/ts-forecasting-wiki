# Evaluation Benchmarks in Current Use

This page catalogs every evaluation benchmark a TS foundation model
is expected to report on, with its purpose, scale, zero-shot status,
and whether it is actively used or retained only for historical
continuity.

If you want head-to-head *numbers*, see
[../benchmarks/leaderboard.md](../benchmarks/leaderboard.md). This
page is about the suites themselves — *what* each benchmark measures
and *why* you would report it.

## 1. The current SOTA benchmarks (2025-2026)

### GIFT-Eval

- **Paper.** Aksu et al., "GIFT-Eval: A Benchmark For General Time
  Series Forecasting Model Evaluation", arXiv:2410.10393.
- **Year introduced.** 2024.
- **Scale.** 23 test datasets / 144k series / 177M observations, 7
  domains, 10 frequencies (GIFT-Eval Tables 3-5).
- **Zero-shot status.** Zero-shot **by construction** — paired with a
  separate `GiftEvalPretrain` (88 datasets / 4.5M series / 230B obs,
  disjoint from the test split) so models can be trained and
  evaluated fairly.
- **Metrics.** MASE (point), WQL and CRPS (probabilistic), aggregated
  as skill scores vs Seasonal Naive (higher better).
- **Access.** `Salesforce/GiftEval` on HuggingFace; leaderboard at
  `Salesforce/GIFT-Eval` HuggingFace Space.
- **Use today.** The de facto SOTA zero-shot leaderboard. Every new
  TS-FM paper reports it. See [gift-eval.md](gift-eval.md) for the
  full dataset list.

### fev-bench

- **Paper.** Shchur et al., 2025 (AWS).
- **Scale.** 98 tasks: 32 univariate / 26 multivariate / 42
  covariate-informed.
- **Metric.** Scaled Quantile Loss skill score (higher better), plus
  median runtime per task, plus a **Leakage (%)** column reporting
  per-baseline overlap with the eval set ([Chronos-2](../papers/chronos-2.md)
  Table 3: Chronos-2 0%, TiRex 1%, TimesFM-2.5 8%, Toto-1.0 8%,
  Moirai-2.0 28%, Sundial 1%, Chronos-Bolt 0%, TabPFN-TS 0%,
  COSMIC 0%).
- **Use today.** The current leaderboard for **covariate-informed
  and multivariate forecasting** — the axis GIFT-Eval under-weights.

### Chronos Benchmark II

- **Paper.** [Chronos](../papers/chronos.md) (Ansari et al., 2024),
  Appendix B / Table 3.
- **Scale.** 27 zero-shot datasets, short-context emphasis.
- **Zero-shot status.** Strict — explicitly held out during
  [Chronos](../papers/chronos.md) and
  [Chronos-2](../papers/chronos-2.md) pretraining. Chronos paper
  footnote 5 says the residual leakage risk is "minimal given that
  the datasets bear no overlap beyond high-level conceptual
  categorization."
- **Metrics.** MASE skill score, WQL skill score.
- **Use today.** Still widely reported, especially for short-context
  univariate probabilistic work. Note that the 2024-era numbers are
  comparable across `Chronos` family papers only if the evaluation
  protocol matches.

## 2. Historical benchmarks still in common use

### Monash Time Series Forecasting Archive

- **Paper.** Godahewa et al., 2021 (NeurIPS Datasets & Benchmarks).
- **Scale.** Dozens of datasets across energy, transport, weather,
  retail, finance, tourism. ~29 datasets are the "standard Monash
  benchmark" subset used by most TS-FMs.
- **Zero-shot status.** **Complicated.** Monash ships fixed splits,
  but because LOTSA, Time-300B, the Time Series Pile, and the
  [Chronos](../papers/chronos.md) corpus all include Monash datasets,
  most foundation models that report Monash numbers are **in-corpus,
  not zero-shot**. [Moirai-MoE](../papers/moirai-moe.md) Figure 3
  asterisks TimesFM*, Chronos-Small/Base/Large* for this reason:
  "asterisks (*) to mark the methods that used the evaluation
  datasets here in their pretraining corpora."
- **Metrics.** MAE scaled by Seasonal Naive, geometric-mean
  aggregated. Classical and supervised deep-learning baselines are
  published alongside.
- **Use today.** Universally reported for historical continuity.
  Always read the asterisk footnote.

### LTSF (Long-Horizon Long-Term Series Forecasting)

- **Source.** Informer / Autoformer / TSLib literature (2021-2023).
- **Datasets.** ETTh1, ETTh2, ETTm1, ETTm2 (electrical transformer
  temperatures), Weather, Electricity, Traffic.
- **Canonical horizons.** {96, 192, 336, 720} prediction steps.
- **Metrics.** Raw MSE and MAE averaged over the four horizons. No
  standardized probabilistic metric.
- **Zero-shot status.** In-corpus for most modern TS-FMs because the
  ETT / Weather / ECL datasets are heavily used in pretraining.
  Papers reporting "zero-shot LTSF" usually mean the specific
  test-window was held out; the series itself was seen during
  training.
- **Use today.** Still reported by every TS-FM paper, but treat the
  numbers as "long-horizon recall of seen series" rather than a
  zero-shot claim.

### TSLib family

Same ETT / Weather / ECL / ILI as LTSF, with the original PatchTST /
TimesNet / Autoformer evaluation conventions. Mostly obsolete for
new TS-FMs; still cited by the LLM-reprogramming subfield
([Time-LLM](../papers/time-llm.md), [GPT4TS](../papers/gpt4ts.md)).

## 3. Benchmarks for non-forecasting tasks

These are adjacent to TS-FM research but not used for forecasting
evaluation:

- **UCR / UEA Archives.** Univariate and multivariate classification
  — used by [MOMENT](../papers/moment.md), UniTS,
  [TSPulse](../papers/tspulse.md).
- **TSB-UAD / TSB-AD.** Time-series anomaly-detection benchmarks.
  TSPulse reports on TSB-AD; MOMENT reports on TSB-UAD.
- **LTSF-Imputation.** Imputation on the LTSF datasets with missing
  windows.

## 4. Retired or narrow-use benchmarks

- **M1 / M2 / M3 Competitions.** Listed in dataset cards for
  historical completeness; only "M4 (hourly/daily/monthly/...)"
  survives in current benchmarks.
- **"LTSF96/192/336/720" reported on Traffic.** The Traffic dataset
  is now considered saturated — most TS-FMs achieve near-identical
  MSE — so it rarely discriminates between models.
- **Paper-specific suites.** TimeGPT's 300k-series internal suite,
  MOIRAI's Table 5 "6-dataset CRPS", Moirai-MoE's "10-dataset
  held-out" are useful inside the parent paper but should not be
  quoted as generic benchmarks.

## 5. What a 2026 TS-FM paper is expected to report

Minimum viable set:

1. **GIFT-Eval test** with MASE + WQL skill scores.
2. Either **fev-bench** (if your model supports multivariate or
   covariates) or **Chronos Benchmark II** (if your model is
   strict-zero-shot univariate probabilistic).
3. **Monash** for continuity (asterisk it if your corpus overlaps).
4. **LTSF** for long-horizon sanity-check on ETT/Weather/ECL.

See [../evaluation/what-was-evaluated.md](../evaluation/what-was-evaluated.md)
for the exact evaluation setup each published paper uses.

## 6. Why the leakage audit matters here

Every "zero-shot" claim is paper-relative. Before quoting a number
from this page's benchmarks against a model trained on LOTSA /
Time-300B / Time Series Pile / the Chronos corpus, cross-reference
[leakage-map.md](leakage-map.md) — it tabulates exactly which
pretraining corpus overlaps which benchmark.

The actionable rule: if you want a clean zero-shot GIFT-Eval claim,
train on `Salesforce/GiftEvalPretrain` (the test-disjoint LOTSA
subset) and report on GIFT-Eval test. Everything else is caveat
territory.

## Related wiki pages

- [datasets-benchmarks.md](datasets-benchmarks.md) — data-section hub.
- [gift-eval.md](gift-eval.md) — full per-dataset breakdown.
- [monash-archive.md](monash-archive.md) — the historical benchmark.
- [leakage-map.md](leakage-map.md) — corpus × benchmark matrix.
- [../benchmarks/leaderboard.md](../benchmarks/leaderboard.md) — head-to-head numeric tables.
- [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md) — metric-conversion and asterisk explanations.
- [../evaluation/protocols.md](../evaluation/protocols.md) — zero-shot definitions and leakage catalog.
- [../evaluation/what-was-evaluated.md](../evaluation/what-was-evaluated.md) — per-paper evaluation setup.
