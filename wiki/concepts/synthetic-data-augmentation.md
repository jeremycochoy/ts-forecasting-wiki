# Synthetic Data Augmentation

## Intuition

Time-series foundation models have a data problem that NLP does not. The total volume of *truly diverse* public TS data is small — a few tens of billions of observations at most — compared to the trillions of tokens available for text. Scaling laws say you need both more parameters *and* more data, so TS FMs have to generate their own. Synthetic data augmentation means building a prior over plausible time-series processes and sampling from it at pretraining time, either to supplement real corpora or, in the extreme, to replace them. Done well, synthetic data covers regimes (periodicities, trends, noise profiles) that real corpora miss, and lets the model learn invariances that would otherwise require huge real datasets.

## Mechanics

Four techniques dominate the current literature.

**1. KernelSynth (Chronos).** Sample a function from a Gaussian process whose kernel is itself drawn from a random combination of base kernels:

```
base_kernels = [Linear, Periodic(p), RBF(l), WhiteNoise]
k_samp       = random_sum_or_product(base_kernels)   # random composition
mu           = 0
y_synthetic  = sample_GP(mu, k_samp, length=T)
```

Each sample is a smooth (or noisy) function with known structure: trend, periodicity, short-range correlation. Generating millions of such series is cheap and covers a wide spectrum of dynamics with controllable parameters.

**2. TSMix (Chronos).** Mixup-style augmentation on real series. Pick two or more series, draw convex weights `lambda ~ Dirichlet(alpha)`, and combine:

```
y_mix = sum(lambda_i * normalize(y_i))
```

The result is a novel series that inherits features from each source but is not literally any of them. This is cheap regularization that expands the effective training corpus.

**3. ARMA + trend + sinusoidal composition (TimesFM).** TimesFM's
synthetic generator (Appendix A.8 of Das et al.) builds each series from
four independently toggleable components:

```
I.   Piecewise linear trend    (2–8 random segments)
II.  ARMA(p, q)                (1 ≤ p, q ≤ 8; coefficients from
                                Gaussian or Uniform, then normalized)
III. Sine wave                 (random period in [4, max_context/2],
                                random phase)
IV.  Cosine wave               (same period/phase sampling as sine)
```

For each synthetic series: randomly enable or disable each of I–IV,
generate each component at length 2048, and sum them with uniformly
sampled random weights. With 50% probability the trend is applied
multiplicatively instead of additively (creating heteroskedastic
series where variance scales with level). Total: 3M series × 2048
time-points ≈ 6.1B synthetic data points.

This is the most explicit and controllable recipe in the TS-FM
literature — each component maps directly to a classical TS concept
(AR memory, MA smoothing, deterministic trend, seasonality). It is
computationally cheaper than KernelSynth (O(n) per series via AR
filtering vs O(n³) or O(n log n) for GP sampling) and produces richer
autoregressive structure at high (p, q) orders. The trade-off is that
the ARMA spectral profile is fixed per series (one set of poles/zeros),
whereas a composed GP kernel can create multi-scale interactions more
naturally.

The real+synthetic mixing ratio is a hyperparameter; TimesFM
interleaves synthetic with Google Trends and Wikipedia pageviews in a
~100B-point total corpus.

**4. Prior-data Fitted Networks (PFN, used by Mamba4Cast).** The most extreme version: pretrain *only* on synthetic data sampled from a rich prior over TS generative processes. At every training step, draw a fresh batch from the prior and train the model to approximate the *Bayesian posterior predictive* under that prior:

```
for step in 1..N:
    process_i  ~ prior over TS generative models
    (x_i, y_i) ~ process_i
    loss       = -log f_theta(y_i | x_i)   # model learns E[p(y|x) under prior]
```

At inference, the model implements amortized Bayesian forecasting: it has never seen the real series but produces calibrated predictions under its learned prior.

## Why it works

Synthetic data is a form of *inductive bias injection*. When you sample from a GP with a random periodic kernel, you are telling the model "series with these exact periodicities exist, learn to recognize them." You are not waiting for such series to appear by luck in a real corpus. For models with strong universal approximators (transformers, SSMs), the limiting factor is often not capacity but *coverage* — whether the training distribution spans the set of target dynamics — and synthetic data is the cheapest way to extend coverage.

The PFN framing makes the Bayesian interpretation explicit. If training minimizes `E[-log f_theta(y | x)]` where the expectation is over `(x, y)` drawn from the prior, then the optimal `f_theta` is the posterior predictive under the prior (standard argument: minimize KL to the true conditional). So a PFN-trained model is a *learned Bayesian oracle* — fast to evaluate, limited only by the quality of the prior. This is conceptually clean and explains why Mamba4Cast can compete with real-data models despite never seeing real data.

