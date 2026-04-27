# The Proper Use of Google Trends in Forecasting Models

> **Short name:** `gtrends-proper` · **arXiv:** [2104.03065](https://arxiv.org/abs/2104.03065) · **PDF:** [local](../../papers/gtrends-proper_2104.03065.pdf) · **Date:** 2021-04 · **Venue:** preprint (PUC-Rio working paper)

**Authors:** Marcelo C. Medeiros, Henrique F. Pires (Pontifical Catholic University of Rio de Janeiro / PUC-Rio)

## Abstract
Google Trends is a widely used free tool for forecasters, with a large literature claiming accuracy gains from search-interest covariates. The paper observes a fact this literature has largely ignored: each Google Trends request returns a *different* sample of underlying search activity, even when the search term, time window, and location are held fixed. Because Google serves only a small per-request sub-sample and rounds the result to the integer grid `[0, 100]`, two downloads of the same query produce numerically distinct series. The paper documents the magnitude of this sampling noise, shows it can drive arbitrary conclusions in downstream forecasting models, and proposes a simple repeated-sampling-and-averaging procedure that recovers a stable estimator of the underlying interest signal.

## Key contributions
- Documents that Google Trends responses are stochastic: each request is a fresh sub-sample of Google's full search logs, so two identical requests yield different series.
- Quantifies the noise on a real example — three independent samples of "GDP Growth" interest for Brazil in 2009-2019 have pairwise correlations as low as 0.496 (Table 1, Panel a); the equivalent US series has min correlation 0.516 (Panel b).
- Proposes a **repeated-sampling-with-averaging** procedure: pull the same (term, window, region) request multiple times, then average across samples pointwise. Averaging seven samples raises pairwise correlation between two disjoint average-of-seven series to 0.92 (Brazil) and 0.95 (US).
- Demonstrates downstream gains in two settings: a LASSO simulation where averaging up to 27 percentage points more correctly-selected variables (Table 2), and a Brazilian COVID-19 SARS nowcast where the averaged-input model has a Root Mean Squared Error up to 51% lower for new cases and ~6% lower than the per-sample mean for new deaths (Table 3).

## Method at a glance
For every (search term, time window, location) the researcher cares about, query Google Trends repeatedly across separate days, retaining each returned series. Align the resulting series by timestamp and compute the pointwise mean across samples. Use the averaged series — not any single download — as the covariate fed into the downstream forecasting or selection model. The protocol is intra-query: it stabilizes a single query against Google's per-request Bernoulli noise, and is orthogonal to the cross-batch calibration problem that GTAB addresses (where queries of different popularity must be put on a common scale). Both fixes stack cleanly.

## Why it matters
Medeiros-Pires is the canonical reference for "raw Google Trends data is noisier than it looks" and for the repeated-sampling-and-averaging fix. The [rebuilding-google-trends-corpus](../benchmarks/rebuilding-google-trends-corpus.md) recipe lifts this protocol verbatim as Step 6, stacked on top of GTAB calibration (Step 5) and trendecon-style frequency reconciliation (Step 8). Without it, a pretraining corpus assembled from raw Google Trends inherits Bernoulli noise that no downstream TS-FM can compensate for.

## Strengths
- Concrete, reproducible diagnostics: Table 1 correlation matrices and the three-sample plots (Figures 1, 2) make the sampling-noise problem visually inescapable.
- The proposed fix is one-paragraph simple — average N independent samples — and any practitioner can apply it without specialized tooling.
- Both a controlled LASSO simulation (Section 3) and a real-world COVID nowcast (Section 4) are reported, so the gain is documented at simulation and at deployment scale.
- Complements rather than competes with GTAB; the two methods address structurally different defects of Google Trends.

## Limitations and open critiques
- API cost is linear in the repetition count `N`. With Google Trends' undocumented but real rate limits, an N=10 pass on a TimesFM-scale 22k head-query corpus is a multi-month wall-clock commitment.
- The paper does not address cross-batch calibration: when popular and unpopular queries are co-requested, the integer-rounding rule collapses the unpopular query to all-zeros regardless of repetition count. GTAB is needed for that.
- The sampling-noise effect is most acute at lower-popularity queries; very popular terms (the paper cites COVID-19 in 2020) vary less across samples, so the marginal value of repeated sampling shrinks at the head.
- Empirical evidence is at monthly granularity over 2009-2019. Google Trends' API and sampling behavior have changed since (e.g., the post-2020 normalization shift noted in [google-trends-data.md](../datasets-benchmarks/google-trends-data.md)); Table 1 magnitudes should be treated as illustrative, not contractual.

## Follow-up work and dialogue
[GTAB](./gtab.md) (West, CIKM 2020) addresses the orthogonal cross-batch calibration problem — putting queries of different popularity on one common scale — and is the natural complement to this paper. The two are routinely cited together in the methodologically-careful Google-Trends literature (`trendecon`, Eichenauer et al. 2022 *Economic Inquiry*, and the [rebuilding-google-trends-corpus](../benchmarks/rebuilding-google-trends-corpus.md) recipe stack them explicitly). Earlier nowcasting work — [Choi & Varian](./choi-varian.md) (2011/2012) and the [Scott & Varian](./scott-varian.md) BSTS paper (2013/2014) — predates this critique and used single-sample Google Trends; their results are not invalidated but should be read against this paper's noise floor. [TimesFM](./timesfm.md)'s "head-query" extraction (~22k queries with high search interest over 15 years) implicitly relies on this paper's analysis: head queries are the regime where Bernoulli noise is small enough that a single pull approximates the underlying signal. The TimesFM paper does not document the sampling protocol it actually used.

## Reproducibility
- **Open artifacts:** —
- **Code:** —
- **Data:** Google Trends is re-extractable; the paper's Brazilian SARS application data is from public health-ministry registries.
- **Compute:** trivial — repeated API calls plus pointwise averaging; no model training beyond the LASSO regressions in Sections 3-4.
- **Deployment footprint:** Python script-level. The averaging procedure adds a multiplicative factor `N` to API request count and is stateless across queries.

## When to cite this paper
Cite Medeiros & Pires (2021) as the canonical reference for the claim that Google Trends responses are a stochastic per-request sub-sample, and for the repeated-sampling-and-averaging fix. It is the methodology citation when building any production-grade Google Trends pipeline (research nowcasting, TS-FM pretraining corpora, operational forecasting). For cross-batch calibration cite [GTAB](./gtab.md); for frequency reconciliation cite `trendecon` (Eichenauer et al. 2022).

## In the knowledge graph
- **Cluster:** Pre-FM Google Trends methodology (sampling stabilization). Not part of the eight-cluster TS-FM taxonomy.
- **Methodology hub:** [Google Trends data](../datasets-benchmarks/google-trends-data.md), [scrub-tools](../datasets-benchmarks/scrub-tools.md), [rebuilding-google-trends-corpus](../benchmarks/rebuilding-google-trends-corpus.md).
- **Related concepts:** [data normalization](../concepts/data-normalization.md), [synthetic data augmentation](../concepts/synthetic-data-augmentation.md).
- **See also:** [GTAB](./gtab.md) (orthogonal cross-batch calibration), [Choi & Varian](./choi-varian.md), [Scott & Varian BSTS](./scott-varian.md), [TimesFM](./timesfm.md) (head-query corpus implicitly relying on this paper's noise analysis).

## Related wiki pages
- [Google Trends data](../datasets-benchmarks/google-trends-data.md)
- [rebuilding-google-trends-corpus](../benchmarks/rebuilding-google-trends-corpus.md)
- [scrub-tools](../datasets-benchmarks/scrub-tools.md)
- [data normalization](../concepts/data-normalization.md)
