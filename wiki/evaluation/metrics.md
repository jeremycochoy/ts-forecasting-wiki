# Metrics for Time-Series Forecasting

This is the deepest page in the wiki's evaluation section. Every
number reported on a TS forecasting leaderboard is an instance of one
of the metrics discussed below. If you finish this page you should be
able to open [../benchmarks/leaderboard.md](../benchmarks/leaderboard.md)
and translate each column into (a) what loss is being minimized, (b)
what unit that loss is in, (c) what it is robust to and what it is
not, and (d) which papers in the wiki actually report it.

Notation used throughout. `y_t` is the ground-truth value at time `t`,
`\hat{y}_t` a point forecast, `F_t` a predictive CDF, `q_\tau` the
`\tau`-quantile of that CDF, `h` the forecast horizon, `n` the number
of horizon steps being scored, `m` the seasonal period in samples,
and `T_in` the in-sample (training) series length.

## 1. Point metrics

Point metrics score a single-valued forecast `\hat{y}_t` against the
observation `y_t`. They are the oldest and most familiar family and
remain the default for papers that do not output a distribution
(TimesFM, Timer, Timer-XL, MOMENT, TTM, UniTS, TOTEM, Time-LLM,
GPT4TS, Mamba4Cast).

### 1.1 MAE — Mean Absolute Error

$$\mathrm{MAE} = \frac{1}{n} \sum_{t=1}^{n} |y_t - \hat{y}_t|$$

What it measures: the average distance between forecast and truth in
the native units of the series. It corresponds to minimizing the
median of the predictive distribution, so an MAE-optimal point
forecast is the conditional median.

Properties. Scale-dependent (kWh, GB/s, vehicles-per-hour — whatever
the target is). This makes cross-dataset aggregation impossible
unless you normalize first. Robust to outliers compared to MSE but
not invariant to them. Reported by essentially every paper in the
wiki as a per-dataset number before aggregation.

Pitfalls. Averaging MAE across datasets of different scales is
meaningless; a 0.3-MAE improvement on Traffic (scale ~100) is
invisible next to a 0.3-MAE improvement on Exchange (scale ~1). The
standard fix is to report MASE (see 1.7).

### 1.2 MSE — Mean Squared Error

$$\mathrm{MSE} = \frac{1}{n} \sum_{t=1}^{n} (y_t - \hat{y}_t)^2$$

What it measures: average squared error. MSE-optimal point forecasts
are the conditional *mean*, which matters when the predictive
distribution is asymmetric.

Properties. Scale-dependent (squared units, so cross-dataset
comparison is even worse than MAE). Quadratic penalty means a single
large error dominates the score, which is useful when large errors
are disproportionately costly and dangerous when your evaluation
window contains a regime change or an outlier.

Reported by: the LTSF cluster (TimesFM, Moirai, Moirai-MoE, Time-MoE,
Timer-XL, Sundial, TTM, MOMENT, Time-LLM, GPT4TS) on ETT / Weather /
Electricity / Traffic / ILI at horizons `{96, 192, 336, 720}`. It is
the single most reported metric on the LTSF suite despite being, in
absolute terms, one of the least cross-comparable.

### 1.3 RMSE — Root Mean Squared Error

$$\mathrm{RMSE} = \sqrt{\mathrm{MSE}}$$

RMSE is MSE pulled back to the original units so that it is
commensurate with MAE. It keeps MSE's outlier sensitivity. Mostly
reported by classical-method baselines in Monash-style comparisons
and by TimeGPT (which quotes rRMSE, the relative version).

### 1.4 MAPE — Mean Absolute Percentage Error

$$\mathrm{MAPE} = \frac{1}{n} \sum_{t=1}^{n} \frac{|y_t - \hat{y}_t|}{|y_t|}$$

