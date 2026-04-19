# Dataset Types in Time-Series Foundation Models

This page is a taxonomy. It tells you what kind of dataset you are
looking at so you can judge its role and its limitations. The
categories are not mutually exclusive — a real dataset usually sits
at the intersection of several axes.

## Axis 1 — Pretraining vs evaluation vs both

**Pretraining corpus.** Large aggregated collection that a foundation
model trains on. Examples: [LOTSA](lotsa.md) (27B obs, 9 domains),
[Time-300B](time-300b.md) (309B obs, Nature 90%+),
[TimeBench](timebench.md) (~1T, not released),
[Time Series Pile](time-series-pile.md) (multi-task).

**Evaluation benchmark.** Curated set of test windows with fixed
splits and metrics. Examples: [GIFT-Eval](gift-eval.md) test split
(23 datasets / 144k series), Chronos Benchmark II (27 datasets),
fev-bench (98 tasks), LTSF (ETT/Weather/ECL).

**Both.** Some datasets act as both. `Monash Archive` is a curated
evaluation archive whose raw data also leaks into every major
pretraining corpus. `GiftEvalPretrain` is a pretraining corpus
paired with GIFT-Eval test, explicitly constructed to be disjoint.

## Axis 2 — Real vs synthetic

**Real.** Observed measurements from the physical or economic world.
Examples: ETT (electrical transformer temperatures), Electricity
(UCI), Traffic, M4 (competition series).

**Synthetic from real.** Derived from real data by an augmentation
rule. Examples: TSMix (convex Dirichlet mixtures of real series;
[Chronos](../papers/chronos.md)), Chronos-Mixup, resampling +
Fourier interpolation ([Timer-S1](../papers/timer-s1.md)).

**Fully synthetic.** Generated from a parametric prior with no real
data as input. Examples: KernelSynth (Gaussian processes with random
kernel compositions; [Chronos](../papers/chronos.md)), ForecastPFN
priors, canonical signal families (linear, sinusoidal, exponential,
power, impulse, step). See
[../concepts/synthetic-data-augmentation.md](../concepts/synthetic-data-augmentation.md).

Synthetic data is the only source that is **guaranteed leakage-free**
by construction. Every real-data addition to your corpus raises a
leakage question.

## Axis 3 — Univariate vs multivariate vs covariate-informed

**Univariate.** Each series is a single channel over time (M4,
Monash's univariate family, GIFT-Eval univariate subset).

**Multivariate, channel-independent.** Multiple channels with the
model treating each independently (PatchTST-style).

**Multivariate, cross-variate.** Multiple channels with attention
across variates (ETT multivariate split, MOIRAI any-variate
attention, Chronos-2 group attention, Timer-XL TimeAttention).

**Covariate-informed.** Target series plus known-future exogenous
variables (retail demand + price / promo, energy demand + weather
forecast). fev-bench's 42-covariate-task subset is the first
large-scale covariate-informed benchmark. Very few TS-FMs support
covariates natively; see [benchmarks/state-of-the-art.md](../benchmarks/state-of-the-art.md).

## Axis 4 — Domain

The standard domain partitioning in TS-FM papers covers: **Energy**,
**Transport**, **Climate/Weather/Nature**, **Healthcare**,
**Econ/Fin**, **Sales/Retail**, **Web/CloudOps**. This matches the
[GIFT-Eval](gift-eval.md) 7-domain split.

Two observations worth knowing:

- **Time-300B** is ~90.5% Nature ([Time-MoE](../papers/time-moe.md)
  Table 1), meaning "the largest TS-FM corpus to date" is effectively
  one weather + climate dataset by observation count. Domain
  diversity matters more than raw volume.
- **Web/CloudOps** in the benchmark-test layer is almost entirely
  CloudOps telemetry (BizITObs / Bitbrains in GIFT-Eval, Grid
  Workloads in LOTSA), not web traffic or pageviews. Wikipedia
  pageview data sits in pretraining corpora only.

## Axis 5 — Frequency

The standard frequency axis spans **secondly / minutely / 5-minute /
15-minute / 30-minute / hourly / daily / weekly / monthly / quarterly
/ yearly**. GIFT-Eval's test suite spans 10 of these (secondly is
absent; yearly is present but narrow).

