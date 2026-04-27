# Probabilistic Forecasting

## Intuition

Probabilistic forecasting produces a full predictive distribution `p(y_{t+1:t+L} | y_{1:t})` rather than a single point estimate `\hat y`. The motivation is entirely practical: almost every downstream decision — inventory, staffing, pricing, capacity planning, risk — requires knowing not just the expected value but the *plausible range*. A point forecast that is right on average is useless if it never tells you "this week might spike". Probabilistic output also tends to *improve training*: matching a distribution is a richer signal than matching a scalar, so NLL objectives regularize better than MSE against heavy-tailed targets.

## Mechanics

There are five recognizable approaches in TS foundation models, each replacing a different piece of the standard regression pipeline.

**1. Parametric head + NLL.** Attach a small MLP that outputs the parameters of a chosen distribution, train by maximizing the data log-likelihood under it:

```
(mu, sigma, nu) = head(z)            # z = transformer output, per horizon step
loss            = -log StudentT(y; mu, sigma, nu)
```

`[Lag-Llama](../papers/lag-llama.md)` uses a Student-t head — heavy-tailed, robust to outliers. `[MOIRAI](../papers/moirai.md)` uses a mixture of K Student-t components, `p(y) = \sum_k pi_k t_{nu_k}(y; mu_k, sigma_k)`, which can capture multimodal next-step distributions.

**2. Categorical sampling over a quantized vocabulary.** This is `[Chronos](../papers/chronos.md)`: bin real values into a fixed vocabulary of size `V`, train with categorical CE, then draw `M` sample trajectories at inference:

```
for m in 1..M:
    y_tokens[m] = sample_decoder(context_tokens, horizon=L)
    y_real[m]   = dequantize(y_tokens[m]) * scale
quantiles = empirical_quantiles(y_real, q=[0.1, 0.5, 0.9, ...])
```

No distributional assumption — arbitrary shapes are captured by sample density.

**3. Direct quantile regression.** `[Chronos-2](../papers/chronos-2.md)` emits a vector of quantile levels `q_\alpha` for each horizon step from a dedicated head, trained with the pinball loss:

```
L_pinball(y, q_alpha) = max(alpha * (y - q_alpha), (alpha - 1) * (y - q_alpha))
```

This removes sampling and the quantization precision ceiling.

**4. Flow matching.** `[Sundial](../papers/sundial.md)` learns a velocity field `v_theta(y_t, t)` mapping Gaussian noise to the data distribution; sampling is ODE integration from noise to data. Multiple seeds give the predictive distribution.

**5. Conformal wrapping.** `[TimeGPT-1](../papers/timegpt.md)` takes a point forecaster and computes calibrated intervals post hoc using conformal prediction, which gives theoretical coverage guarantees independent of model correctness.

## Why it works

The common thread is that a probabilistic objective is a richer supervision signal than MSE. For heavy-tailed targets, MSE lets a few outliers dominate gradients; NLL under a t-distribution down-weights them automatically. For multi-modal next-step distributions — imagine a series that either reverts to baseline or spikes — a unimodal Gaussian head collapses to the mean and predicts neither mode well, whereas a mixture head or categorical head represents both. Categorical sampling in particular is elegant because it lets the model approximate *any* predictive distribution non-parametrically via the empirical distribution of samples, exactly the Monte-Carlo estimator the forecasting community has been writing by hand for decades.

From an information-theoretic angle, NLL (or cross-entropy) is the KL divergence between the true and model distributions, so it trains the model to cover the posterior rather than chase its mode. Quantile regression is a robust alternative that loses the full distribution but directly targets the statistics most users care about (`q_{0.1}`, `q_{0.5}`, `q_{0.9}`). Conformal prediction, finally, is a distribution-free coverage trick: if the calibration residuals are exchangeable, `(1-\alpha)`-coverage is guaranteed regardless of the base forecaster.

## Trade-offs and failure modes

Parametric heads are efficient but *only as good as the assumed family*. Student-t is robust but unimodal; a mixture is more flexible but harder to optimize (the mixing weights can collapse and one component absorbs all the mass). Categorical sampling is non-parametric but caps precision at the bin width and can be slow (sampling `M` trajectories is `M×` the inference cost). Quantile regression drops the full distribution and predicts only pre-chosen quantiles — you cannot answer "what is the probability that y > 5?" without interpolation hacks, and quantile crossings (`q_{0.9} < q_{0.1}` for some step) are a real nuisance that ad-hoc sorting only partially fixes.

