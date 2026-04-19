# Data Normalization for Time-Series Foundation Models

How raw time series are transformed before the model sees them, and
how predictions are transformed back into original units. This is the
most load-bearing preprocessing decision in a TS-FM pipeline — get it
wrong and zero-shot transfer collapses entirely.

## Intuition

A foundation model must handle series from wildly different domains:
energy demand in megawatts, stock prices in dollars, sensor readings in
millivolts. The raw values span orders of magnitude. Without
normalization, the model either wastes capacity learning domain-specific
affine transforms or simply fails to generalize. Normalization removes
the per-series level and scale, letting the model focus on shape and
dynamics — the parts that actually transfer across domains.

The reverse direction matters equally: after the model produces a
forecast in normalized space, the normalization must be inverted so the
output is in the original units. Any method that makes this inversion
lossy or error-prone (like differencing + cumsum) is at a disadvantage
vs methods where the inversion is an exact pointwise operation (like
RevIN).

## The normalization landscape

Every TS-FM uses one of these approaches. The field has strongly
converged on the first two.

### 1. RevIN (Reversible Instance Normalization)

Per-series, per-channel: subtract mean, divide by std. Model operates
in normalized space. Output: multiply by std, add mean.

```
x_norm = (x - mu) / sigma       # forward
y_raw  = y_norm * sigma + mu    # inverse (exact, pointwise)
```

This is the de facto standard. Used by [MOMENT](../papers/moment.md),
[Timer](../papers/timer.md), [Timer-XL](../papers/timer-xl.md),
[SEMPO](../papers/sempo.md), [PatchTST](../foundations/deep-learning-era.md),
and most masked-encoder and decoder-only TS-FMs. The deep dive on
RevIN mechanics, failure modes, and learnable variants is at
[revin-normalization.md](revin-normalization.md).

**Why it dominates:** the inverse is exact and pointwise — no error
accumulation, no information loss. The model trains on a proper loss in
original units (after inversion). Every series looks zero-mean
unit-variance regardless of domain.

### 2. Mean-scaling (absolute-value scaling)

Per-series: divide by the mean of absolute values. Output: multiply
back.

```
s      = mean(|x|) + eps
x_sc   = x / s
y_raw  = y_sc * s
```

Used by [Chronos](../papers/chronos.md) and
[Chronos-2](../papers/chronos-2.md) as the step before value
quantization into a discrete vocabulary. Not reversible in the
affine-std sense (no mean subtraction — the series keeps its shape,
just rescaled to unit magnitude), but operationally equivalent for the
purpose of cross-domain transfer.

**When to prefer over RevIN:** when downstream tokenization needs
non-negative or bounded values (Chronos's uniform quantizer maps
`[-C, C]` to bins, so centering around zero + unit scale is the right
preprocessing).

### 3. Partial-window instance normalization

Same as RevIN but statistics are computed on only a fraction of the
context window (e.g., the first 30%).

Used by [Moirai 2.0](../papers/moirai-2.md). The motivation: if the
series has a trend or regime shift within the window, using the full
window's mean and variance biases the normalization toward the
midpoint. Computing statistics on the initial segment aligns the
normalization with the "starting regime" and lets the model learn
the trend in normalized space.

**Trade-off:** fragile if the initial segment is unrepresentative
(e.g., contains an outlier or a flat segment before the signal starts).

### 4. Any-variate learned normalization

Per-variate affine transform with learned parameters (not just
instance statistics). Used by [MOIRAI](../papers/moirai.md) alongside
its multi-patch-size projections and any-variate attention.

### 5. Raw numerical text (no normalization)

[LLMTime](../papers/llmtime.md) feeds raw numerical strings directly
to a frozen LLM (GPT-3, Llama-2). No normalization at all — the LLM
sees digits like `"102.3, 105.7, 108.2"` and predicts the next digits.
This works surprisingly well for series whose numerical range happens
to be "LLM-friendly" (small integers, common decimal scales) but fails
for values outside the LLM's training distribution (e.g., sensor
readings in microvolts).

### 6. Z-score normalization (non-reversed)

Standard `(x - mu) / sigma` normalization applied to the training set,
but NOT reversed on the output — the model predicts and is evaluated in
normalized space. Common in LTSF benchmarks (Informer, Autoformer,
FEDformer) but NOT used by any foundation model, because it makes
cross-series comparison impossible (each series has its own scale in
the loss).

## Why not differencing?

Classical ARIMA makes a non-stationary series stationary by
differencing: `dx[t] = x[t] - x[t-1]`. An ARIMA(p, d, q) model
differences `d` times, fits a stationary ARMA(p, q) to the result, and
reconstructs the original scale by cumulative summation (inverse
differencing). It is natural to ask whether deep TS-FMs should do the
same.

**No published TS foundation model uses differencing as a preprocessing
step.** The reasons:

