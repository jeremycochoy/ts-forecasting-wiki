# Predicting the Present with Google Trends

> **Short name:** `choi-varian` · **PDF:** [local](../../papers/choi-varian_2012.pdf) · **Date:** 2011-12 (revised 2012) · **Venue:** Economic Record, 2012

**Authors:** Hyunyoung Choi, Hal Varian (Google)

## Abstract
Choi and Varian show that Google Trends search-volume indices, available in real time at daily and weekly resolution, can improve short-horizon forecasts of economic indicators that are otherwise released with a multi-week reporting lag and revised months later. The paper does not claim search data forecasts the future; it claims search data helps "predict the present" — i.e., contemporaneous nowcasting. Four canonical case studies demonstrate the recipe: U.S. motor vehicle and parts retail sales, U.S. seasonally adjusted initial unemployment claims, monthly visitor arrivals to Hong Kong by country of origin, and the Roy Morgan Australian Consumer Confidence Index. In each case a simple seasonal autoregressive baseline augmented with one or two Google Trends category indices outperforms the baseline by roughly 5–20% in mean absolute error.

## Key contributions
- Coined the "predicting the present" framing for nowcasting and tied it explicitly to the central-bank / government-statistics use case (paper §1, citing Castle et al. 2009 and McLaren & Shanbhoge 2011).
- Established the methodological template later inherited by the entire Google-Trends nowcasting literature: a baseline `AR(1)` (optionally with seasonal lag `y_{t-12}`) on `log y_t`, augmented with contemporaneous Google Trends category indices as exogenous regressors.
- Documented Google Trends' Insights for Search interface, the broad-match query semantics, the "max value normalized to 100" indexing rule, and the ~30 / ~250 hierarchical category taxonomy that subsequent corpus-builders rely on (paper §2).
- Ran the four canonical case studies — motor vehicles and parts, initial unemployment claims, Hong Kong tourism, Australian consumer confidence — with both in-sample fits and rolling-window out-of-sample evaluation, providing a reusable benchmark template.
- Introduced spike-and-slab Bayesian variable selection (George & McCulloch 1997) for picking which of Google's ~250 categories to use, the seed of the BSTS approach later formalized in [Scott & Varian 2014](./scott-varian.md).

## Method at a glance
For each target series the authors estimate a baseline of the form `y_t = b_1 y_{t-1} + b_{12} y_{t-12} + e_t` on `log y_t` (or a simpler `AR(1)` for already-seasonally-adjusted series like initial claims). They augment it with one or two Google Trends category indices selected by in-sample t-tests or by spike-and-slab posterior probability. Out-of-sample evaluation uses one-step-ahead rolling-window forecasting starting at observation `k = 17`, refitting on `[k, t-1]` for each `t`. Reported metric is mean absolute error of `log y_t`, with separate MAE numbers tabulated for the full sample and the December 2007 – June 2009 recession sub-sample.

### Sizes

Methodology paper — **no neural architecture component.** Model class: linear seasonal AR(1) with Google Trends category indices as additional contemporaneous regressors, fit by OLS in a rolling window. The `(L, d_model, d_ff, heads, d_kv, params, patch, context)` tuple does not apply. Variable selection is done either by in-sample t-tests (most case studies) or by spike-and-slab Bayesian posterior probability (Australian consumer-confidence case), which seeds the BSTS approach formalized in [scott-varian](./scott-varian.md).

## Why it matters
This is the canonical reference that put Google Trends on the macro-forecasting map. The "search interest as a leading or contemporaneous indicator" paradigm it codified is the basis for [Scott & Varian 2014](./scott-varian.md) (BSTS + spike-and-slab), [Ferrara & Simoni 2020](./ferrara-simoni.md) (theoretical pre-selection for GDP nowcasting), [Kohns & Bhattacharjee 2022](./kohns-nowcast.md) (horseshoe priors for COVID-era nowcasting), and the entire applied epidemiology / labor-economics line that uses Google search counts as exogenous covariates. A decade later, the same paradigm underwrites [TimesFM](./timesfm.md)'s decision to build ~50% of its public pretraining corpus from Google Trends (~22k head queries, ~0.5B time points; see [Google Trends data](../datasets-benchmarks/google-trends-data.md)).

## Strengths
- Concrete, replicable wins on the four case studies. Motor vehicles and parts: MAE drops from 6.34% (baseline AR with seasonal lag) to 5.66% with Trends — a 10.6% relative improvement; during the 2007–2009 recession the improvement widens to 21.4% (paper §3.1, MAE comparison after Figure 2).
- Initial unemployment claims: full-sample MAE goes from 3.37% to 3.68% (a slight regression) but during the recession Trends reduces MAE from 3.98% to 3.44%, a 13.6% improvement, with Table 1 documenting MAE reductions at all four turning points (paper §3.2). Interpretation: search data buys turning-point sensitivity.
- Hong Kong visitor arrivals from nine source countries achieve an average in-sample R² of 73.3% (excluding Japan) using only a single Vacation Destinations / Hong Kong category index plus seasonal lags (paper §3.3, after Figure 5).
- Australian consumer confidence: the spike-and-slab procedure flags Crime & Justice, Trucks & SUVs, and Hybrid & Alternative Vehicles as the most posterior-probable predictors; the resulting model reduces in-sample MAE by 12.7% and one-step-ahead out-of-sample MAE by 9.3% over an `AR(1)` baseline (paper §4).