TSMix has a different flavor: it is the TS analog of NLP paraphrasing or image cutmix. Mixing two real series creates a point off the manifold but close enough to teach the model smoothness without over-regularizing. KernelSynth is closer to ImageNet-style data generation; PFN is closer to meta-learning a prior directly.

## Trade-offs and failure modes

Synthetic data is only as good as its prior. KernelSynth's GP family covers smooth, periodic, and noisy regimes but *not* the sharp step changes, heavy-tailed bursts, or long-range nonstationarity that real series exhibit. A model trained on too much KernelSynth can overfit to smoothness assumptions and underestimate extreme events. TSMix inherits whatever biases exist in the source series; it does not create genuinely new dynamics. PFN pretraining's output quality is capped by the prior — "garbage prior in, calibrated-but-useless-predictions out."

A practical failure mode is *prior-real gap*: synthetic and real data have different statistical fingerprints (autocorrelation decay, spectral shape), and a pretrained model can end up excelling at the former while struggling on the latter. Ablations in Chronos's paper show the right *blend* matters: too much synthetic hurts, too little hurts too.

Finally, synthetic data augmentation does not solve the *covariate* problem. Real series come with exogenous features, business context, and calendar effects that are hard to simulate. Models that depend on exogenous signals cannot be pretrained synthetic-only.

## Design choices in the literature

- `[Chronos](../papers/chronos.md)` — introduced KernelSynth and TSMix, demonstrated that adding them to the pretraining mix improves zero-shot accuracy across a 42-dataset benchmark.
- `[TimesFM](../papers/timesfm.md)` — mixed synthetic functions into a real-data backbone (Google Trends, Wikipedia pageviews) at ~100B points.
- `[Chronos-2](../papers/chronos-2.md)` — relies heavily on synthetic augmentation to construct the *sibling panels* needed to train its group-attention in-context mechanism — you cannot train ICL without sibling context, and synthetic data is the only scalable source of it.
- `[Mamba4Cast](../papers/mamba4cast.md)` — synthetic-only PFN pretraining, zero real series used; the model is an amortized Bayesian forecaster whose only reference is the prior.
- `[Sundial](../papers/sundial.md)` — [TimeBench](../datasets-benchmarks/timebench.md) (~1T points) combines real and synthetic components as part of its diverse mixture, enabling flow-matching training at scale.
- `[LaT-PFN](../papers/lat-pfn.md)` — synthetic-only PFN pretraining like Mamba4Cast, but with a *context-aware* prior: a triple-sampling scheme (`hyperprior → context → sub-context → series`) generates each batch as a small panel of mutually related synthetic series, so the model learns in-context posterior approximation rather than per-series fitting. Adapts the ForecastPFN prior with a normalized abstract time axis.

## Open questions

- **What is the right synthetic/real ratio?** There is no principled answer; current recipes use heuristics.
- **How rich must the prior be for PFN to generalize?** Mamba4Cast uses a specific prior; other priors give different behavior, and the Pareto frontier between prior complexity and downstream accuracy is unmapped.
- **Can synthetic data replace real data entirely for specific regimes?** For smooth and periodic series, probably yes; for heavy-tailed financial data, probably no.
- **Synthetic covariates and panels.** Generating realistic sibling structures for ICL training is an open design question.
- **Measurable prior-real gap.** How do we quantify and close the distributional gap between synthetic and real series without expensive downstream evaluation?

## Papers that exemplify this

- `[Chronos](../papers/chronos.md)` — KernelSynth (random-kernel GP samples) and TSMix (mixup across real series) alongside real pretraining data.
- `[TimesFM](../papers/timesfm.md)` — ARMA(p,q) up to (8,8) + piecewise linear trends + sin/cos waves, 4 components randomly composed, 3M series × 2048 = 6.1B synthetic points blended with Google Trends and Wikipedia pageviews in a ~100B-point total mixture.
- `[Chronos-2](../papers/chronos-2.md)` — heavy synthetic augmentation to generate sibling panels for in-context group-attention training.
- `[Mamba4Cast](../papers/mamba4cast.md)` — PFN pretraining from a rich TS prior, no real data, amortized Bayesian forecasting.
- `[Sundial](../papers/sundial.md)` — synthetic components within the ~1T-point TimeBench mixture, enabling flow-matching at scale.
- `[LaT-PFN](../papers/lat-pfn.md)` — context-aware synthetic prior with triple-sampling, generating per-batch panels of related series for in-context posterior approximation under a JEPA latent objective.

## Related wiki pages

- [Zero-shot forecasting](zero-shot-forecasting.md)
- [Scaling laws](scaling-laws.md)
- [Probabilistic forecasting](probabilistic-forecasting.md)
- [Lightweight non-transformer](../architectures/lightweight-non-transformer.md)
- [Flow-matching continuous](../architectures/flow-matching-continuous.md)
