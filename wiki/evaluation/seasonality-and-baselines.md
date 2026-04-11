# Seasonality, Seasonal Period, and Baselines

A forecast is only interesting if it beats the thing you can compute
for free. This page explains what "the thing you can compute for
free" actually is for time series, how the seasonal period `m` enters
every scaled metric, and which baselines belong in a fair
evaluation — classical, naive, and deep. Getting the baseline wrong
is the single most common way papers quietly overstate their claims.

## 1. Why baselines matter

If you forecast hourly electricity consumption and your model beats
ARIMA by 3%, congratulations: you have beaten a 1970s algorithm.
Seasonal Naive — literally "tomorrow at 9am will look like today at
9am" — also beats ARIMA on hourly electricity, and it has zero
parameters and zero training time. Unless your pretrained foundation
model also beats Seasonal Naive it has added no value on top of
free-and-instant.

This is why MASE (see [metrics.md](metrics.md)) is built around
Seasonal Naive: by construction, `MASE = 1` is "you tied the free
baseline" and `MASE > 1` is "a statistician with no model could have
done better." Skill-score formulations used by fev-bench,
Chronos Benchmark II and GIFT-Eval work the same way: a model with
skill score 0 did nothing, and a model with negative skill score
actively lost to Seasonal Naive.

## 2. The naive family

Four standard baselines, all parameter-free.

- **Naive.** `\hat{y}_{t+h} = y_t`. Predict the last observed value
  for every horizon step. Hard to beat on random walks.
- **Seasonal Naive (sNaive).** `\hat{y}_{t+h} = y_{t + h - m}` where
  `m` is the seasonal period. Hard to beat on any series with strong
  calendar seasonality (traffic, retail, electricity).
- **Drift.** `\hat{y}_{t+h} = y_t + h \cdot \frac{y_t - y_1}{t - 1}`.
  Extrapolates the average slope through the series. Equivalent to a
  straight line through the first and last observations.
- **Mean / Average.** `\hat{y}_{t+h} = \bar{y}_{1:t}`. Predict the
  running mean. Only reasonable on mean-reverting series.

In practice, the **combined naive** baseline for a given dataset is
just `min(Naive, sNaive, Drift, Mean)` per series — Monash and
GIFT-Eval both use this implicitly because different series in the
same archive have different dynamics.

## 3. Determining the seasonal period `m`

`m` is the number of samples in one seasonal cycle. It is a property
of the *data*, not the model, and getting it right is essential
because every scaled metric (MASE, skill score, WQL in its
volume-normalized form) uses it.

Canonical choices:

- **Yearly** — `m = 1` (no intra-series seasonality; falls back to
  Naive).
- **Quarterly** — `m = 4`.
- **Monthly** — `m = 12`.
- **Weekly** — `m = 52` (sometimes 52 or 4 on sparse weekly series).
- **Daily** — `m = 7` (weekly cycle).
- **Hourly** — `m = 24` (daily) or `m = 168` (weekly, recommended by
  Monash for energy and traffic).
- **15-minutely** — `m = 96` (daily) or `m = 672` (weekly).
- **Minute-level** — dataset-specific; often `m = 1440` (daily).

The Monash archive
([../datasets-benchmarks/monash-archive.md](../datasets-benchmarks/monash-archive.md))
ships an explicit `seasonality` field per dataset, and evaluations
that report MASE on Monash must use those values or their numbers
are not comparable to the published leaderboard.

**Multi-seasonal series.** Hourly electricity has *both* a 24-hour
and a 168-hour cycle. MASE with `m = 24` and MASE with `m = 168`
produce different denominators, and the difference is not small.
Classical methods like TBATS handle multi-seasonality in the model
itself; for scaled metrics the convention is to pick the longest
period that has measurable support in the in-sample window. Chronos
and MOIRAI both follow Monash's choice where Monash covers the
dataset.

