# Predicting the Present with Bayesian Structural Time Series

> **Short name:** `scott-varian` · **PDF:** [local](../../papers/scott-varian_2014_bsts.pdf) · **Date:** 2013-06 · **Venue:** International Journal of Mathematical Modelling and Numerical Optimisation, 2014

**Authors:** Steven L. Scott, Hal Varian (Google)

## Abstract
The paper presents a short-term forecasting system that combines a structural time series model — local linear trend plus annual seasonal component — with a regression component that consumes a very high-dimensional set of contemporaneous Google Trends predictors. A spike-and-slab prior on the regression coefficients induces sparsity, dramatically reducing the effective regression dimension. MCMC sampling produces an ensemble of model configurations, and forecasts are averaged across this ensemble (Bayesian model averaging). The authors illustrate the method on weekly U.S. initial unemployment claims (top 100 Google Correlate queries) and monthly seasonally-adjusted U.S. retail sales (100 Correlate queries plus ~600 Google Trends verticals).

## Key contributions
- A unified state-space formulation decomposing the target into trend `mu_t`, seasonal `tau_t`, regression `beta^T x_t`, and noise — all additive, jointly estimated by the Durbin-Koopman simulation smoother (paper §3.1, equation 3).
- A spike-and-slab prior on the regression coefficients (paper §3.2.1, equations 4-6) that places positive probability *mass* on `beta_k = 0`, unlike L1-based sparsity (lasso) which only assigns density at zero.
- User-facing prior elicitation reduced to three knobs — expected model size, expected `R^2`, prior sample size `nu` — with defaults (`R^2 = 0.5`, `nu = 0.01`, `pi_k = 0.5`).
- Bayesian model averaging across MCMC draws as a built-in hedge against picking the wrong predictor subset; forecasts are samples from `p(tilde y | y)` (paper §4.2).
- A reference implementation in C++ with an R interface — the `bsts` package, widely adopted.

## Method at a glance
The model is `y_t = mu_t + tau_t + beta^T x_t + epsilon_t` with a local-linear-trend transition and a 52-week dummy-style seasonal `tau_t` (paper §3.1, equation 3). The regression component is appended to the observation matrix by adding a constant 1 to the state, so the state dimension grows by only one regardless of how many candidate predictors `x_t` carries (paper §3.2). MCMC alternates between drawing latent state via the Durbin-Koopman simulation smoother, drawing variance parameters from inverse-Gamma full conditionals, and drawing `(gamma, beta, sigma_epsilon^2)` via SSVS (George-McCulloch 1997) on the marginal in equation 8 (paper §4.1).

### Sizes

Methodology paper — **no neural architecture component.** Model class: Bayesian Structural Time Series (BSTS) with local-linear-trend + 52-week seasonal + spike-and-slab regression on contemporaneous Google Trends predictors, fit by Kalman filter + Forward-Filtering-Backward-Sampling (FFBS) for the state and SSVS MCMC over inclusion indicators. The `(L, d_model, d_ff, heads, d_kv, params, patch, context)` tuple does not apply. Case studies use ~100 (initial claims) to ~700 (retail sales) candidate Google Trends predictors; median posterior model size is 9, with 95th percentile 12. MCMC: 5,000 iterations (1,000 burn-in) on a single CPU in roughly 105 seconds for the case-study scale. Released as the `bsts` R package.

## Why it matters
This is the canonical reference for "BSTS plus Google Trends nowcasting" and the methodological successor to Choi-Varian's 2009/2012 regression approach: where Choi-Varian required hand-curated query selection, scott-varian automates it via spike-and-slab. The framework is what [kohns-nowcast](./kohns-nowcast.md) explicitly extends to the COVID era with horseshoe priors. The `bsts` R package, written by Scott himself, shipped this approach into production-economics use, where it remains a standard nowcasting baseline.

## Strengths
- On weekly U.S. initial unemployment claims, the BSTS-with-Google-Trends model accumulates one-step-ahead absolute error at a roughly constant rate across the 2008-2009 recession, while the pure time-series baseline (no regression component) experiences a large error jump at the recession onset (paper §5.1, Figure 6). Recession turning points — the hardest part of nowcasting — are where the GT signal helps.
- The spike-and-slab prior produces visibly sparse posteriors. On initial claims, 67% of MCMC-sampled models have at most 3 predictors and 98% have at most 5; on retail sales (184 candidates), median model size is 9 with 95th percentile 12 (paper §5.1 and §5.2). Model averaging hedges *across* small models rather than betting on one large fit.
- Selected predictors are economically interpretable. Initial-claims top selections are `unemployment.office`, `filing.for.unemployment`, `idaho.unemployment` (with state-switching across runs — hedging, not defect; paper §5.1, Figure 3). For retail sales, the top retained verticals are `Real Estate / Home Financing` and `Social Services / Welfare / Unemployment`, both with negative coefficients consistent with recession dynamics (paper §5.2, Figure 8).