1. **Error accumulation under cumsum.** Predicted differences `dŷ` have
   some error `e_t` at each step. Cumulative summation turns this into a
   random walk: `ŷ[T] = y[0] + sum(dŷ[1:T]) = y[0] + sum(dy[1:T]) +
   sum(e[1:T])`. The error term `sum(e[1:T])` grows as `O(sqrt(T))` for
   unbiased errors, or linearly for biased ones. Over a long forecast
   horizon, a tiny systematic bias in predicted differences produces a
   large drift in reconstructed levels. RevIN's pointwise inverse has no
   such accumulation — each output timestep's error is independent.

2. **Loss function mismatch.** If the model predicts differences and the
   loss is MSE on differences, the model optimizes for `dŷ` accuracy,
   not `ŷ` accuracy. These are not equivalent after cumsum — a model
   that is excellent at predicting one-step differences can produce poor
   multi-step level forecasts because it never sees the cumulated error
   during training. Training on cumsum'd output is possible but
   expensive (backprop through cumsum at every horizon length).

3. **RevIN already handles non-stationarity.** The level and scale
   shifts that differencing removes are exactly the nuisance parameters
   that RevIN removes — but without changing the prediction target. The
   model still predicts levels (in normalized space), which is what
   downstream users care about.

4. **Patch tokenization is awkward after differencing.** A patch of
   16–32 raw values has natural structure (a short trajectory). A patch
   of 16–32 differences loses the absolute level information that the
   first value in the patch carries. The patch embedding must then learn
   to reconstruct level from context, which is exactly the work that
   RevIN already does for free.

5. **Multi-step probabilistic forecasting.** For probabilistic models
   (Chronos, MOIRAI, Lag-Llama, Sundial), the predictive distribution
   is over future *values*, not future *differences*. Converting a
   distribution over differences to a distribution over levels requires
   convolving T marginals — a non-trivial operation that most
   probabilistic heads (quantile, mixture-of-Student-t, flow-matching)
   do not support natively.

**The one partial exception:** [N-BEATS](../foundations/deep-learning-era.md)
and N-HiTS use a form of residual decomposition (trend + seasonal +
remainder) that is conceptually adjacent to differencing. But even they
operate on raw levels internally and decompose additively, not via
first-differences.

## Summary table

| Method | Inverse | Error accumulation | Used by |
|---|---|---|---|
| RevIN (mean + std) | Exact, pointwise | None | [MOMENT](../papers/moment.md), [Timer](../papers/timer.md), [Timer-XL](../papers/timer-xl.md), [SEMPO](../papers/sempo.md), [TimesFM](../papers/timesfm.md), [TTM](../papers/ttm.md) |
| Mean-scaling | Exact, pointwise | None | [Chronos](../papers/chronos.md), [Chronos-2](../papers/chronos-2.md) |
| Partial-window norm | Exact, pointwise | None | [Moirai 2.0](../papers/moirai-2.md) |
| Any-variate learned | Exact, pointwise | None | [MOIRAI](../papers/moirai.md) |
| Raw text (none) | N/A | N/A | [LLMTime](../papers/llmtime.md) |
| Z-score (non-reversed) | Not reversed | N/A (eval in norm space) | LTSF baselines (not TS-FMs) |
| Differencing + cumsum | Cumsum (lossy) | O(sqrt(T)) to O(T) | **No TS-FM uses this** |

## Open questions

- **Robust RevIN.** Median/MAD instead of mean/std would handle
  heavy-tailed series and outlier-dominated windows. Almost no TS-FM
  reports ablations on robust instance statistics.
- **Trend-aware normalization.** If the series has a strong linear
  trend within the context window, subtracting the mean flattens it but
  the model must then learn the slope from the normalized signal. A
  detrend-then-normalize approach could help but adds complexity.
- **Normalization × quantization interaction.** Chronos's mean-scaling
  interacts with its bin grid in non-obvious ways; a finer treatment of
  this interaction is open (see [value quantization](value-quantization.md)).
- **Differencing as a learnable module.** Could a model learn to
  optionally difference (when it helps) and cumsum on the output, with
  the differencing degree as a latent variable? This would unify RevIN
  and ARIMA-style preprocessing. No paper has tried it.

## Related wiki pages

- [RevIN normalization](revin-normalization.md) — deep dive on RevIN
  mechanics, learnable affine variants, and failure modes
- [Patch tokenization](patch-tokenization.md) — how raw values become
  transformer tokens (normalization happens before patching)
- [Value quantization](value-quantization.md) — how Chronos maps
  normalized values to discrete bins
- [Probabilistic forecasting](probabilistic-forecasting.md) — why
  cumsum on predictive distributions is hard
- [Zero-shot forecasting](zero-shot-forecasting.md) — normalization is
  what makes zero-shot possible
- [Classical methods](../foundations/classical-methods.md) — ARIMA and
  the differencing tradition this page contrasts with
- [HP transfer across scales](hp-transfer-across-scales.md) — another
  preprocessing-level design decision that affects the whole pipeline
- [Decoder-only autoregressive](../architectures/decoder-only-autoregressive.md)
  — normalization choice interacts with the patch-level autoregressive
  loss used by this family
- [Masked encoder](../architectures/masked-encoder.md) — the masked
  reconstruction objective has its own normalization-vs-output-space
  subtlety (MOMENT uses RevIN, MOIRAI uses per-context standardization)