**Detecting `m` from data.** Autocorrelation-based heuristics
(`statsmodels.tsa.stattools.acf`) work for single-seasonal series.
For irregular or event-driven series the concept of a seasonal period
is ill-defined and MASE with any `m` is only loosely meaningful;
these series are generally filtered out of the large benchmarks.

## 4. Classical statistical baselines

The next rung up from the naive family is classical statistical
forecasting. These are the baselines TS-FMs inherited from the M3/M4
competition era and that still appear in every Monash-style
evaluation.

- **ETS (Exponential Smoothing / ETS state-space family).** A model
  with level, trend and seasonal state components fit by maximum
  likelihood. Implemented in `statsforecast.AutoETS` and `forecast::ets`.
  Strong on smooth, clean series; weak when the seasonal structure
  changes over time.
- **Theta.** An exponential-smoothing-plus-drift decomposition that
  won the original M3 competition. Stubborn, simple, competitive on
  Monash yearly/monthly tracks.
- **ARIMA / AutoARIMA.** Automatic order selection over
  (p, d, q)(P, D, Q, m). Reasonable on stationary or
  differenceable series; has a persistent failure mode on series
  with regime shifts.
- **TBATS.** Exponential smoothing with Box-Cox transform,
  trigonometric seasonality and ARMA errors — the standard
  multi-seasonal classical baseline.

These are "the right baseline" when the question is *whether a TS-FM
has learned anything beyond classical statistics* on data that is
well-behaved and has ample history. They are a **straw-man** when
the competition is against modern deep per-dataset supervised models
(PatchTST, N-BEATS, N-HiTS, iTransformer) — beating ETS by 5% and
claiming SOTA is the commonest way to inflate apparent performance.

See [../foundations/classical-methods.md](../foundations/classical-methods.md)
for a deeper treatment of ETS, ARIMA, Theta and TBATS.

## 5. Deep baselines routinely used

Modern TS-FM evaluations almost always include a suite of strong
per-dataset-trained deep baselines. These are what TS-FMs actually
need to beat to claim "competitive with supervised models in a
zero-shot setting."

- **DeepAR.** Autoregressive RNN with a probabilistic output head;
  the canonical neural probabilistic baseline on Monash-style
  multi-series evaluation.
- **N-BEATS / N-HiTS.** Fully-connected residual stack trained
  per-dataset; strong on M4 and Monash.
- **PatchTST.** Channel-independent patch transformer, per-dataset
  trained; the reference LTSF comparison and still the strongest
  supervised baseline on several ETT horizons.
- **iTransformer.** Inverts the attention axis so that each channel
  is a token; competitive with PatchTST on multivariate long-horizon.
- **TFT (Temporal Fusion Transformer).** Attention-based multi-horizon
  model with gating and exogenous variable handling, the default
  covariate-aware baseline.
- **TiDE.** MLP-based long-horizon model used as a MOIRAI comparison.

The standard practice when reporting a TS-FM is: run these in their
per-dataset best configuration on the same test windows as the
foundation model, and report both. If the TS-FM comes within 5% of
PatchTST zero-shot and PatchTST was trained on that exact dataset
for thousands of steps, that is a strong zero-shot claim. If a paper
only reports against classical baselines, the claim is weaker and
the reader should assume the omitted deep baselines are stronger.

## 6. Pitfalls in baseline choice

1. **Wrong `m`.** By far the most common. Running MASE with `m = 7`
   on an hourly series with true `m = 24` inflates the denominator
   (weekly-lagged MAE is larger than daily-lagged MAE because the
   daily cycle repeats) and makes your model look better than it is.
2. **Cherry-picking baselines per dataset.** Reporting against
   whichever weak baseline happens to look worst on each dataset.
   Solvable by committing to a fixed baseline set in advance.
3. **"SOTA against classical only."** If the TS-FM beats ETS and
   ARIMA but is not compared to PatchTST, the reader should assume
   PatchTST wins. This is especially common in papers that are
   trying to demonstrate universality rather than raw accuracy
   (UniTS, TOTEM, MOMENT).
