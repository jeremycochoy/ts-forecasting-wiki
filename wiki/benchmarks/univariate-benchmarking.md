# Benchmarking a Univariate-Only Model Against TS-FM SOTA

This page is a focused practical recommendation for a specific reader:
you have built a forecasting model that handles *one univariate series
at a time* — no multivariate input channels, no cross-series batching,
no panel or covariate handling — and you want to compare it honestly
against the 2024–2026 wave of time-series foundation models. Every
column on the main [leaderboard](leaderboard.md) page assumes the
reader already picked a benchmark; this page is the step before that.

## TL;DR recommendation

> If your model is univariate-only and you want a head-to-head
> comparison against 2024–2026 TS foundation models, run it on
> **Chronos Benchmark II (zero-shot, ~27 datasets)** and the
> **univariate subset of [GIFT-Eval](../datasets-benchmarks/gift-eval.md)**. These are
> the two suites on which every recent TS-FM ([Chronos](../papers/chronos.md),
> [Chronos-2](../papers/chronos-2.md), [TimesFM](../papers/timesfm.md),
> [MOIRAI](../papers/moirai.md), [Moirai-MoE](../papers/moirai-moe.md),
> [Timer](../papers/timer.md), [Timer-S1](../papers/timer-s1.md),
> [Sundial](../papers/sundial.md), [TTM](../papers/ttm.md)) publishes
> numbers, and both are strictly one-series-at-a-time at evaluation
> time. Report **MASE** and **WQL / CRPS** in skill-score form so
> your row lands in the same tables. For historical continuity, add
> **M4** and the univariate tracks of the
> [Monash Archive](../datasets-benchmarks/monash-archive.md) —
> but be explicit about the known pretraining-overlap issue for
> TimesFM and Chronos on Monash (see
> [methodology-caveats.md](methodology-caveats.md) section 1).

Decision tree:

1. Is your model probabilistic (outputs quantiles or samples)?
   - Yes → Chronos Benchmark II + GIFT-Eval univariate, report MASE and WQL.
   - No → Chronos Benchmark II + GIFT-Eval univariate, report MASE only; note you cannot enter the WQL column.
2. Do you want to compare against pre-2024 baselines (N-BEATS, ARIMA, ETS, classical Monash leaderboard)?
   - Yes → add M4 and the univariate Monash tracks.
   - No → stop at (1).
3. Do you need MSE-on-normalized numbers for a deep-TS audience?
   - Yes → see the "benchmarks to skip" section below before you reach for LTSF; the default LTSF numbers are multivariate-input and not a fair reference.

## What "univariate-only" actually means here

In this page, *univariate-only* means the model consumes one channel
per call and produces one channel per call. No cross-series batching
that exchanges information between series. No dynamic or static
covariates. No group / any-variate attention. Each series is an
independent i.i.d. call with its own context window.

