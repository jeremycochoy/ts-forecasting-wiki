# Nowcasting Growth using Google Trends Data: A Bayesian Structural Time Series Model

> **Short name:** `kohns-nowcast` · **arXiv:** [2011.00938](https://arxiv.org/abs/2011.00938) · **PDF:** [local](../../papers/kohns-nowcast_2011.00938.pdf) · **Date:** 2020-11 (revised 2022-05) · **Venue:** Journal of Forecasting

**Authors:** David Kohns (Heriot-Watt), Arnab Bhattacharjee (Heriot-Watt & NIESR)

## Abstract
This paper investigates the benefits of internet search data, in the form of Google Trends, for nowcasting real U.S. GDP growth in real time through the lens of mixed-frequency Bayesian Structural Time Series (BSTS) models. The authors augment and enhance both the model and the methodology to make it amenable to nowcasting with a large number of potential covariates: shrinking state variances toward zero to avoid overfitting, extending Scott-Varian's spike-and-slab variable selection to the more flexible Normal-Inverse-Gamma prior, and adapting the horseshoe global-local prior to BSTS. The empirical application to U.S. GDP growth and an accompanying simulation study show that the horseshoe-BSTS markedly improves on the original BSTS, with the largest gains in dense data-generating processes. Search terms with high posterior inclusion probability admit clear economic interpretation, capturing leading signals of economic anxiety and wealth effects.

## Key contributions
- Three methodology refinements to the [Scott-Varian BSTS](./scott-varian.md) framework, targeted at high-dimensional Google Trends predictor sets: (1) a non-centred state-space parameterisation that lets the data shrink trend / drift state variances to zero; (2) the Normal-Inverse-Gamma extension of spike-and-slab variable selection (Ishwaran et al. 2005); (3) the horseshoe global-local prior adapted to BSTS, sampled with the Bhattacharya et al. (2016) algorithm.
- The first BSTS-with-Google-Trends study to include the COVID-19 pandemic period in its out-of-sample evaluation for U.S. GDP, using only latest-vintage real-time data.
- Mixed-frequency variable selection via U-MIDAS skip-sampling: monthly Google Trends and macro covariates are vertically realigned to a 3K-column quarterly regressor matrix, so that the same shrinkage prior decides whether a given month-within-quarter is informative.
- A SAVS sparsification step (Ray-Bhattacharya 2018) applied draw-by-draw to horseshoe posterior samples, producing posterior inclusion probabilities that recover spike-and-slab-style interpretability inside a continuous-shrinkage model.
- Reports up to 40% gains in point and density nowcast accuracy versus the original BSTS during specific nowcast windows, with the largest improvements early in a quarter before macroeconomic data are released.

## Method at a glance
The BSTS observation equation decomposes quarterly real GDP growth into a stochastic trend `tau_t`, a drift `alpha_t`, an optional seasonal block `delta_t`, and a high-dimensional regression `x_t' beta` on Google Trends and macro covariates. The non-centred form rewrites trend and drift as `tau_t = tau_0 + sigma_tau * tilde-tau_t + ...`, with normal priors on the standard deviations `sigma_tau` and `sigma_alpha` so that the data can shrink them to zero. The 37 Google Trends topics and categories are sampled monthly; U-MIDAS skip-sampling stacks the three within-quarter monthly observations side-by-side and lets the regression prior pick which month-lag matters. States are drawn with the Chan-Jeliazkov (2009) precision sampler; regression coefficients are drawn under spike-and-slab, NIG, or horseshoe priors and optionally sparsified per draw via SAVS.

### Sizes

Methodology paper — **no neural architecture component.** Model class: extended BSTS-MIDAS state-space model with non-centred parameterisation + horseshoe / Normal-Inverse-Gamma priors for variable selection on ~37 Google Trends covariates, fit by Chan-Jeliazkov precision sampler + Gibbs MCMC. The `(L, d_model, d_ff, heads, d_kv, params, patch, context)` tuple does not apply. SAVS sparsification step (Ray-Bhattacharya 2018) applied draw-by-draw to horseshoe posterior samples to recover spike-and-slab-style interpretability. Compute is single-CPU Gibbs over a low-dimensional state-space + few-hundred-column regression — laptop-scale.

## Why it matters
This is the canonical reference for BSTS-with-Google-Trends in a regime-shift setting. Scott-Varian (2014) had established the Bayesian template for keyword-driven nowcasting under stationary conditions; Kohns-Bhattacharjee tighten the priors and the state-space parameterisation so that the same template survives the COVID-19 break, when traditional macro indicators lagged badly and faster signals such as Google Trends became disproportionately valuable. The paper also operationalises horseshoe shrinkage inside BSTS for the first time, giving a practical route to handling tens of correlated search-interest covariates without manual pre-screening.

## Strengths
- Real-time evaluation: only the latest available data vintages at each nowcast date are used (with second-vintage GDP for evaluation, following Carriero-Clark conventions), so reported gains are not driven by future revisions.
- Out-of-sample window covers the COVID-19 collapse, where most pre-pandemic nowcasting models broke; the horseshoe-BSTS variant retains accuracy when the original Scott-Varian BSTS does not.
- Simulation study spans sparse, dense, and intermediate data-generating processes; the horseshoe prior dominates in dense regimes while remaining competitive in sparse ones, addressing the standard worry that global-local priors over-shrink true zeros.
- Predictor pool is reasoned about explicitly: the authors use Google *topics* and *categories* rather than raw search strings to mitigate ambiguity (e.g., "jobs" vs. "Steve Jobs"), echoing the recommendations later formalised by Woloszko (2020).
- Density nowcasts are evaluated, not just point forecasts; BSTS produces calibrated intervals by construction, which the paper exploits to report log-predictive-score gains in addition to RMSFE.

## Limitations and open critiques
- Single target, single country: U.S. real GDP growth only. There is no evidence that the same prior tuning transfers to euro-area, emerging-market, or non-GDP indicators.
- BSTS spike-and-slab MCMC scales poorly with predictor count; the paper's 37-topic pool is small by modern standards. A TimesFM-scale Google Trends corpus (~22k head queries, see [rebuilding-google-trends-corpus](../benchmarks/rebuilding-google-trends-corpus.md)) is well outside the practical range of this method.
- The Google Trends raw-data quality issues are orthogonal to the model: the paper does not apply G-TAB calibration (West 2020) or Medeiros-Pires repeated-sample averaging, so its inputs inherit Bernoulli sampling noise. Improving the input pipeline would compound with the prior-tuning gains the paper claims.
- Pre-foundation-model: the entire approach is regression-on-handcrafted-features. It pre-dates the neural-pretraining alternative ([TimesFM](./timesfm.md), [Chronos](./chronos.md)) and therefore cannot exploit cross-series pretraining on a large heterogeneous Google Trends corpus.
- Seasonal component is dropped during estimation because of differing monthly-vs-quarterly seasonal patterns and the short Google Trends sample, which trims one of BSTS's headline strengths.

## Follow-up work and dialogue
This paper is a direct extension of [Scott & Varian (2014)](./scott-varian.md) into the COVID-era data and into modern shrinkage priors. It competes methodologically with [Ferrara & Simoni (2019/2020)](./ferrara-simoni.md), whose preselection-and-shrinkage pipeline is a frequentist alternative for the same nowcasting target, and shares a lineage with [Choi & Varian (2011/2012)](./choi-varian.md), the foundational predict-the-present paper. It does not address the raw-Google-Trends quality problem, which is solved in parallel by the G-TAB calibration recipe and Medeiros-Pires averaging — see [google-trends-data](../datasets-benchmarks/google-trends-data.md). The TS-FM lineage replaces the BSTS+regression substrate entirely with neural pretraining on much larger Google Trends corpora ([TimesFM](./timesfm.md) trained on ~22k head queries, ~0.5 B time points). Even so, Kohns-Bhattacharjee's COVID stress-test result remains a useful sanity check: a small interpretable Bayesian state-space model with strong priors can compete with or outperform much larger black-box approaches when the regime shifts and pretraining data does not yet reflect it.

## Reproducibility
- **Open artifacts:** —
- **Code:** Bayesian sampling routines extend the `bsts` R package of Steven Scott; the paper's specific Gibbs sampler with non-centred precision sampling is described algorithmically (Appendix A) but no public repository URL is extracted.
- **Data:** U.S. real GDP growth from FRED (real-time vintages via `FredFetch`); 37 Google Trends topics and categories, re-extractable through `pytrends` though not bundled with the paper; PMI from Quandl `ISM/MANPMI`.
- **Compute:** modest — Gibbs sampler over a low-dimensional state-space model with a few-hundred-column regression. Runs on a single workstation.
- **Deployment footprint:** R / Matlab script-level; no model weights, no inference server.

## When to cite this paper
Cite this paper as the canonical reference for BSTS + Google Trends in the COVID era, for the horseshoe and Normal-Inverse-Gamma prior extensions of the Scott-Varian template, and for the U-MIDAS-style mixed-frequency variable-selection construction. It is also the right citation when arguing that Bayesian shrinkage priors and a non-centred state-space parameterisation matter as much as model architecture for GT-based nowcasting under regime shifts.

## In the knowledge graph
- **Cluster:** Pre-FM Google Trends methodology (BSTS / COVID stress-test; not part of the eight-cluster TS-FM taxonomy).
- **Methodology hub:** [Google Trends data](../datasets-benchmarks/google-trends-data.md), [classical-methods](../foundations/classical-methods.md), [rebuilding-google-trends-corpus](../benchmarks/rebuilding-google-trends-corpus.md).
- **Related concepts:** [probabilistic forecasting](../concepts/probabilistic-forecasting.md) (BSTS produces calibrated density nowcasts by construction).
- **See also:** [Scott-Varian BSTS](./scott-varian.md) (predecessor), [Choi-Varian](./choi-varian.md) (foundational predict-the-present paper), [Ferrara-Simoni](./ferrara-simoni.md) (alternative preselection-and-shrinkage methodology), [TimesFM](./timesfm.md) (foundation-model successor for Google Trends data).

## Related wiki pages
- [Google Trends data](../datasets-benchmarks/google-trends-data.md)
- [classical-methods](../foundations/classical-methods.md)
- [rebuilding-google-trends-corpus](../benchmarks/rebuilding-google-trends-corpus.md)
