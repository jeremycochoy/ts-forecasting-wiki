# Probabilistic Evaluation

Point forecasts answer "what is the most likely next value?"
Probabilistic forecasts answer "what is the distribution of the next
value?" The second question is what almost every real forecasting
user actually wants — inventory planning, staffing, risk, energy
dispatch, medical alerting — because the downstream decision depends
on tail events rather than conditional means. This page goes deeper
than [metrics.md](metrics.md) on how TS foundation models are
actually scored probabilistically, where the common failure modes
are, and what a complete probabilistic evaluation looks like.

## 1. Why probabilistic evaluation exists

A point forecast `\hat{y}_t` tells you what the model "thinks" and
says nothing about how confident it is. Two models that output
identical point forecasts can be wildly different: one might assign
a 90% credible interval of `\pm 2`, the other of `\pm 20`. In a
warehouse stocking decision, that difference determines whether the
user orders a safety buffer of 3 units or 30. MAE and MASE cannot
distinguish the two. CRPS can.

Decisions under uncertainty also require per-quantile accuracy, not
per-mean accuracy. A food bank that needs to stock to the 95th
percentile of demand cannot use a median-optimal forecast; a power
grid that dispatches to the 5th percentile of load cannot either.
Reporting only MASE gives these users no actionable information.

## 2. Point vs distributional forecasts: what MAE / MASE miss

- A forecaster can be median-optimal (low MAE) and still fail
  catastrophically on tails.
- A forecaster can be point-biased (high MAE) but
  distribution-correct, which is still useful for risk-weighted
  decisions.
- Strictly proper scoring rules are the only honest way to score
  distributional forecasts; MAE and MSE are *not* proper rules over
  distributions.

"Proper" is technical and important. A scoring rule `S(F, y)` is
strictly proper if `E_{y \sim G}[S(F, y)] \ge E_{y \sim G}[S(G, y)]`
with equality iff `F = G`. Expected score is uniquely minimized by
the true distribution. MAE-and-report-point fails this: you can get
perfect MAE by always predicting the conditional median, even when
you know nothing about the rest of the distribution. CRPS, pinball
loss, NLL and the Brier score are all strictly proper.

## 3. CRPS: the core probabilistic rule

$$\mathrm{CRPS}(F, y) = \int_{-\infty}^{\infty} \left(F(z) - \mathbb{1}\{z \ge y\}\right)^2 \, dz$$

CRPS integrates the squared distance between the predicted CDF and
the ideal step function at the observation. Three properties make it
the workhorse of TS-FM probabilistic evaluation.

**Unit-full.** CRPS is in the units of the target. A CRPS of 0.05 on
a normalized series means "half a unit of squared CDF distance",
which is intuitive in a way that a log-score is not. This is why
MOIRAI Table 5 (six datasets, electricity / solar / walmart /
weather / istanbul traffic / turkey power) can report CRPS in the
original domain's units and the reader can compare to MAE mentally.

**Reduces to MAE for a point forecast.** If `F` is a Dirac mass at
`\hat{y}`, `CRPS(F, y) = |y - \hat{y}|`. This is the property that
makes CRPS fair to point-only models: Timer or Timer-XL or MOMENT
can be evaluated on CRPS (trivially: wrap their point forecast in a
degenerate CDF) and they get their MAE back. A Chronos or MOIRAI
sample-based distribution has the *opportunity* to do strictly
better by spreading mass calibrated around the median. If the
distribution is miscalibrated, CRPS punishes it toward the
point-forecast floor.

**Decomposition.** CRPS decomposes as `reliability + resolution - uncertainty`:

- *Reliability* is how calibrated the predicted CDF is (larger when
  predicted quantiles systematically disagree with empirical
  frequencies; zero when calibration is perfect).
- *Resolution* is the informativeness of the forecast (larger when
  the forecast varies meaningfully with inputs rather than defaulting
  to the marginal).
- *Uncertainty* is the irreducible entropy of the observation
  process itself.

The decomposition tells you *why* a CRPS number is what it is: a
poorly-calibrated but sharp forecast and a well-calibrated but vague
one can have the same CRPS for very different reasons.

### CRPS sample estimator

When you only have samples `\hat{y}^{(1)}, \ldots, \hat{y}^{(N)}`
from the predictive distribution — the case for Chronos (categorical
sampling), Sundial (flow-matching integration) and Lag-Llama
(Student-t sampling) — the standard unbiased estimator is:

$$\widehat{\mathrm{CRPS}} = \frac{1}{N}\sum_{i=1}^{N} |\hat{y}^{(i)} - y| - \frac{1}{2 N^2} \sum_{i=1}^{N}\sum_{j=1}^{N} |\hat{y}^{(i)} - \hat{y}^{(j)}|$$

