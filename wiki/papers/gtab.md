# Calibration of Google Trends Time Series

> **Short name:** `gtab` · **arXiv:** [2007.13861](https://arxiv.org/abs/2007.13861) · **PDF:** [local](../../papers/gtab_2007.13861.pdf) · **Date:** 2020-07 · **Venue:** CIKM 2020

**Authors:** Robert West (EPFL)

## Abstract
Google Trends returns search-interest time series for up to five queries per request, normalized so the largest value in the (query, time, region) window is 100 and every other value is rounded to an integer in `[0, 100]`. The interaction of "peak = 100" with integer rounding makes it impossible to (1) directly compare more than five queries on the same scale and (2) compare queries whose true search interests differ by orders of magnitude — unpopular queries can collapse to all-zero series when batched with popular ones. The paper proposes Google Trends Anchor Bank (G-TAB), a two-phase calibration procedure that places an arbitrary number of queries onto a single common scale by transitively chaining requests against a pre-built bank of anchor queries spanning the full popularity spectrum. The method is shipped as an open-source Python library at `github.com/epfl-dlab/GoogleTrendsAnchorBank`.

## Key contributions
- Names and quantifies the integer-rounding × batch-mixing problem that ruins naive Google Trends pipelines, including the 5-query-per-request cap and the all-zero collapse for unpopular queries paired with popular ones.
- Defines the *anchor bank*: a list of queries calibrated against one reference query through pairwise shingled requests, with calibrated maximum search interests `R_x = m*_x / m*_Q` propagated transitively via a shortest-path computation on a query graph weighted by rounding-error bound ratios.
- Derives the optimal anchor-bank spacing analytically: neighboring anchors should sit at a constant maximum ratio `c = 1/e ≈ 0.37` to minimize the cumulative bound ratio (Appendix A); the worst-case bound ratio is below 1.55 for any `r* > 10⁻⁷`.
- Demonstrates calibration empirically — 200 Bavarian towns spanning five orders of magnitude on one log axis (Fig. 1c), 100 soccer clubs (Fig. 3a) — at a mean cost of 1.4–2.0 Google Trends requests per online query (Fig. 1d, Fig. 3b), and ships the method as an installable Python package.

## Method at a glance
G-TAB has an offline phase that builds the anchor bank once per (region, time-span) and an online phase that calibrates each new query in a handful of requests. Offline: sample `n` anchor candidates from a popularity-stratified Freebase/ClueWeb pool, send `n − k + 1` shingled five-query requests so neighboring anchors co-occur, drop pairs where either max value falls below `τ = 10` to limit rounding error, then estimate every pairwise maximum ratio `r_xQ` against a reference query `Q` via the shortest path on a graph with edge weights `log η_xy`. An optimization round refits the bank to the `c ≈ 1/e` spacing using `k = 2` requests for the surviving neighbors. Online: given a new query `q`, binary-search the sorted anchor bank for an anchor `x*` whose ratio with `q` falls in `[ε, 1/ε]` (default `ε = 0.1`), then return the calibrated series `q · R_{x*} / m_q`.

### Sizes

Methodology paper — **no neural architecture, no forecasting model.** G-TAB is a calibration tool that places Google Trends queries on a common scale via a precomputed anchor bank + online binary search (~1.4–2.0 GT requests per calibrated query). The `(L, d_model, d_ff, heads, d_kv, params, patch, context)` tuple does not apply. Released as the Python package `gtab` at `github.com/epfl-dlab/GoogleTrendsAnchorBank`.

## Why it matters
Without calibration, a multi-query Google Trends corpus has multiplicative scale errors — and outright zero-series collapses — that no downstream forecasting model can recover. G-TAB is the standard reference for the methodology of placing heterogeneous-popularity queries on one scale, and its open-source implementation makes it the default Step 2 in any Google-Trends pipeline that touches more than a single popularity band. The wiki's [rebuilding-google-trends-corpus](../benchmarks/rebuilding-google-trends-corpus.md) recipe uses G-TAB as the popularity-filter and stitching anchor in §5 and §7.

## Strengths
- Clean problem statement: the worked example with five Bavarian towns (Fig. 1) makes the rounding-collapse failure mode unmissable, and the calibrated 200-town log-axis plot makes the fix visually obvious.
- Theoretical optimality argument: Appendix A proves that equidistant anchors at `c = 1/e` minimize the global bound ratio, with a worst-case `< 1.55` for `r* > 10⁻⁷`.
- Cheap online cost: median 1–2 Google Trends requests per calibrated query in both reported example domains, against unbounded waste in any approach that re-batches popular and unpopular queries.
- Open, maintained Python implementation at `github.com/epfl-dlab/GoogleTrendsAnchorBank`, reusable as a library.
- Addresses a problem the foundational nowcasting literature (Choi & Varian 2011; Scott & Varian 2014) does not explicitly handle, since those papers work within a narrow popularity band where rounding-collapse does not bite.

## Limitations and open critiques
- Each calibration adds Google Trends requests on top of the data fetch — typically 1–6 per query during binary search plus the offline anchor-bank construction, which inflates the rate-limit budget for any pipeline running on `pytrends` or a similar scraper.
- Calibration drift over time. The anchor bank is built for a fixed (region, time-span); shifting the time window or the geography requires rebuilding it, and the paper does not analyze how stable an anchor bank is across multi-year windows.
- G-TAB does not address the orthogonal Bernoulli-resampling noise: Medeiros & Pires (`gtrends-proper_2104.03065.pdf`) show that two identical Google Trends requests return materially different series because each is a fresh draw of the underlying query logs. Users who care about variance still need repeated-sample averaging on top of G-TAB.
- The method assumes the calibration target is a *maximum-ratio* on a per-(query, time-span) basis. It does not fix the categorical-aggregation behavior of plain-text vs. Freebase-entity queries, nor the post-2020 normalization changes that Google rolled out for some geographies.
- The paper's evaluation is illustrative rather than comparative: there is no head-to-head benchmark against, e.g., `trendecon`'s frequency-reconciliation approach or against repeated-sampling baselines, so the precise marginal gain over alternatives is left to the reader.

## Follow-up work and dialogue
G-TAB and Medeiros & Pires (2021) ([gtrends-proper](./gtrends-proper.md)) attack different facets of the same data source. G-TAB fixes cross-batch scale and rounding; Medeiros-Pires fixes within-batch sampling-noise via repeated draws. The two are complementary rather than competing, and any TimesFM-quality Google Trends corpus needs both — see [rebuilding-google-trends-corpus](../benchmarks/rebuilding-google-trends-corpus.md) §5–6, where G-TAB sits at Step 2 (anchor bank) and §6 (per-query calibration) and Medeiros-Pires-style averaging sits at Step 6. [TimesFM](./timesfm.md) does not cite G-TAB explicitly, but its head-query selection and four-granularity stitching face the same calibration constraints that G-TAB and `trendecon` codify; the TimesFM paper's §7 deliberately does not release the calibration protocol it used. The classic nowcasting line — [Choi & Varian](./choi-varian.md), Scott & Varian, Ross — works in narrow-popularity regimes where the rounding-collapse problem is masked, so G-TAB is the right citation when extending those methods to broad-coverage corpora.

## Reproducibility
- **Open artifacts:** G-TAB source code at `github.com/epfl-dlab/GoogleTrendsAnchorBank`.
- **Code:** Python package `gtab`, pip-installable.
- **Data:** —
- **Compute:** lightweight — runs on a single laptop with API access; the cost is rate-limited Google Trends requests, not local compute.
- **Deployment footprint:** a pre-built default anchor bank ships with the package; custom banks for other regions or time-spans are constructed offline at the cost of a few thousand Google Trends requests.

## When to cite this paper
Cite G-TAB as the canonical reference for the Google-Trends integer-rounding-and-batch-mixing problem and for the anchor-bank-with-binary-search calibration methodology. It is the standard "you must do this before training on Google Trends data at scale" citation, complementary to [Medeiros & Pires](./gtrends-proper.md) (Bernoulli sampling noise) and to `trendecon` (frequency reconciliation across daily/weekly/monthly).

## In the knowledge graph
- **Cluster:** Pre-FM Google Trends methodology (calibration tooling; not part of the eight-cluster TS-FM taxonomy).
- **Methodology hub:** [Google Trends data](../datasets-benchmarks/google-trends-data.md), [scrub-tools](../datasets-benchmarks/scrub-tools.md), [rebuilding-google-trends-corpus](../benchmarks/rebuilding-google-trends-corpus.md).
- **Related concepts:** [data normalization](../concepts/data-normalization.md) (the "scale all 5 to peak 100" rule is a normalization choice that G-TAB inverts), [synthetic data augmentation](../concepts/synthetic-data-augmentation.md) (the leakage-free alternative when API-collected GT is too costly).
- **See also:** [Choi & Varian](./choi-varian.md) (foundational GT nowcasting, narrow popularity band, no calibration step), [Medeiros & Pires](./gtrends-proper.md) (orthogonal sampling-noise issue on the same data source), [TimesFM](./timesfm.md) (large-scale TS-FM consumer of Google Trends data whose calibration protocol is not disclosed).

## Related wiki pages
- [Google Trends data](../datasets-benchmarks/google-trends-data.md)
- [Rebuilding a TimesFM-quality Google Trends corpus](../benchmarks/rebuilding-google-trends-corpus.md)
- [Scrub tools and dataset loaders](../datasets-benchmarks/scrub-tools.md)
- [TimesFM](./timesfm.md)