## Limitations and open critiques
- Single-domain case studies. Two U.S. macroeconomic series; generalization to other geographies, sectors, or non-economic series is asserted but not benchmarked.
- Spike-and-slab MCMC scales poorly as the candidate set grows. The case studies use ~100-700 predictors; modern GT seed lists run into the tens of thousands, where SSVS mixing degrades.
- Pre-mobile-search-era data. Initial claims study runs Jan 2004 – mid-2012; query semantics shifted after the smartphone transition.
- Does not handle the Bernoulli-sampling noise that [Medeiros & Pires (2021)](../datasets-benchmarks/google-trends-data.md) flag as the dominant source of irreproducibility in raw GT data — selection statistics on a single sample can be misleading. Repeated-sample averaging *before* BSTS is the right practice, and scott-varian predates this recommendation.
- Subjective curation is still required to remove spurious predictors. The paper itself flags `Science / Scientific Equipment` as an example: it spiked during the 2008 crisis because of Large Hadron Collider news coverage, not economic relationship (paper §6).
- For TS-FM use cases, the per-target supervised regression is incompatible with cross-domain pretraining; the lineage is replaced by direct neural pretraining on much larger GT corpora (TimesFM-style).

## Follow-up work and dialogue
[Kohns & Bhattacharjee (2022)](./kohns-nowcast.md) extends the BSTS framework to mixed-frequency COVID-era nowcasting, replacing spike-and-slab with horseshoe priors for heavier-tailed sparsity and showing in sensitivity analyses that keyword-selection protocol matters roughly as much as model architecture. [Ferrara & Simoni (2020)](./ferrara-simoni.md) compete with a frequentist alternative — pre-screening (targeting methods) followed by shrunk bridge regressions — applied to euro-area GDP nowcasting. [Medeiros & Pires (2021)](../datasets-benchmarks/google-trends-data.md) flag a problem affecting all three: any GT-based selection statistic on a single sample is unreliable because the request returns a Bernoulli draw, and conclusions can flip across resamples. The modern TS-FM lineage — [TimesFM](./timesfm.md), Chronos — sidesteps keyword selection by pretraining on tens of thousands of head queries directly, treating GT as a bulk pretraining corpus. See [classical-methods](../foundations/classical-methods.md) for BSTS's place in the broader structural-state-space lineage.

## Reproducibility
- **Open artifacts:** the `bsts` R package by S. L. Scott; widely used in production economics.
- **Code:** R package `bsts` on CRAN; the paper-§5.1 implementation runs MCMC for 5,000 iterations (1,000 burn-in) on a 3.4 GHz Linux workstation in roughly 105 seconds.
- **Data:** weekly U.S. initial unemployment claims (FRED) and monthly U.S. retail sales (FRED, seasonally adjusted, ex-food-services); Google Trends queries are re-extractable via current GT tooling, modulo the post-2012 calibration concerns documented in [google-trends-data](../datasets-benchmarks/google-trends-data.md).
- **Compute:** modest — single-CPU MCMC; no GPU dependency.
- **Deployment footprint:** R package, runs on a laptop. Scales to ~hundreds of predictors comfortably; thousands stress SSVS mixing.

## When to cite this paper
Cite scott-varian as the canonical reference for **BSTS-plus-Google-Trends nowcasting**, for the **spike-and-slab prior used as automated variable selection inside a structural state-space model**, and for the **Bayesian-model-averaging interpretation of MCMC over predictor subsets**. It is also the right citation for the lineage connecting Choi-Varian-style hand-curated GT regression to the horseshoe-BSTS extensions of Kohns-Bhattacharjee. See [rebuilding-google-trends-corpus § 3d](../benchmarks/rebuilding-google-trends-corpus.md) for the published-methodology context.

## In the knowledge graph
- **Cluster:** Pre-FM Google Trends methodology (BSTS branch; not part of the eight-cluster TS-FM taxonomy).
- **Methodology hub:** [Google Trends data](../datasets-benchmarks/google-trends-data.md), [rebuilding-google-trends-corpus](../benchmarks/rebuilding-google-trends-corpus.md), [classical-methods](../foundations/classical-methods.md).
- **Related concepts:** [zero-shot forecasting](../concepts/zero-shot-forecasting.md) (the modern analog — pretrained priors instead of per-target spike-and-slab selection), [probabilistic forecasting](../concepts/probabilistic-forecasting.md) (BSTS is probabilistic by construction; forecasts come from `p(tilde y | y)`).
- **See also:** [Choi-Varian](./choi-varian.md) (predecessor — hand-curated GT regression), [Ferrara-Simoni](./ferrara-simoni.md) (frequentist preselection-and-shrinkage alternative), [Kohns-Bhattacharjee](./kohns-nowcast.md) (BSTS-COVID extension with horseshoe priors), [TimesFM](./timesfm.md) (uses Google Trends at corpus scale instead of per-target selection).

## Related wiki pages
- [Google Trends data](../datasets-benchmarks/google-trends-data.md)
- [classical-methods](../foundations/classical-methods.md)
- [rebuilding-google-trends-corpus](../benchmarks/rebuilding-google-trends-corpus.md)