This is not a disadvantage against the mainstream TS-FMs. Chronos v1,
Chronos-2's default mode, Sundial, Timer, Timer-S1, TimesFM and
Lag-Llama are all univariate-per-call by design: they forecast one
series at a time and aggregate afterward. The multivariate crowd
([Timer-XL](../papers/timer-xl.md), Chronos-2 with group attention,
MOIRAI's any-variate attention, [Moirai-MoE](../papers/moirai-moe.md))
has extra tricks on top, but those tricks are only exercised on
benchmarks that explicitly contain covariate or panel structure.
On Chronos Benchmark II and the univariate subset of GIFT-Eval,
everyone runs one series at a time, so your architecture sits in the
mainstream.

## Recommended benchmarks, ranked

### 1. Chronos Benchmark II — zero-shot held-out, ~27 datasets

The evaluation suite introduced in the [Chronos](../papers/chronos.md)
paper as its "zero-shot" track: a collection of roughly 27 datasets
that Chronos v1 explicitly held out from pretraining. It has since
become a required row in every major TS-FM paper — Chronos-2, TimesFM,
Moirai-MoE, Timer-S1, Sundial and TTM all publish Benchmark II
numbers. Each dataset is univariate by construction, forecast one
series at a time. Metrics are **WQL** (weighted quantile loss over
the 9-quantile grid) and **MASE**, both usually reported as geometric
mean skill score relative to Seasonal Naive. The dataset split files
and evaluation scripts are distributed with the `chronos-forecasting`
repository, so onboarding is one clone plus one `gluonts` install.

Why pick this: it is the single benchmark on which the largest
number of 2024–2026 SOTA numbers exist side-by-side, and the
one where your univariate setup incurs zero unfairness penalty.

### 2. GIFT-Eval — univariate subset

[GIFT-Eval](../datasets-benchmarks/gift-eval.md) is the newer
cross-paper yardstick (2024–2025), with roughly 97 tasks across
55 datasets covering many domains, frequencies and horizons. Most
tasks are univariate; a smaller number are multivariate. You can
run only the univariate subset and still place in the published
tables, because the leaderboard breaks down per task. It is the
reference benchmark for Chronos-2, Moirai-2, Sundial, TimesFM-2.5
and Timer-S1. Metrics are **WQL skill score** and **MASE skill
score** against Seasonal Naive; the public leaderboard is hosted
as a HuggingFace Space under the Salesforce organization and
accepts community submissions.

Why pick this: it is the most current SOTA comparison venue, and
the only one where 2025 releases (Chronos-2, TimesFM-2.5, Moirai-2,
Sundial, Timer-S1) have all published numbers under the same
protocol at the same time.

### 3. Monash Forecasting Archive — univariate tracks

The [Monash Archive](../datasets-benchmarks/monash-archive.md) is
the historical univariate benchmark: roughly 30 datasets covering
yearly/quarterly/monthly/weekly/daily/hourly frequencies with
per-dataset seasonal periods. It is reported by TimesFM, Chronos v1,
MOIRAI, Moirai-MoE, Lag-Llama and TTM, and it is still the *lingua
franca* for MASE tables. The metric is **MASE** with the
Hyndman–Koehler scaling and the per-dataset seasonal period `m`
specified in the archive.

Caveat: both TimesFM and Chronos v1 have pretraining corpora
that overlap with several Monash datasets. The Moirai-MoE paper's
Figure 3 explicitly asterisks these models to mark that their
Monash results are not strictly zero-shot. This is flagged in
[methodology-caveats.md](methodology-caveats.md) section 1 and in
[protocols.md](../evaluation/protocols.md). You should still run
Monash for historical continuity, but do not report your row as
"zero-shot against zero-shot competitors" — label the competitor
numbers as "in-corpus" where the corresponding paper acknowledges it.

### 4. M4 Competition dataset

The M4 dataset (100,000 univariate series across yearly, quarterly,
monthly, weekly, daily and hourly frequencies) is the pre-TS-FM
classic and still the most-cited univariate benchmark in the
broader forecasting community. Chronos v1 and TTM report M4
numbers; most 2025 TS-FM papers do not. The metric convention is
**OWA** (the overall weighted average of sMAPE and MASE used by
the original M4 competition), alongside raw sMAPE and MASE. Treat
M4 as a comparison against 2018–2023 SOTA rather than 2025 TS-FM
SOTA — you will not find Chronos-2 or Sundial rows here.

### 5. fev-bench

A newer suite used by Chronos-2 and Sundial that emphasizes win-rate
aggregation across many datasets. Lower visibility than GIFT-Eval;
mention it as a secondary row, not a primary one.

## Benchmarks to skip for a univariate-only model

- **LTSF (ETTh1/2, ETTm1/2, Weather, Traffic, ECL, Exchange, ILI).**
  The default deep-TS suite but every dataset is *inherently*
  multivariate, and the published headline numbers (PatchTST,
  iTransformer, TimeMixer, Timer-XL) are multivariate-input models.
  PatchTST publishes a channel-independent row you can compare
  against cleanly; most other SOTA papers do not. Direct comparison
  against their *published* numbers is unfair unless you
  z-normalize per-channel and explicitly label your row
  "channel-independent". See
  [methodology-caveats.md](methodology-caveats.md) section 3.
- **M5 / Tourism.** Both have hierarchical or panel structure; a
  true one-series-at-a-time baseline underperforms by construction
  because it ignores the cross-series signal the benchmark was
  designed to measure.
- **Electricity / Traffic as standalone.** Almost always reported
  multivariate; same problem as LTSF above.

## Metric conventions so your numbers land in the existing tables

Adopt the exact conventions used by the recent SOTA papers:

- **MASE** with the *per-dataset* seasonal period `m` that the
  benchmark specifies (Monash has a canonical `m` per dataset;
  GIFT-Eval uses the standard seasonal period for each frequency).
  Definition and formula in
  [../evaluation/metrics.md](../evaluation/metrics.md#17-mase--mean-absolute-scaled-error).
- **WQL / weighted quantile loss** if your model is probabilistic.
  Aggregate over the quantile grid {0.1, 0.2, ..., 0.9} and weight
  by the absolute target scale, exactly as Chronos and Chronos-2
  do. Definition in
  [../evaluation/metrics.md](../evaluation/metrics.md#23-wql--weighted-quantile-loss).
- **Skill score** (geometric mean of per-dataset skill against
  Seasonal Naive) alongside the raw aggregate. This is the form
  Chronos-2, fev-bench and the GIFT-Eval leaderboard all use, and
  it is the form in which Sundial's CRPS and Chronos-2's WQL can
  be cross-walked (see
  [methodology-caveats.md](methodology-caveats.md) section 2 for
  the identity between skill score, GM-relative error and rank).
- Always report the raw aggregate too, so a future reader can
  convert between conventions.

## Zero-shot vs. fine-tuned vs. in-domain — what to claim

A univariate-only custom model is typically trained per-dataset or
per-series, which is neither strict zero-shot nor strict in-domain
in the TS-FM sense. The honest framing is:

- Use **"fine-tuned per dataset"** or **"full-shot supervised"** as
  your protocol label.
- Compare against each TS-FM's *fine-tuned* column where one
  exists. Chronos, TTM, Moirai-MoE, Timer and Timer-S1 all report
  fine-tuned numbers alongside zero-shot; these are the rows
  directly comparable to yours.
- Do not place your numbers in the *zero-shot* column of a table.
  Even if your model has never seen the test series, the protocol
  definition of "zero-shot" in TS-FM papers is *"pretrained once,
  applied without per-dataset fitting"* — see
  [../evaluation/protocols.md](../evaluation/protocols.md). A
  per-dataset fit is not that.

## Statistical significance to include

The wiki's [what-was-evaluated.md](../evaluation/what-was-evaluated.md)
observation (5) notes that only Chronos-2 currently reports
confidence intervals; small wins of < 2% skill score are not
statistically meaningful without them. To land in the table and
also be defensible:

- **Bootstrap 95% CIs** on MASE and WQL per dataset, matching the
  Chronos-2 protocol. Resample series indices within each dataset.
- **Friedman + Nemenyi** critical-difference diagrams across
  datasets — the convention inherited from the Monash competition
  and used by [Mamba4Cast](../papers/mamba4cast.md).
- **Diebold–Mariano** pairwise tests when you make a specific
  same-dataset claim like "mine beats Chronos-Base on Australian
  electricity." This is the right test for one model-pair on one
  series at the horizon level.

All three tests and their conventions are discussed in
[../evaluation/protocols.md](../evaluation/protocols.md).

## Practical reproducibility shortcut

- The **`chronos-forecasting` GitHub repo** ships the Benchmark II
  split files and evaluation scripts; it is the easiest onboarding
  path because you get the dataset, the split, the baseline
  metrics and the aggregation code in one place.
- **`gluonts`** has loaders for the Monash datasets and the
  standard seasonal-period tables.
- The **GIFT-Eval HuggingFace Space** (hosted by Salesforce) has
  the current leaderboard and the submission instructions.
- **M4** data and the M4 competition baselines are available via
  the `M4-methods` GitHub repository.

Describe each by name in your reproducibility section rather than
copy-pasting URLs — the repos move, but the names are stable.

## Related wiki pages

- [leaderboard.md](leaderboard.md) — normalized head-to-head
  tables on Monash, Chronos Benchmark II, GIFT-Eval, fev-bench and
  the LTSF long-horizon suite.
- [methodology-caveats.md](methodology-caveats.md) — the
  intellectual-honesty layer: pretraining overlap, skill-score vs
  raw-error conventions, context and horizon alignment.
- [../evaluation/metrics.md](../evaluation/metrics.md) — the
  flagship metric reference. Read the MASE, WQL and CRPS entries
  before you run anything.
- [../evaluation/protocols.md](../evaluation/protocols.md) — the
  meaning of zero-shot, in-domain, fine-tuned and full-shot in
  TS-FM papers, plus statistical-significance conventions.
- [../evaluation/what-was-evaluated.md](../evaluation/what-was-evaluated.md)
  — the per-paper table of what each TS-FM in the wiki actually
  reported; use this to build your list of comparison rows.
- [../datasets-benchmarks/gift-eval.md](../datasets-benchmarks/gift-eval.md)
  and [../datasets-benchmarks/monash-archive.md](../datasets-benchmarks/monash-archive.md)
  — the corpus-level descriptions of the two primary suites.
- Relevant paper leaves:
  [../papers/chronos.md](../papers/chronos.md),
  [../papers/chronos-2.md](../papers/chronos-2.md),
  [../papers/timesfm.md](../papers/timesfm.md),
  [../papers/moirai.md](../papers/moirai.md),
  [../papers/moirai-moe.md](../papers/moirai-moe.md),
  [../papers/sundial.md](../papers/sundial.md),
  [../papers/timer-s1.md](../papers/timer-s1.md),
  [../papers/timer.md](../papers/timer.md),
  [../papers/ttm.md](../papers/ttm.md),
  [../papers/lag-llama.md](../papers/lag-llama.md).