4. **Reporting against Seasonal Naive with no period.** Setting
   `m = 1` on a monthly series silently turns Seasonal Naive into
   Naive and the MASE denominator becomes the first-difference MAE.
   Papers that report MASE without stating `m` are not reproducible.
5. **Baseline drift.** Benchmark code repositories sometimes ship
   stale baseline scores. Chronos-2's Table 4 showing Chronos-Bolt
   at 42.6 WQL-skill is from the fev-bench leaderboard's *October
   2025* snapshot; Sundial's Feb 2025 table ranks the same model
   differently because the leaderboard itself moved. See
   [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md)
   section 7.

## 7. Concrete examples from the wiki

- **Chronos** benchmarks against Seasonal Naive, ETS, ARIMA, Theta,
  DeepAR, TFT, PatchTST and N-BEATS on 42 datasets (Chronos
  Benchmark II) — one of the most complete baseline stacks in the
  literature, and the reason Chronos's zero-shot claims are
  trusted.
- **MOIRAI** Table 5 compares MOIRAI-Large zero-shot against
  PatchTST, TiDE, TFT and DeepAR *trained full-shot per dataset* and
  wins on 5 of 6 datasets. This is the strong form of a zero-shot
  claim.
- **Moirai-MoE** Figure 3 on Monash deliberately includes
  task-specific WaveNet, DeepAR and FFNN alongside TS-FMs, and shows
  that Moirai-MoE beats all of them while the earlier Moirai-Base
  does not.
- **TTM** argues that a 1-5M-parameter MLP-Mixer matches
  billion-parameter TS-FMs. That claim is only as valuable as the
  baseline set TTM compares to — and TTM reports against Chronos,
  Moirai, TimesFM and per-dataset PatchTST on LTSF, which is a fair
  set.
- **TimesFM** Figure 2 on Monash reports scaled-MAE geometric mean
  against N-BEATS as the deep baseline and llmtime as a sanity
  check. Narrow but honest.
- **Sundial** Table 2 on GIFT-Eval uses the full set of full-shot
  PatchTST, iTransformer and N-BEATS as deep baselines against
  zero-shot Sundial / TimesFM / Moirai / Chronos — an example of the
  "run everything in its best mode, let the reader decide"
  philosophy.

## 8. Recommended default baselines for a TS-FM paper

A defensible baseline suite for any new TS-FM evaluation contains
the following, in order:

1. **Seasonal Naive** with the dataset-declared `m`.
2. **One classical**: AutoETS or AutoARIMA via `statsforecast`.
3. **One deep probabilistic**: DeepAR.
4. **One supervised SOTA LTSF model**: PatchTST or iTransformer,
   trained per-dataset with standard hyperparameters.
5. **At least one other TS-FM**: Chronos-Base, MOIRAI-Base, or
   TimesFM run via its official release.
6. **Ideally, a lightweight non-transformer**: TTM as the
   capacity-efficiency reality check.

Anything less than (1) + (2) + (4) is not a fair evaluation. Papers
that also report on (3), (5) and (6) are the ones whose numbers
survive the comparability checklist in
[comparability-checklist.md](comparability-checklist.md).

## Related wiki pages

- [metrics.md](metrics.md) — how `m` enters MASE and the other
  scaled metrics.
- [protocols.md](protocols.md) — the train/val/test splits and
  rolling-origin conventions that these baselines are run under.
- [../foundations/classical-methods.md](../foundations/classical-methods.md)
  — a deeper pass through ETS, ARIMA, Theta and TBATS.
- [../datasets-benchmarks/monash-archive.md](../datasets-benchmarks/monash-archive.md)
  — the dataset-specific `m` values used by Monash.
- [../benchmarks/leaderboard.md](../benchmarks/leaderboard.md) — the
  aggregate numbers these baselines produce.
