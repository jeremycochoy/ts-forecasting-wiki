# Evaluation Protocols

Protocols are the rules of the game: how data is split, which
windows are scored, how context and horizon are chosen, what
"zero-shot" actually means, and how confidence intervals are
constructed. Protocol choices can swing an apparent ranking by more
than metric choices can, and are under-documented in most TS-FM
papers. This page gathers every protocol knob and names the
convention for each benchmark in the wiki.

## 1. Train / validation / test splits

Time series is not IID. A random shuffle of (window, label) pairs
leaks the future into the training set. The first rule is therefore:
**splits are chronological, never random**. Concretely, for a series
of length `T` the forecasting convention is

```
[   train   |   val   |   test   ]
  1 .. T_tr   T_tr+1   T_va+1 .. T
```

with all hyperparameter selection done on the val window and the
test window held out until the final run. The Monash archive,
GIFT-Eval, fev-bench and Chronos Benchmark II all ship explicit
split indices for each series so that papers reporting on them
cannot accidentally contaminate.

## 2. Rolling-origin evaluation

A single "last `n` steps is the test set" number is one data point.
A **rolling-origin** evaluation slides the split point forward,
producing many forecasts from many origins and reporting the mean
metric across them. Two flavors.

- **Expanding window.** Training set grows as the origin advances;
  validation and test windows slide forward. The most common choice
  for Monash.
- **Sliding window.** Training set has a fixed length and slides
  forward with the origin. Preferred when older data is less
  representative (regime changes).

Rolling-origin evaluation reduces the variance of the reported
metric by `\sqrt{k}` where `k` is the number of origins, and also
exposes temporal robustness — if a model's MASE on origin 1 is 0.7
and on origin 4 is 1.4, something is wrong. Monash runs rolling
windows at a fixed horizon; GIFT-Eval and fev-bench run roll-outs
at multiple horizon lengths per dataset; Chronos Benchmark II uses
a fixed "last horizon `h`" cut per series with no rolling.

## 3. Horizon conventions

**Short horizon** (`h \le 24` hourly or daily). Monash's Hospital,
Traffic and Electricity subsets; the short-term forecasting subset
of TSLib. Classical baselines are strong here and TS-FMs have to
work for their lead.

**Medium horizon** (`h \in \{24, 48, 96\}`). Monash's longer
monthly and daily tracks; Darts; the shorter end of ETT.

**LTSF long horizon**. The canonical LTSF suite reports
`h \in \{96, 192, 336, 720\}` on hourly / 15-minute series (ETTh1,
ETTh2, ETTm1, ETTm2, Electricity, Traffic, Weather, ILI). This is
the regime where MSE-on-normalized-series is the dominant metric
and where TimesFM, Moirai, Time-MoE, Timer-XL, TTM, Sundial, MOMENT,
Time-LLM and GPT4TS all report numbers.

**Horizon-dependent rank reversal is common.** A model that beats
the field at `h = 96` can lose at `h = 720` because autoregressive
decoding compounds errors superlinearly (a critique Timer-S1 and
Mamba4Cast both pick up on). Conversely, a model that beats the
field at `h = 720` by committing to a smooth trend can lose at
`h = 96` because it under-fits local dynamics. Reporting a single
"average over horizons" obscures this, which is why Moirai Table 6
and Sundial Table 1 report all four horizons individually.

## 4. Context length conventions

Context length `C` is how many past steps the model sees before
forecasting. Each TS-FM ships with a default and a maximum:

- **TimesFM.** Default context 512 (input patches of 32 each); max
  claimed to be configurable but most evaluations stay at 512.
- **Chronos.** 512 tokens at prediction length 64.
- **Chronos-2.** Up to 2048 tokens at horizon 64.
- **MOIRAI.** Variable-patch-size, context up to 5000+ steps;
  different tables use different `C`.
- **Timer / Timer-XL.** Timer at 672 tokens; Timer-XL scales to
  2880+ tokens.
- **Timer-S1.** Extended from 2880 to 11520 tokens via RoPE
  continuation.
- **TTM.** 512 tokens at horizon 96, as shipped.
- **MOMENT.** 512 tokens.
- **Sundial.** Variable, with a default used on LTSF.