Finite-sample bias. For small `N` the estimator is biased downward
(the `\frac{1}{2 N^2}` term is underestimated). The fix is to use
`\frac{1}{N(N-1)}` scaling on the cross term, which removes the bias;
Chronos-2 and Sundial both use the unbiased form. The bias is
typically negligible for `N \ge 100` samples, which is the TS-FM
convention.

## 4. Quantile loss, WQL, and SQL

### Pinball loss, reviewed

$$\rho_\tau(y, \hat{q}_\tau) = (y - \hat{q}_\tau)(\tau - \mathbb{1}\{y < \hat{q}_\tau\})$$

Strictly proper for estimation of the `\tau`-quantile of `y`.
Summed over a finite quantile grid `\mathcal{T} \subset (0, 1)`, it
gives a weighted approximation to CRPS that becomes tighter as
`|\mathcal{T}|` grows. At the standard 9-quantile grid
`{0.1, 0.2, ..., 0.9}` the pinball sum is already within a few
percent of the true CRPS on smooth distributions, which is why
GIFT-Eval adopts the quantile-loss form.

### WQL as the GIFT-Eval standard

$$\mathrm{WQL} = \frac{2}{\sum_t |y_t|} \sum_{\tau \in \mathcal{T}} \sum_{t=1}^{n} \rho_\tau(y_t, \hat{q}_{\tau,t})$$

The `|y_t|` in the denominator is the "weighted" part: it scales
total loss by the magnitude of the target. This has two
consequences. (a) High-volume series dominate in raw scale — a retail
warehouse with 10^6 units/day counts more than one with 10^3
units/day — which is often what the user wants but sometimes hides
low-volume failures. (b) Cross-dataset aggregation of WQL is
meaningful as long as every dataset shares the same normalization.

### SQL: fev-bench's variant

SQL divides per-series WQL by the WQL of the Seasonal Naive forecast
on the same series, producing a **relative** quantile loss that is
per-dataset dimensionless. fev-bench and Chronos-2 both use SQL, and
the "skill score" reported in Chronos-2's Table 3 (fev-bench) is
`1 - geometric_mean(SQL)`.

### Quantile-aligned training

Aligning training loss with evaluation metric is the Timer-S1 move:
it trains a 9-quantile head directly with pinball loss on the same
grid GIFT-Eval scores, eliminating the "optimize MSE then report
CRPS" mismatch. Chronos-2 does the same with a quantile decoder, and
both papers argue this is the reason they lead on GIFT-Eval among
pretrained-only models.

## 5. Calibration vs. sharpness trade-off

Every probabilistic forecast lives on a two-dimensional plane
(calibration × sharpness). Calibration is "when you say 90%, is it
really 90%?" Sharpness is "how narrow are your intervals?" The
goal is maximum sharpness *subject to* calibration — not sharpness
alone, because an arbitrarily narrow forecast is always badly
calibrated, and not calibration alone, because a maximally wide
forecast is always perfectly calibrated (and useless).

CRPS rewards sharpness given calibration. A poorly-calibrated but
sharp forecast is punished because its mass is in the wrong place;
a well-calibrated but vague one is punished because its mass is
spread too thin.

### Reliability diagrams

Plot predicted quantile level `\tau` on the x-axis against the
empirical fraction of observations falling below the predicted
`q_\tau` on the y-axis, for `\tau \in \{0.1, 0.2, ..., 0.9\}`. A
perfectly calibrated forecast sits on the diagonal. Most real
TS-FM evaluations produce something sagging below the diagonal on
high quantiles (over-confident upper tails).

### PIT histograms

Compute `F_t(y_t)` for every forecast-observation pair and histogram
the values. Under perfect calibration the histogram is uniform on
`[0, 1]`. Under-dispersion (over-confidence) produces a U-shape;
over-dispersion (under-confidence) produces a hump. PIT is the
single most informative plot for a distributional forecaster and is
essentially free to compute from samples.

### Empirical coverage

Easier for practitioners. Construct the 80% prediction interval,
check the fraction of observations inside it. An 80%-nominal
interval covering 68% is under-covering by 12 points. TimeGPT
reports empirical coverage as its main distributional claim because
its intervals are built by conformal prediction rather than a
learned distribution.

## 6. Evaluating sample-based generative forecasters

Chronos, Sundial, and parts of Chronos-2 do not have a closed-form
predictive CDF. The model outputs samples drawn from a discrete
codebook (Chronos) or from an integrated flow-matching ODE (Sundial).
CRPS is estimated from samples, quantiles are estimated from
samples, and both come with finite-sample bias.