Flow matching is elegant but currently compute-heavy at inference: each forecast is an ODE integration, and the probabilistic output requires multiple seeds. Conformal prediction guarantees *marginal* coverage, not conditional — intervals are right on average but can be too wide or too narrow on specific regimes.

A subtle failure mode across all approaches is *miscalibration*: a model can have great point accuracy and still produce intervals that are too narrow. Reporting [CRPS](../evaluation/metrics.md#21-crps--continuous-ranked-probability-score), pinball loss at multiple quantiles, or interval coverage is essential to diagnose this.

## Design choices in the literature

- `[Lag-Llama](../papers/lag-llama.md)` — Student-t head on a decoder-only stack, explicit choice of a heavy-tailed family for robustness to outliers.
- `[MOIRAI](../papers/moirai.md)` — mixture of Student-t, trading head complexity for multi-modal capacity at every horizon step.
- `[Chronos](../papers/chronos.md)` — categorical sampling over a mean-scaled uniform vocabulary, non-parametric probabilistic output from an unmodified T5.
- `[Chronos-2](../papers/chronos-2.md)` — quantile decoder emitting a fixed set of quantiles, no sampling, no vocabulary.
- `[Sundial](../papers/sundial.md)` — flow-matching velocity field, continuous-valued probabilistic output via ODE integration.
- `[TimeGPT-1](../papers/timegpt.md)` — conformal intervals on top of a point forecaster, guaranteed coverage independent of the backbone.
- `[Moirai 2.0](../papers/moirai-2.md)` — 9-level pinball quantile head on a decoder-only backbone, aligned with GIFT-Eval's CRPS metric; uses a depth-2 expand-collapse decoding procedure to autoregress a quantile *vector* without committing to the median or mismatching dimensions.
- *Counter-examples.* [TiDE](../papers/tide.md) (pre-FM MLP encoder-decoder) and [LaT-PFN](../papers/lat-pfn.md) / [TS-JEPA](../papers/ts-jepa.md) / [MTS-JEPA](../papers/mts-jepa.md) (the JEPA cluster) are deliberately *point-only* — a useful reminder that probabilistic output is not free for every paradigm. JEPA's latent-target loss has no probability head built in; LaT-PFN approximates a posterior predictive but emits 100-bin categorical decode, not a calibrated quantile family; TiDE optimizes pure MSE.

## Open questions

- **Is a parametric head ever optimal, or should we always go non-parametric?** Lag-Llama's Student-t vs Chronos's categorical sampling have no clean head-to-head ablation.
- **How to train probabilistic output for *long* horizons without compounding error?** Next-step NLL accumulates; direct multi-horizon heads (Chronos-2, Sundial) avoid the rollout but have their own biases.
- **Calibration under distribution shift.** Conformal methods need exchangeable residuals; real deployments violate this.
- **Joint distributions over variates.** Current heads emit marginal distributions per horizon step; truly joint multivariate predictive distributions are mostly open.
- **Compute cost of sampling-based methods.** Chronos and Sundial need many samples per forecast; how to match their CRPS with fewer samples is an open efficiency question.

## Papers that exemplify this

- `[Lag-Llama](../papers/lag-llama.md)` — Student-t parametric head on a decoder-only backbone; motivates a heavy-tailed choice for real-world series.
- `[MOIRAI](../papers/moirai.md)` — mixture-of-Student-t for multi-modal predictive distributions from a masked encoder.
- `[Chronos](../papers/chronos.md)` — probabilistic-by-construction via categorical sampling and dequantization across 42 datasets.
- `[Chronos-2](../papers/chronos-2.md)` — quantile decoder replacing the vocabulary, direct multi-quantile output per horizon step.
- `[Sundial](../papers/sundial.md)` — flow-matching objective yielding continuous probabilistic forecasts at [TimeBench](../datasets-benchmarks/timebench.md) scale.
- `[TimeGPT-1](../papers/timegpt.md)` — conformal prediction wrapping a commercial point forecaster for calibrated intervals.
- `[Moirai 2.0](../papers/moirai-2.md)` — nine-quantile pinball head on a decoder-only stack with an expand-then-collapse procedure for autoregressive multi-quantile decoding.

## Related wiki pages

- [Value quantization](value-quantization.md)
- [RevIN normalization](revin-normalization.md)
- [Zero-shot forecasting](zero-shot-forecasting.md)
- [Flow-matching continuous](../architectures/flow-matching-continuous.md)
- [Masked encoder](../architectures/masked-encoder.md)
- [Encoder-decoder T5](../architectures/encoder-decoder-t5.md)