Stretching `C` is a paper-specific decision and can swing LTSF MSE
numbers noticeably: a 2048-context Moirai and a 512-context Moirai
are effectively different models. Most published tables **do not
state `C`** explicitly. The ETTh1 MSE numbers you see in
`../benchmarks/leaderboard.md` section 5 come from at least three
different `C` configurations across Moirai, TimesFM, Sundial and
Time-MoE and are not cleanly comparable without checking each
paper's text.

## 5. The zero-shot protocol

Strict definition: a model is zero-shot on dataset `D` if no time
series (train, val or test partition) from `D` appeared in the
pretraining corpus. This is stricter than it sounds, because many
TS-FM pretraining corpora are aggregations of public archives that
*include* Monash, LOTSA, Time-300B, TimeBench and the Time Series
Pile. If your pretraining corpus includes the Monash archive and
you evaluate on Monash, you are not zero-shot on Monash even if you
have never seen the specific test window.

**Known leakage cases in the wiki.**

- TimesFM's pretraining corpus includes Google Trends and Wikipedia
  Pageviews, which overlap with Monash for several frequencies.
  Moirai-MoE Figure 3 asterisks TimesFM for this reason.
- Chronos Small / Base / Large pretraining has Monash datasets.
  Same asterisk.
- Chronos-2 Section 5.1 acknowledges partial overlap with GIFT-Eval
  training portions; test portions were explicitly scrubbed.
- Timer-S1 explicitly removes GIFT-Eval leakage from TimeBench
  during data curation, which is why its GIFT-Eval numbers are
  unusually clean.

This is why "zero-shot" in most TS-FM papers should be read as
"zero-shot as defined by the paper's own corpus policy." The
genuinely leakage-free cross-paper comparisons are:

- **Chronos Benchmark II.** Explicitly held out during Chronos
  pretraining, re-held-out during Chronos-2 pretraining.
- **Moirai-MoE Table 2 (10 held-out datasets).** Explicitly
  LOTSA-disjoint.
- **Moirai Table 5 (6 datasets).** Explicitly LOTSA-disjoint.
- **fev-bench.** Leakage-audited per model, with per-baseline
  leakage percentages reported in Chronos-2 Table 3 (up to 28% for
  Moirai-2.0).

See [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md)
section 1 for the long list.

## 6. In-domain, few-shot, and fine-tune protocols

