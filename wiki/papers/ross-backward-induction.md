# Nowcasting with Google Trends: A Keyword Selection Method

> **Short name:** `ross-backward-induction` · **PDF:** [local](../../papers/ross-backward-induction_2013.pdf) · **Date:** 2013 · **Venue:** Fraser of Allander Institute working paper (University of Strathclyde)

**Authors:** Andrew Ross (Fraser of Allander Institute, University of Strathclyde)

## Abstract
Aggregate Google Trends search volume has been shown to nowcast variables ranging from influenza outbreaks to unemployment, retail sales, and trading volatility, but the recurring obstacle in the literature is the absence of a principled procedure for choosing *which* keywords to feed a nowcasting regression. Ross proposes the "backward induction method" — extract candidate keywords from the referring-traffic logs of websites topically relevant to the target variable, then test each candidate as an additional regressor in a simple autoregressive nowcast and retain those that are statistically significant. The procedure is evaluated against the Bank of England's expert-picked benchmark keyword `jsa` for nowcasting UK unemployment growth and the monthly Claimant count.

## Key contributions
- A reproducible "backward induction" recipe for keyword discovery: pick dominant target-domain websites (here, UK Jobseeker portals), extract their top referring queries via SEO tools (Alexa, Semrush, Sistrix, AdWords Keyword Tool), then filter by significance in an AR(2)-plus-Trends regression.
- An empirical case study on UK unemployment growth (2004:M1–2012:M4) and the monthly Claimant count, testing 10 backward-induction-selected keywords against the Bank of England benchmark keyword `jsa` (McLaren & Shanbhogue 2011).
- Practical noise-reduction guidance specific to Google Trends: download each keyword in a separate single-keyword request rather than the 5-keyword batch (since dominant keywords degrade siblings under the peak-equals-100 normalization), and average the series across seven consecutive daily downloads to suppress Google's Bernoulli sampling noise.
- Demonstrates that the same selected keyword set transfers to a second target (Claimant count) and the author reports successful but unpublished extensions to UK house-price inflation and individual insolvencies.

## Method at a glance
The "backward" in the name is from game-theoretic backward reasoning, not from backward-elimination regression: the assumption is that relevant keywords have *already* been chosen — by users, when they typed queries that brought them to topically relevant websites — so the analyst's job is to recover and re-use those user-revealed keywords. Concretely: identify dominant websites in the target domain (Open Directory, Yahoo Directory, Alexa, or a plain Google search); use an SEO tool such as Sistrix to extract the top referring queries to those sites' subpages and subdomains (e.g., the Jobseekers subpage of `direct.gov.uk` rather than the homepage, for topical specificity); then add each candidate keyword `t_k` to a baseline AR(2) nowcasting regression `Δ(u_t) = α + β_1 Δ(u_{t-1}) + β_2 Δ(u_{t-2}) + γ t_k + ε_t` and rank candidates by their HC3-robust p-value, in-sample adjusted `R²` and AIC, and one-step-ahead out-of-sample RMSE / MAE.

## Why it matters
This is a practical reference for the keyword-selection bottleneck in Google-Trends-based nowcasting: [Choi-Varian 2011](./choi-varian.md) used Google Trends "Categories" (verticals) as a fixed top-down hierarchy; [Scott-Varian 2014](./scott-varian.md) automated selection inside a BSTS model with spike-and-slab priors over many candidate predictors; [Ferrara-Simoni 2020](./ferrara-simoni.md) combined Granger-style preselection with Lasso shrinkage. Ross supplies a complementary, lighter-weight, and more interpretable alternative — useful when the candidate pool is small enough that MCMC is overkill and when the analyst wants per-keyword interpretability rather than a posterior over coefficient inclusion.

## Strengths
- Algorithmic clarity: the recipe is short enough to fit on a notecard, and every step (website discovery, SEO extraction, AR-plus-Trends regression, significance ranking) is reproducible without specialist software.
- Engages directly with Google Trends' measurement noise: downloads each keyword in its own request to avoid the 5-keyword batch normalization collapse, and averages seven daily redraws — a 2013 paper that already anticipated the Bernoulli-resampling issue formalized later by [Medeiros & Pires 2021](../datasets-benchmarks/google-trends-data.md).
- Two distinct evaluation targets (unemployment growth and Claimant count) plus a within-month variant using only first-week-of-month Trends data, providing some evidence of robustness.
- Explicitly compares against an expert-picked benchmark keyword (`jsa`) rather than declaring victory in a vacuum.