Two practical consequences:

- **`N` matters.** Sample CRPS is biased low for small `N`; the bias
  decreases with increasing samples. `N = 20` is *not* enough. Most
  papers report `N \ge 100`.
- **Quantile sharpness.** Extreme quantiles (`\tau \in \{0.01, 0.99\}`)
  estimated from 100 samples have large variance — the 99th-
  percentile estimate comes from the 99th-largest of 100 samples,
  and that single value drives the pinball loss.

A generative forecaster that samples in fast batch (Sundial, via
flow-matching ODE) can afford `N = 1000` and make these issues
disappear. A slow sample-per-call architecture (LLMTime, querying
GPT-3 for every digit) cannot.

## 7. Multivariate probabilistic evaluation

Univariate CRPS averaged independently across channels is not a
multivariate metric — it scores marginals only and is blind to
cross-channel dependence. A forecaster that predicts every channel
marginal perfectly but decorrelates them from the truth looks
optimal under marginal CRPS and is unusable for portfolio risk,
covariate-informed retail planning, or joint weather forecasting.

**Energy score** is the multivariate CRPS generalization — replace
`|X - y|` with `\|X - y\|_2`. Strictly proper in the multivariate
sense but relatively insensitive to specific correlation structure.

**Variogram score** directly scores pairwise differences between
channels and is sensitive to correlation errors.

The wiki's most relevant case is Chronos-2, which explicitly evaluates
on fev-bench's multivariate and covariate-informed subsets and
reports the covariate gap at 47.0 SQL skill score vs. 40.0 for the
next-best covariate-aware model. However, even Chronos-2 reports its
multivariate performance through skill-score-SQL rather than an
explicit energy or variogram score, so "does Chronos-2 actually get
cross-channel joint structure right?" remains an open question in the
TS-FM literature.

## 8. Scoring rule gaming and the calibration/CRPS tension

A worryingly common failure: a training loss that minimizes
"expected CRPS" can be minimized by returning an over-smooth,
under-confident distribution when the data are easy, and a
wide-and-vague distribution when the data are hard. Both give
reasonable CRPS on average, and the PIT histogram is uniform in
aggregate, but the per-quantile reliability diagram shows clear
slumping at the upper tails. The fix is to report CRPS **alongside**
calibration — either a reliability diagram or a PIT histogram — so
that the reader can see which side the average is averaging over.

Essentially no paper in the wiki reports both. CRPS is reported by
all probabilistic TS-FMs; calibration diagnostics are reported by
approximately none. This is the single biggest gap in TS-FM
probabilistic evaluation and is noted in
[what-was-evaluated.md](what-was-evaluated.md).

## 9. Paper-level map

- [../papers/chronos.md](../papers/chronos.md) — samples from the
  categorical head, CRPS and WQL reported on Benchmark II.
- [../papers/chronos-2.md](../papers/chronos-2.md) — quantile
  decoder, SQL on fev-bench, WQL skill score on GIFT-Eval and
  Chronos-II; reports 95% bootstrap CIs on aggregate skill scores.
- [../papers/moirai.md](../papers/moirai.md) — Student-t mixture
  head, Table 5 CRPS on six held-out datasets.
- [../papers/moirai-moe.md](../papers/moirai-moe.md) — Table 2 CRPS
  / MASE on 10 held-out datasets, both paper-clean zero-shot.
- [../papers/sundial.md](../papers/sundial.md) — flow-matching
  generative head, Table 2 MASE / CRPS on GIFT-Eval.
- [../papers/lag-llama.md](../papers/lag-llama.md) — Student-t
  output, CRPS-as-eval, trained with NLL.
- [../papers/llmtime.md](../papers/llmtime.md) — digit-distribution
  to continuous density, evaluated with NLL and CRPS on Darts /
  Monash / Informer benchmark sets.
- [../papers/timegpt.md](../papers/timegpt.md) — conformal
  intervals, empirical coverage reported instead of CRPS / WQL.
- [../papers/timer-s1.md](../papers/timer-s1.md) — 9-quantile
  decoder, CRPS on GIFT-Eval as its headline metric.

## Related wiki pages

- [metrics.md](metrics.md) — the broader metric reference.
- [seasonality-and-baselines.md](seasonality-and-baselines.md) — why
  Seasonal Naive is the WQL normalizer.
- [protocols.md](protocols.md) — the splits and windows under which
  these scores are computed.
- [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md)
  — WQL vs SQL vs GM-relative-CRPS conversions.
- [../concepts/probabilistic-forecasting.md](../concepts/probabilistic-forecasting.md)
  — the architectural side of probabilistic TS-FMs.