- **In-domain.** Evaluation dataset is also in the pretraining
  corpus (the normal "train on all data, evaluate on held-out test
  window" mode). Strong per-dataset baselines like PatchTST and
  iTransformer are always in-domain. In-domain TS-FMs are still
  generalists and usually lose to the per-dataset specialist, but
  the gap is often small (< 5% on LTSF).
- **Few-shot.** `k` target-domain examples are used to adapt the
  model, typically via linear probing or LoRA. UniTS and MOMENT
  report forecasting numbers in this mode. "Few-shot" in the TS-FM
  literature usually means either k in {5, 10} windows or a fixed
  fraction (1%, 10%) of the target training set.
- **Fine-tune.** Full gradient-descent adaptation of all weights on
  the target dataset. This is the strongest configuration a TS-FM
  can be used in, and supervised per-dataset models are a fine-tune
  baseline themselves. TTM reports both zero-shot and fine-tuned
  numbers; its fine-tuned numbers are where it most clearly beats
  larger TS-FMs.

## 7. Statistical significance

### Bootstrap confidence intervals

The convention on aggregate metrics like MASE, CRPS, WQL and skill
score is a nonparametric bootstrap over datasets or over windows.
Chronos-2 reports 95% bootstrap CIs on every fev-bench, GIFT-Eval
and Chronos-II aggregate — which is what tells the reader that its
0.4-point lead over TimesFM-2.5 on GIFT-Eval WQL is *not*
statistically significant (the CIs overlap) and its 4.7-point lead
over TiRex on fev-bench SQL *is* (the CIs do not). Nearly every
other TS-FM paper in the wiki reports point estimates only; readers
should be skeptical of small "wins" without CIs.

### Diebold-Mariano test

Pairwise test for equal predictive accuracy on a single series.
Common in the classical forecasting literature, rarer in TS-FM
evaluations because modern benchmarks aggregate across many series.
Useful for per-dataset head-to-head claims.

### Friedman + Nemenyi

The Monash-archive convention: run Friedman's nonparametric test
across `k` models and `n` datasets, then the Nemenyi post-hoc to
rank them. Produces a *critical difference diagram*, which is what
you see in Moirai-MoE Figure 3 and Mamba4Cast's headline plot. A
gap smaller than the critical difference is statistically
insignificant.

## 8. Normalization: RevIN and friends

Almost every modern TS-FM applies some form of
reversible-instance-normalization at inference time: MOMENT uses
RevIN, Chronos uses mean scaling, MOIRAI applies per-context
standardization, Chronos-2 uses "robust scaling" with a
sinh-inverse transform, Sundial normalizes per patch. These
preprocessors can change MSE by > 10% relative on LTSF.

When MSE is reported on an already-z-normalized series (as ETT
LTSF tables do), the number is effectively dimensionless and
cross-dataset aggregation is meaningful. When MSE is reported on
raw series with per-paper normalization applied silently, numbers
are not comparable across papers. See
[../concepts/revin-normalization.md](../concepts/revin-normalization.md).

## 9. Reproducibility hazards

- **Different eval windows.** Two papers both reporting ETTh1 MSE
  at `h = 96` can be measuring on different 96-step windows because
  of different train/val/test cut points; the TSLib canonical split
  is the defensible default.
- **Different dataset versions.** Monash has had at least two
  revisions; LOTSA is versioned; ETT had a cleanup pass. Papers
  usually pin a specific version, but not always, and "Monash" in
  one paper can differ from "Monash" in another.
- **Different aggregation.** Arithmetic vs geometric mean across
  datasets, per-series-weighted vs per-dataset-weighted. Chronos-2
  uses geometric-mean skill score; Sundial uses GM-relative-MASE;
  Moirai uses plain MASE averaged with explicit per-dataset tables.
  See [metrics.md](metrics.md) section 1.7 and
  [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md).
- **Sample count for CRPS.** Probabilistic metrics estimated from
  fewer than ~100 samples are biased; not every paper states the
  sample count it used.

## 10. Protocols used by each benchmark in the wiki

- **Chronos** (original): in-domain (15 datasets) + Chronos
  Benchmark II (27 held-out datasets). MASE and WQL. Fixed last-h
  window per series.
- **Chronos-2**: GIFT-Eval (97 tasks) + Chronos-II (27 tasks) +
  fev-bench (98 tasks, 42 covariate). MASE/WQL/SQL skill score,
  bootstrap CIs on aggregate.
- **MOIRAI**: Monash (multivariate splits) + LOTSA hold-out + LTSF
  suite + 6-dataset CRPS table (Table 5). Mixed MASE / MSE / MAE /
  CRPS.
- **Moirai-MoE**: 29-dataset Monash aggregate + 10-dataset held-out
  CRPS / MASE. Normalized rel-Seasonal-Naive.
- **Time-MoE**: LTSF 6-dataset suite (ETTh1/2, ETTm1/2, Weather,
  ECL) + Monash held-out. MSE / MAE averaged over 4 horizons.
- **Sundial**: GIFT-Eval (97 tasks, MASE/CRPS) + Chronos Benchmark II
  + LTSF 6-dataset (MSE/MAE per horizon).
- **Timer-S1**: GIFT-Eval (MASE/CRPS) as the primary benchmark.
- **TimesFM**: Monash (18 datasets) + Darts + ETT. Scaled MAE
  geometric mean vs Seasonal Naive.
- **TTM**: LTSF 6-dataset zero-shot and few-shot. MSE / MAE.
- **Lag-Llama**: 27-dataset probabilistic suite. CRPS.
- **MOMENT, UniTS, TOTEM, Time-LLM, GPT4TS**: TSLib multi-task
  benchmark. MSE / MAE point metrics, F1 for classification /
  anomaly.
- **LLMTime**: Darts + Monash + Informer. NLL and CRPS.
- **Mamba4Cast**: 17-dataset Monash-style suite vs Chronos / DeepAR /
  ARIMA / sNaive. MASE and critical-difference diagram.
- **TimeGPT**: 300k-series internal suite. rMAE / rRMSE, conformal
  empirical coverage.

## Related wiki pages

- [metrics.md](metrics.md) — what the scored quantity means.
- [seasonality-and-baselines.md](seasonality-and-baselines.md) —
  the baseline set these protocols are evaluated against.
- [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md)
  — the "zero-shot is paper-relative" discussion and the
  normalization / aggregation conversions.
- [../concepts/zero-shot-forecasting.md](../concepts/zero-shot-forecasting.md)
  — the architectural meaning of zero-shot.
- [../datasets-benchmarks/README.md](../datasets-benchmarks/README.md)
  — the suites whose protocols are listed above.