## Limitations and open critiques
- Data-vintage problems are not addressed. The paper uses contemporaneously available revisions of the target series rather than the real-time vintages a forecaster would have had — the well-known "real-time data" issue Castle et al. 2009 raised but Choi-Varian explicitly do not solve.
- Pre-mobile-search era. The data window is 2004 – mid-2011, before the mobile-search transition reshaped query distributions; category-mix stability across that boundary is not audited.
- Google Trends' category taxonomy in 2011 (~30 top-level, ~250 second-level categories) has since been revised, so direct replication of the four case studies today produces somewhat different category indices.
- Sampling-noise issues — that each Google Trends request is a fresh Bernoulli draw and returned values vary across calls — are not yet recognized. Identification of this problem and the repeated-sampling fix come later, in [Medeiros & Pires 2021](./gtrends-proper.md).
- Cross-popularity calibration is not addressed. Mixing a popular and a rare query in one request collapses the rare query to integer-rounded zeros; the fix ([GTAB](./gtab.md), West 2020) postdates this paper by nearly a decade.
- Variable selection in three of four cases is informal (in-sample t-tests). Only the consumer-confidence case uses spike-and-slab, which is the procedure that survives in the modern literature.

## Follow-up work and dialogue
[Scott & Varian 2014](./scott-varian.md) replaced the ad-hoc OLS+t-test selection with a full Bayesian Structural Time Series model and made spike-and-slab the default Google-Trends variable-selection method. [Medeiros & Pires 2021](./gtrends-proper.md) flagged the Bernoulli-resampling problem and showed that naïve single-sample analyses can produce arbitrary keyword-selection conclusions, prescribing repeated-sample averaging. [West 2020](./gtab.md) (GTAB) addressed the orthogonal calibration / zero-bucket problem, providing an anchor-bank protocol that lets queries of wildly different popularity sit on a common scale. [Ferrara & Simoni 2020](./ferrara-simoni.md) gave a theoretical pre-selection-and-shrinkage foundation for picking GT regressors for GDP nowcasting, and [Kohns & Bhattacharjee 2022](./kohns-nowcast.md) extended Scott-Varian's BSTS with horseshoe priors for COVID-era nowcasting under heavier-tailed sparsity. A decade after this paper, [TimesFM 2024](./timesfm.md) elevated Google Trends from a hand-picked-categories regression covariate to the largest public component of a foundation-model pretraining corpus (~0.5B time points across four granularities), and the lineage is direct: TimesFM cites the search-interest-as-signal premise that Choi & Varian made canonical.

## Reproducibility
- **Open artifacts:** R source code and data referenced in the paper as forthcoming in an online appendix (paper §1, "URL to be provided"); no permanent location pinned in the published version.
- **Code:** R, freely available via CRAN; specific scripts not extracted from the paper.
- **Data:** Google Trends category indices used are re-extractable today via the public web UI, [`pytrends`](../datasets-benchmarks/google-trends-data.md), or [GTAB](./gtab.md). Target series are public: U.S. Census Advance Retail and Food Services (motor vehicles and parts), U.S. DOL initial unemployment claims, Hong Kong Tourism Board monthly visitor arrivals, Roy Morgan Consumer Confidence.
- **Compute:** trivial — linear regressions and `stl` seasonal decomposition in R.
- **Deployment footprint:** —

## When to cite this paper
Cite Choi & Varian as the canonical reference for the claim that Google Trends category indices are useful contemporaneous predictors of near-term economic activity, and for the "predicting the present" / nowcasting framing. Cite it for the four canonical case studies (motor vehicles, initial unemployment claims, Hong Kong tourism, Australian consumer confidence), which are the standard demonstration set the GT-nowcasting literature still benchmarks against. For Bayesian variable selection over GT categories, prefer [Scott & Varian 2014](./scott-varian.md); for sampling-noise mitigation, [Medeiros & Pires 2021](./gtrends-proper.md); for cross-popularity calibration, [GTAB](./gtab.md).

## In the knowledge graph
- **Cluster:** Pre-FM Google Trends methodology (foundational; not part of the eight-cluster TS-FM taxonomy).
- **Methodology hub:** [Google Trends data](../datasets-benchmarks/google-trends-data.md), [rebuilding-google-trends-corpus](../benchmarks/rebuilding-google-trends-corpus.md).
- **Related concepts:** [synthetic data augmentation](../concepts/synthetic-data-augmentation.md) (counter-thread — TimesFM blends GT with synthetic series to compensate for narrow corpus character), [zero-shot forecasting](../concepts/zero-shot-forecasting.md) (the modern TS-FM analog of nowcasting: train once, predict an unseen target with no task-specific fitting).
- **See also:** [Scott & Varian BSTS](./scott-varian.md), [Ferrara & Simoni](./ferrara-simoni.md), [Kohns & Bhattacharjee](./kohns-nowcast.md), [GTAB](./gtab.md), [Medeiros & Pires](./gtrends-proper.md), [TimesFM](./timesfm.md).

## Related wiki pages
- [Google Trends data](../datasets-benchmarks/google-trends-data.md)
- [rebuilding-google-trends-corpus](../benchmarks/rebuilding-google-trends-corpus.md)