What it measures: average relative error as a percentage. Attractive
because it is scale-free and easy to explain to stakeholders ("we
were off by 7% on average").

Failure modes. Three serious ones, each of which appears in TS-FM
evaluations regularly.

1. **Division by zero.** If `y_t = 0` anywhere in the evaluation
   window, MAPE is literally undefined. On intermittent-demand series
   (retail, healthcare events) this is fatal.
2. **Asymmetry.** A forecast that is 50% too high on a true value of
   10 gives MAPE 50%; a forecast that is 50% too low gives MAPE 50%;
   but a forecast of 0 for a true value of 10 gives MAPE 100%,
   whereas a forecast of 20 for a true value of 10 gives MAPE 100% as
   well — so both sides look identical even though the multiplicative
   error is 1/0 vs. 2. MAPE systematically rewards under-forecasting
   on positive series.
3. **Blowup at small values.** Because the denominator is `|y_t|`,
   MAPE can assign enormous weight to a single near-zero observation,
   making a per-window MAPE unbounded in practice.

MAPE is still reported occasionally on well-behaved retail or
macroeconomic series (TimeGPT reports rMAPE implicitly via its
relative-error tables), but modern TS-FM papers have largely replaced
it with MASE.

### 1.5 sMAPE — symmetric MAPE

$$\mathrm{sMAPE} = \frac{1}{n} \sum_{t=1}^{n} \frac{|y_t - \hat{y}_t|}{(|y_t| + |\hat{y}_t|)/2}$$

sMAPE was proposed to fix MAPE's asymmetry; the name is aspirational.
It bounds the per-point error in `[0, 2]` (or `[0, 200]%`), which
seems like an improvement, but introduces its own problems. When both
`y_t` and `\hat{y}_t` are near zero the denominator goes to zero;
when the forecast has the wrong sign the per-point contribution
saturates at 2 regardless of how wrong it is; and the sMAPE of a
symmetric pair `(y, \hat{y})` and `(\hat{y}, y)` is the same, which
means sMAPE cannot tell you whether you are over- or under-forecasting
systematically.

Hyndman & Koehler (2006) called sMAPE behaviour at small values an
"insanity", specifically because tiny absolute deviations produce
huge sMAPE contributions that dwarf the much more meaningful errors
on large observations. sMAPE is still the headline metric of the M3
and M4 competitions, which is the main reason LLMTime and a few
earlier baselines report it.

### 1.6 WAPE — Weighted Absolute Percentage Error

$$\mathrm{WAPE} = \frac{\sum_{t=1}^{n} |y_t - \hat{y}_t|}{\sum_{t=1}^{n} |y_t|}$$

WAPE replaces "mean of ratios" with "ratio of sums", which removes
the division-by-zero problem pointwise and reweights the error by
volume. It is the standard retail-forecasting metric and is sometimes
reported in TS-FM retail-subset evaluations, but is less common in
the wiki's headline tables.

### 1.7 MASE — Mean Absolute Scaled Error

MASE is the most important metric on this page. It is the standard
aggregator on Monash, on Chronos Benchmark II, on GIFT-Eval (as a
skill score), and on fev-bench (as a skill score). Every TS-FM in the
wiki that reports aggregated accuracy reports MASE or a skill-score
derivative of it. Origin: Hyndman & Koehler 2006, *Another look at
measures of forecast accuracy*.

$$\mathrm{MASE} = \frac{\frac{1}{n}\sum_{t=1}^{n} |y_t - \hat{y}_t|}{\frac{1}{T_{in}-m} \sum_{i=m+1}^{T_{in}} |y_i - y_{i-m}|}$$

The numerator is the out-of-sample MAE of the forecast. The
denominator is the **in-sample** MAE of the *seasonal-naive*
forecaster — i.e., the MAE you would get by predicting each point
equal to the observation `m` steps earlier — computed on the
training portion of the series. The seasonal period `m` matches the
natural frequency of the data: daily → 7, hourly → 24, monthly → 12,
quarterly → 4, yearly → 1. See
[seasonality-and-baselines.md](seasonality-and-baselines.md) for how
Monash and GIFT-Eval pick `m`.

Interpretation at familiar values. `MASE = 1` means your forecast is
exactly as good as the in-sample seasonal-naive baseline. `MASE < 1`
means you beat seasonal naive on that series; `MASE > 1` means you
lost to a forecaster that did nothing but repeat the last seasonal
cycle. The bar is much harder than it sounds: on strongly seasonal
series (hourly electricity, quarterly sales) seasonal naive is
already a strong baseline, so MASE 0.7 on a Monash dataset is a
respectable TS-FM number and MASE 0.5 is excellent. See the Monash
geometric-mean relative MASE numbers for Moirai-MoE (0.651),
Chronos-Base (0.656) and Moirai-Large (0.729) in
[../benchmarks/leaderboard.md](../benchmarks/leaderboard.md) — these
are all below 1 but by remarkably thin margins.

Why MASE is the right choice for aggregation.

- **Scale-free.** The denominator has the same units as the
  numerator, so MASE is dimensionless and can be averaged across
  datasets with wildly different scales without losing meaning.
- **Grounded in a meaningful baseline.** A MASE of 0.9 on a
  hard-to-predict weather series and a MASE of 0.9 on a trivially
  seasonal retail series both translate to "10% better than
  seasonal-naive", which is a coherent cross-dataset statement.
- **Robust across horizons.** MASE on `h=24` and MASE on `h=96` are
  directly comparable for the same series; MSE at those horizons is
  not.

Pitfalls, in order of severity.

1. **The wrong `m` quietly destroys the metric.** If Monash lists
   `m = 24` for hourly electricity and your evaluation harness uses
   `m = 7` (the daily default), the denominator is computed against
   a different baseline and every MASE you report is on a different
   scale than the published numbers. Chronos Benchmark II, Monash
   and GIFT-Eval ship explicit `m` per dataset; use theirs.
2. **Near-constant training series.** If the in-sample seasonal-naive
   MAE is zero or near zero (near-constant water-level series,
   saturated sensors), the denominator blows up and MASE goes to
   infinity. Monash filters these series; custom evaluations need to
   do the same.
3. **MASE is not probabilistic.** Reporting only MASE for a model
   that claims a distributional output gives it partial credit for a
   forecast it never committed to. Pair with CRPS or WQL.
4. **Skill-score transformations.** fev-bench and GIFT-Eval report a
   MASE-derived **skill score** `S = 1 - G_MASE` as a percentage
   where `G_MASE` is the geometric mean of per-dataset MASE divided
   by seasonal-naive MASE. Skill score 30% on GIFT-Eval MASE is
   equivalent to `G_MASE = 0.70`, which corresponds to raw relative
   MASE 0.70. Chronos-2 documents this identity explicitly (see
   [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md)
   section 2). Don't confuse a skill-score delta of 5 points with a
   raw-MASE delta of 0.05.

Which papers in the wiki report MASE. Chronos ([papers/chronos.md](../papers/chronos.md))
on Benchmark II in the original paper; Chronos-2 ([papers/chronos-2.md](../papers/chronos-2.md))
on GIFT-Eval, Chronos-II and fev-bench; MOIRAI ([papers/moirai.md](../papers/moirai.md))
and Moirai-MoE ([papers/moirai-moe.md](../papers/moirai-moe.md)) on Monash and a 10-dataset
held-out; TimesFM ([papers/timesfm.md](../papers/timesfm.md)) on Monash/Darts/ETT; Lag-Llama
([papers/lag-llama.md](../papers/lag-llama.md)); Sundial ([papers/sundial.md](../papers/sundial.md))
on GIFT-Eval; Timer-S1 ([papers/timer-s1.md](../papers/timer-s1.md)) on GIFT-Eval; Mamba4Cast
([papers/mamba4cast.md](../papers/mamba4cast.md)) on its 17-dataset
suite. TimeGPT ([papers/timegpt.md](../papers/timegpt.md)) reports
`rMAE`, a sibling quantity using a different baseline.

### 1.8 NRMSE / normalized MSE

$$\mathrm{NRMSE} = \frac{\mathrm{RMSE}}{\sigma(y)}$$

Normalized RMSE divides RMSE by either the standard deviation of the
target series (the "normalized by variance" form used by Informer,
PatchTST and MOMENT) or by the interquartile range. This converts MSE
into a dimensionless number comparable across scales and is the form
the LTSF long-horizon literature uses implicitly: the ETT MSE numbers
reported across papers are computed on z-normalized series, which is
why an MSE of 0.4 on ETTh1 is not "0.4 of the original unit" but
"0.4 after subtracting the per-series mean and dividing by the
per-series standard deviation." This is one of the biggest hidden
assumptions in LTSF benchmarking and is revisited in
[protocols.md](protocols.md) and
[../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md).

## 2. Probabilistic metrics

A probabilistic forecast commits to an entire predictive distribution
`F_t`, not just a point. The metrics below are *strictly proper
scoring rules*: the expected score is minimized uniquely by the true
conditional distribution, so an optimal forecaster cannot cheat by
hedging.

### 2.1 CRPS — Continuous Ranked Probability Score

$$\mathrm{CRPS}(F, y) = \int_{-\infty}^{\infty} \left(F(z) - \mathbb{1}\{z \ge y\}\right)^2 \, dz$$

CRPS integrates the squared distance between the predictive CDF `F`
and the ideal step function at the observation `y`. It is a strictly
proper rule, expressed in the *units of `y`*, which is a rare
combination and the reason CRPS is the probabilistic metric of choice
across modern TS-FMs.

Two key properties.

- **Reduces to MAE for a point forecast.** If `F` puts all its mass
  on a single value `\hat{y}`, then `CRPS(F, y) = |y - \hat{y}|`.
  This is extremely useful and commonly misunderstood — it means you
  can compare a Chronos-style probabilistic forecast and a
  Timer-style point forecast on the same CRPS scale without unfairly
  punishing the deterministic one. It also means that CRPS is
  literally never worse than MAE: a model that returns its
  conditional median as a degenerate distribution gets MAE; a model
  that spreads mass calibrated around it gets strictly lower CRPS.
- **Sample estimator.** When `F` is only accessible via `N` samples
  `\hat{y}^{(1)}, \ldots, \hat{y}^{(N)}` (as in Chronos, which
  samples from a categorical head, or Sundial, which integrates a
  flow-matching field), the standard plug-in is
  `\mathrm{CRPS}(F, y) \approx \frac{1}{N} \sum_i |\hat{y}^{(i)} - y| - \frac{1}{2 N^2} \sum_{i, j} |\hat{y}^{(i)} - \hat{y}^{(j)}|`.
  The estimator is biased downward for finite `N`; probabilistic
  TS-FM papers typically use `N \ge 100` samples.

CRPS can be decomposed into reliability + resolution − uncertainty,
giving a principled way to separate "is your predicted CDF
calibrated" from "is it sharp." See
[probabilistic-evaluation.md](probabilistic-evaluation.md) for the
decomposition.

Reported by: MOIRAI (Table 5, six datasets), Moirai-MoE (Table 2, 10
datasets), Chronos (original paper), Chronos-2 (GIFT-Eval and
Chronos-II as skill score), Sundial (Table 2 on GIFT-Eval),
Lag-Llama, LLMTime (via its digit-distribution density), Timer-S1
(GIFT-Eval). See also the paper-level summary in
[what-was-evaluated.md](what-was-evaluated.md).

### 2.2 Pinball / quantile loss

$$\rho_\tau(y, \hat{q}_\tau) = (y - \hat{q}_\tau)(\tau - \mathbb{1}\{y < \hat{q}_\tau\})$$

The pinball (or quantile) loss at level `\tau \in (0, 1)` is the
strictly proper rule for the `\tau`-quantile. It is asymmetric: at
`\tau = 0.5` it reduces to `0.5 |y - \hat{q}_{0.5}|`, i.e. the
median-regression loss. At `\tau = 0.9` it penalizes under-forecasts
nine times more than over-forecasts.

Pinball loss is typically aggregated over a multi-quantile grid
(standard choice: `\tau \in \{0.1, 0.2, \ldots, 0.9\}`) to give a
single probabilistic score. It is the training loss of quantile
regression heads, and is minimized jointly in Timer-S1 (9 quantiles,
0.1-0.9) and Chronos-2 (nine-quantile decoder). MOIRAI uses the
Student-t mixture's negative log-likelihood during training and
pinball/CRPS for evaluation.

### 2.3 WQL — Weighted Quantile Loss

WQL (Chronos, GIFT-Eval, Chronos-2) aggregates pinball loss across
quantiles and reweights by the target scale so that high-volume
series do not dominate:

$$\mathrm{WQL} = \frac{2}{\sum_t |y_t|} \sum_{\tau \in \mathcal{T}} \sum_{t=1}^{n} \rho_\tau(y_t, \hat{q}_{\tau,t})$$

where `\mathcal{T}` is the quantile grid. Lower is better. WQL is the
probabilistic metric of Chronos Benchmark II and the probabilistic
column of GIFT-Eval. fev-bench's **SQL** (Scaled Quantile Loss) is
the same object but scaled by seasonal-naive WQL rather than by
target magnitude, which makes it directly comparable across datasets.
The SQL vs. WQL distinction is the source of the "51.4 vs. 46.6" kind
of number you see in Chronos-2's GIFT-Eval / fev-bench tables — same
underlying loss, different normalization. See
[../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md)
section 5 for the conversion identities.

### 2.4 NLL / log-score

$$\mathrm{NLL}(F, y) = -\log f(y)$$

Where `f` is the predictive density. The log-score is strictly proper
and was the original scoring rule for probabilistic forecasts, but it
is infinitely sensitive to tail observations — a single `y` outside
`F`'s support contributes `+\infty`. For TS-FMs with discrete output
heads (Chronos), NLL is the cross-entropy training loss itself.
LLMTime reports NLL because its digit-string distribution is a proper
density and NLL is the natural per-token score. Lag-Llama trains a
Student-t head whose NLL is the training objective, but reports CRPS
for evaluation for robustness reasons.

### 2.5 Brier score

Brier score is the squared error for probabilistic forecasts of a
binary event. It is the 0/1 special case of CRPS and is rarely used
in the TS-FM literature because continuous forecasting is the
overwhelming majority of the task. It appears in rare probability-of-
exceedance use cases (e.g. flood warning). No paper in this wiki
reports Brier.

## 3. Multivariate probabilistic metrics

Univariate CRPS / WQL averaged independently across channels does not
score cross-channel dependence. A forecaster that predicts the
correct marginal for every channel but assigns zero correlation where
the truth has `\rho = 0.9` will look perfect under marginal CRPS and
will be unusable in downstream decisions (portfolio risk, joint
covariate-informed retail stockout planning). Two scoring rules fix
this.

### 3.1 Energy score

$$\mathrm{ES}(F, y) = \mathbb{E}_{X \sim F} \|X - y\| - \tfrac{1}{2} \mathbb{E}_{X, X' \sim F} \|X - X'\|$$

The multivariate generalization of CRPS: replace absolute value with
Euclidean norm. Strictly proper in the multivariate sense, sensitive
to both marginal calibration and joint structure, but somewhat
insensitive to *specific* correlations — a forecast can have the
right joint spread and the wrong correlation structure and still look
good. Estimated from samples.

### 3.2 Variogram score

$$\mathrm{VS}_p(F, y) = \sum_{i, j} \left(|y_i - y_j|^p - \mathbb{E}_{X \sim F} |X_i - X_j|^p\right)^2$$

Scores the agreement between predicted and observed *pairwise
differences*, making it explicitly sensitive to correlations. Chosen
`p` is typically 0.5. More diagnostic than the energy score when the
question is "does the model get cross-channel structure right."

Relevance in the wiki. Chronos-2's covariate-informed and
multivariate fev-bench results (47.0 SQL skill score on the
covariates subset) are the clearest case where multivariate structure
is directly evaluated; MOIRAI and Timer-XL argue for long-context
multivariate attention but evaluate mostly on marginal CRPS plus
per-channel MSE. The field as a whole still under-reports multivariate
probabilistic scores — this is the biggest open evaluation gap in
TS-FMs and is called out in
[../research/open-problems.md](../research/open-problems.md).

## 4. Calibration diagnostics

A scoring rule tells you how wrong the forecast is on average; a
calibration diagnostic tells you *why*. Both matter: a CRPS-optimal
forecaster can still be badly miscalibrated on specific quantiles if
the aggregate score was driven by a few easy series.

### 4.1 Reliability diagram

Plot predicted probability of exceeding `q_\tau` against empirical
frequency of exceeding `q_\tau`, across a validation sample. A
well-calibrated forecast sits on the diagonal. Deviations above the
diagonal mean over-confident high-tail predictions; deviations below
mean under-confident.

### 4.2 PIT histogram

Probability Integral Transform: compute `F_t(y_t)` for every
(forecast, observation) pair. If `F_t` is perfectly calibrated, the
PIT values are uniform on `[0, 1]`. A U-shaped PIT histogram means
the forecast is over-confident (too narrow); a hump-shaped one means
under-confident. PIT is the standard visual diagnostic for
distributional forecasters and is cheap to compute from samples.

### 4.3 Empirical coverage

For a stated prediction interval (e.g., the 10-90 interval), measure
the empirical fraction of observations that land inside. A
95%-nominal interval that covers 86% of observations is badly
miscalibrated. TimeGPT's conformal-prediction intervals report
empirical coverage as the main probabilistic claim.

**Why calibration matters alongside CRPS.** A proper score like CRPS
can be gamed by returning a narrow, over-confident distribution when
the data are easy and a wide, under-confident distribution when
they're hard. Both give good CRPS on average and terrible calibration
on tails. The standard discipline is to report CRPS **plus** a
reliability diagram or PIT histogram. Almost no paper in the wiki
does this routinely — the gap is discussed in
[what-was-evaluated.md](what-was-evaluated.md).

## 5. Which metric for which use case

| If you care about... | Report... | Why |
|---|---|---|
| A single headline point-forecast number across many datasets | MASE or skill-score-MASE | Scale-free, meaningful baseline, aggregates well |
| Long-horizon accuracy on a single dataset with known scale | MSE or NRMSE | Matches the LTSF literature; unit-full |
| Cost-weighted retail / inventory forecasts | WAPE or volume-weighted pinball | Reflects the business loss function |
| Probabilistic univariate, cross-dataset | CRPS or WQL | Strictly proper, units of `y`, sample estimator |
| Probabilistic multivariate / joint structure | Energy score + variogram score | Univariate CRPS misses cross-series dependence |
| Quantile-specific decisions (tail risk, stockouts) | Pinball at that `\tau` | Directly minimized by the quantile you need |
| Calibration / trustworthiness of intervals | Reliability diagram + PIT histogram + empirical coverage | CRPS alone can hide miscalibration |
| Beating a strong operator baseline | Skill score vs. seasonal naive | Negative values = you lost to free forecast |

## 6. Summary: reading a TS-FM table

When you see a cell on the leaderboard, ask:

1. **What metric?** MASE, CRPS, WQL, SQL, MSE, skill score?
2. **What normalization?** Raw, divided-by-seasonal-naive,
   z-normalized, robust-scaled, RevIN on or off? (See
   [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md).)
3. **What seasonal period `m`?** If MASE is involved, wrong `m` ≠
   your MASE.
4. **What aggregation?** Arithmetic mean across datasets, geometric
   mean, mean rank, skill score?
5. **Point or probabilistic?** A point MAE and a probabilistic CRPS
   with the same number on the same series are not the same claim.

Only once all five are the same across two models is it meaningful to
say "model A beats model B by 0.02".

## Related wiki pages

- [seasonality-and-baselines.md](seasonality-and-baselines.md) — how
  `m` and the seasonal-naive denominator are chosen.
- [probabilistic-evaluation.md](probabilistic-evaluation.md) — the
  deeper CRPS / calibration discussion.
- [protocols.md](protocols.md) — normalization, splits and horizons.
- [../benchmarks/leaderboard.md](../benchmarks/leaderboard.md) — the
  tables these metrics populate.
- [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md)
  — skill-score / GM-relative-MASE / WQL / SQL conversions.
- [../research/glossary.md](../research/glossary.md) — short
  definitions linking back here.