The gotcha: many "frequency" labels are after aggregation. The Kaggle
Web Traffic competition's raw data is daily pageviews; Monash ships
both a daily version and a weekly-aggregated version as two
different datasets.

## Axis 6 — License and accessibility

**Fully public, permissive.** LOTSA, Time-300B, Time Series Pile,
GIFT-Eval, GiftEvalPretrain, Chronos's public dataset card, Monash
Archive. Use freely with attribution.

**Public but mixed per-dataset licenses.** Most aggregated corpora
(LOTSA, Time-300B) are "release of an aggregation" where each
component dataset retains its own license. Read the HuggingFace
dataset card before redistribution.

**Public but gated or not redistributable.** The exact slices used by
TimesFM (22k Google Trends head queries + "all Wikimedia pageviews")
are described in the paper but not released as a fixed file.

**Proprietary.** Moirai 2.0's CloudOps telemetry component
(~2.15M series / ~1.48B daily observations) is Salesforce-internal.
TimeGPT's training corpus is Nixtla-internal. Several commercial
forecasting services rely on proprietary data that cannot be
redistributed.

## Axis 7 — Release form

**Single-file dump.** Monash TSF files, individual UCR/UEA archives.

**HuggingFace dataset.** LOTSA (`Salesforce/lotsa_data`),
GiftEvalPretrain (`Salesforce/GiftEvalPretrain`), Time Series Pile
(`AutonLab/Timeseries-PILE`), Chronos corpus
(`autogluon/chronos_datasets`).

**Sharded Parquet with loader.** Time-300B (released via
`Maple728/TimeMoE` repo), TimeBench (internal 4 TB Parquet, not
released).

**API-only, regenerate per-request.** Google Trends, Wikimedia
pageviews (both raw sources behind TimesFM's Google Trends + Wiki
Pageviews pretraining data). See
[google-trends-data.md](google-trends-data.md) for the specific case.

## Axis 8 — Deprecation and current use

| Status | Dataset family |
|---|---|
| Current SOTA benchmark | [GIFT-Eval](gift-eval.md) test, fev-bench |
| Still widely reported (asterisked) | [Monash](monash-archive.md), Chronos Benchmark II |
| Current standard pretraining | GiftEvalPretrain, LOTSA, Time-300B |
| Legacy benchmark (still reported, conditionally) | LTSF (ETT/Weather/ECL long-horizon tables), TSLib |
| Retired / superseded | M-competition predecessors M1/M2/M3 (only reported in classical baseline columns) |
| Specialty only | TSB-UAD (anomaly), UCR/UEA (classification), TSB-AD |

A TS-FM paper submitted in 2026 is expected to include GIFT-Eval and
either fev-bench or Chronos Benchmark II. Monash is reported for
continuity with 2021–2024 papers and is almost always asterisked
for leakage. See [evaluation-benchmarks.md](evaluation-benchmarks.md)
for the detailed breakdown.

## Related wiki pages

- [datasets-benchmarks.md](datasets-benchmarks.md) — the hub.
- [evaluation-benchmarks.md](evaluation-benchmarks.md) — deep dive on evaluation suites.
- [leakage-map.md](leakage-map.md) — leakage cross-reference.
- [scrub-tools.md](scrub-tools.md) — tooling for leakage removal.
- [../concepts/synthetic-data-augmentation.md](../concepts/synthetic-data-augmentation.md) — KernelSynth / TSMix / ForecastPFN.
- [../benchmarks/training-a-small-model.md](../benchmarks/training-a-small-model.md) — which corpus to pick and why.
- [../evaluation/protocols.md](../evaluation/protocols.md) — zero-shot semantics and leakage catalog.