## Limitations and open critiques
- Empirical scope is small: 10 candidate keywords, monthly UK macro indicators only, sample 2004:M1–2012:M4. No held-out validation across countries or domains beyond the unemployment family.
- Pre-mobile-search-era and pre-G-TAB: the calibration problems documented in [Google Trends data](../datasets-benchmarks/google-trends-data.md) — peak-rounding collapse, popular/unpopular query co-batching, post-2020 normalization shift — are not addressed.
- No theoretical guarantees on the selection procedure; significance is judged from a single regression per candidate without multiple-testing correction across the 10 candidates.
- Relies on third-party SEO tools (Sistrix, AdWords Keyword Tool) whose referring-keyword data is closed and changes over time, undermining long-term reproducibility of the seed list.
- Recovers expert-picked-comparable keyword candidates but does not cleanly outperform them on the in-sample / out-of-sample fit headline metrics; gains are partial and target-dependent. Subsequent literature (BSTS, Lasso) reports stronger automated-selection results on related targets.

## Follow-up work and dialogue
[Scott-Varian 2014](./scott-varian.md) and [Ferrara-Simoni 2020](./ferrara-simoni.md) are the more cited predictor-selection methods today, and both supersede Ross at the level of automated empirical accuracy. Ross's procedure remains a useful baseline when the candidate pool is small, the analyst wants per-keyword interpretability, or MCMC infrastructure is unavailable. The TS foundation-model lineage — [TimesFM](./timesfm.md), [Chronos](./chronos.md), and successors — sidesteps the keyword-selection problem entirely by treating Google Trends as one of many heterogeneous pretraining streams; explicit selection of "the right keywords for this target" is replaced by implicit selection inside large-scale neural pretraining. See [rebuilding-google-trends-corpus](../benchmarks/rebuilding-google-trends-corpus.md) §3d for the broader keyword-selection methodology landscape and §3e for why econometric selection methods (including this one) are the wrong tool for *corpus construction* even when they are the right tool for downstream nowcasting.

## Reproducibility
- **Open artifacts:** —
- **Code:** not released; the paper describes the procedure in prose with sufficient detail to reimplement.
- **Data:** UK unemployment and Claimant count from ONS (public); Google Trends queryable for the same keyword list (re-extractable, subject to Google's resampling noise); referring-keyword extraction relied on Sistrix and similar paid SEO tools.
- **Compute:** trivial — OLS regressions on 100 monthly observations.
- **Deployment footprint:** spreadsheet / R-script-level.

## When to cite this paper
Cite Ross 2013 as the canonical reference for the backward-induction (referring-keyword extraction) approach to Google-Trends keyword selection in nowcasting — the small-data, interpretable alternative to spike-and-slab BSTS or Lasso pre-screening, useful when the candidate pool is limited and per-keyword statistical interpretability matters. Also a useful citation for early documentation of the per-keyword single-request Google Trends extraction protocol and the seven-day-averaging noise-reduction trick.

## In the knowledge graph
- **Cluster:** Pre-FM Google Trends methodology (keyword selection; not part of the eight-cluster TS-FM taxonomy).
- **Methodology hub:** [Google Trends data](../datasets-benchmarks/google-trends-data.md), [rebuilding-google-trends-corpus](../benchmarks/rebuilding-google-trends-corpus.md).
- **Related concepts:** [scaling laws](../concepts/scaling-laws.md) (keyword selection is the small-data analog of TS-FM corpus curation; pretraining replaces it with implicit selection at scale).
- **See also:** [Choi-Varian](./choi-varian.md) (canonical predecessor; Trends "Categories" verticals as fixed seed), [Scott-Varian BSTS](./scott-varian.md) (spike-and-slab automated alternative), [Ferrara-Simoni](./ferrara-simoni.md) (preselection-plus-shrinkage alternative), [TimesFM](./timesfm.md) (TS-FM successor that subsumes explicit keyword selection inside pretraining).

## Related wiki pages
- [Google Trends data](../datasets-benchmarks/google-trends-data.md)
- [rebuilding-google-trends-corpus](../benchmarks/rebuilding-google-trends-corpus.md)
- [Choi-Varian](./choi-varian.md)
- [Scott-Varian](./scott-varian.md)
- [Ferrara-Simoni](./ferrara-simoni.md)
- [TimesFM](./timesfm.md)
