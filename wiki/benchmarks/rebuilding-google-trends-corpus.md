# Rebuilding a TimesFM-Quality Google Trends Corpus

This page is a practical recipe for assembling a large Google Trends
time-series corpus — of the scale and quality used by
[TimesFM](../papers/timesfm.md) — from scratch, and extending it to
**cover present-day data** (not just 2007–2022). It is the
Google-Trends-specific sibling of
[rebuilding-timebench.md](rebuilding-timebench.md).

The background knowledge (what Google Trends returns, why naïve
access fails, what `pytrends` / `G-TAB` / `trendecon` do) lives in
[../datasets-benchmarks/google-trends-data.md](../datasets-benchmarks/google-trends-data.md).
Read that first if you have not seen it.

## 1. What "TimesFM quality" means

TimesFM §2 "Data" and Table 1 describe the original corpus you are
trying to match or exceed. The benchmark to beat:

| Property | TimesFM v1 | Your target |
|---|---|---|
| Number of queries | ~22,000 "head queries" | ≥22k; more is fine |
| Query selection criterion | "Head queries based on search interest over 15 years from 2007 to 2022" — sparse below top-22k | Same: pick the top popularity band |
| Hourly granularity | Jan 2018 – Dec 2019 | Extend through today |
| Daily, weekly, monthly | Jan 2007 – Dec 2021 | Extend through today |
| Total time-points | ~0.5 B | Proportional to query count × coverage |
| Release? | No — exact slice not published | Yours should be releasable |
| Calibration? | Undisclosed | G-TAB (see §5) |
| Sampling noise mitigation? | Undisclosed | Repeated sampling with averaging |

The two gaps in TimesFM's release — the specific query list and the
calibration protocol — are the reasons this rebuild exists. You are
not trying to clone TimesFM bit-for-bit; you are trying to produce a
corpus that is (a) comparable in scale and diversity, (b) correctly
calibrated, (c) extended to recent data, and (d) releasable.

## 2. End-to-end pipeline at a glance

```
  Seed-query  ─┐
  selection    │
               ▼
        [head-query list, ~25-50k terms]
               │
               ▼
  Anchor-bank ─┐         Offline, one-time (G-TAB build)
  construction │
               ▼
       [anchor bank covering full popularity spectrum]
               │
               ▼
  Per-query    │         For each query × granularity:
  calibrated   │          • request at g (hourly/daily/...)
  download     │          • calibrate against anchor bank (G-TAB)
               │          • repeat N times and average (Medeiros)
               ▼
    [(query, granularity, calibrated series) tuples]
               │
               ▼
  Temporal     │         Stitch sub-windows across the full date
  stitching    │         range (earlier years need scale matching)
               ▼
    [one long, calibrated time-series per (query, granularity)]
               │
               ▼
  Frequency    │         Chow-Lin disaggregation (trendecon):
  reconciliation         ensure daily ≅ weekly ≅ monthly are
               ▼         mutually consistent
    [frequency-consistent daily / weekly / monthly series]
               │
               ▼
  Quality      │         Variance checks, consistency audits,
  audits       │         comparison with Google Trends Datastore
               ▼         for sanity on shared queries
         [final corpus]
               │
               ▼
  Storage + ─ publication (HuggingFace Dataset + tooling repo)
```

Sections 3-10 below walk each stage.

## 3. Pick the seed query list

This is the highest-leverage decision. Bad query selection → corpus
is narrow or noise-dominated. Good practices below; see §3d for the
published-methodology references.

### 3a. Mix of discovery sources

- **Google Trends categories (verticals) as the top-down spine.**
  Google Trends organizes queries under ~25 top-level categories
  (e.g., `All Categories > Jobs and Education > Jobs`, which
  includes "jobs", "resume", "careers"). Walking the vertical tree
  is the canonical starting seed — see the Ross 2013 and
  Choi & Varian 2011 references in §3d below.
- **Google Trends' own top-charts.** "Year in Search" top lists per
  year, 2001–present. The `ronantakizawa/trending-words-google`
  HuggingFace dataset (described in
  [../datasets-benchmarks/google-trends-data.md § 6](../datasets-benchmarks/google-trends-data.md))
  has 2,784 entries — a clean source of ~top-100-per-year queries.
- **Wikipedia article titles, ranked by pageviews.** The top-100k
  English Wikipedia articles by pageview (via the Wikimedia
  Pageviews API) double as a high-recall "things people search
  for" list. This is the trick [TimesFM](../papers/timesfm.md)
  implicitly uses by pairing Wikipedia Pageviews with Google Trends —
  the two corpora cover overlapping topical space. See
  [wikipedia-pageviews-leakage.md](wikipedia-pageviews-leakage.md)
  for how Wikipedia pageview data reaches pretraining corpora.
- **Search-log head lists.** Kaggle's 2017 "Web Traffic Time Series
  Forecasting" competition dataset is a 145,063-article Wikipedia
  snapshot that already encodes "what was searched in 2015-2017"
  (see [wikipedia-pageviews-leakage.md § 2](wikipedia-pageviews-leakage.md)).
  Useful as a coverage benchmark to check your seed list does not
  miss obvious categories.
- **Domain seeds.** If you want good coverage on specific domains
  (health, finance, sports), seed with domain glossaries: ICD-10
  disease names for health, SP500 tickers + company names for
  finance, WordNet noun synsets for broad English coverage.
- **Programmatic expansion with `related_queries` / `suggestions`.**
  Once you have an initial seed of ~1k, use
  `pytrends.related_queries` and the autocomplete / `suggestions`
  endpoint (see
  [../datasets-benchmarks/google-trends-data.md § 3](../datasets-benchmarks/google-trends-data.md))
  to discover terms Google itself considers related. Two rounds of
  expansion usually saturate coverage per domain.
- **LLM-based expansion (newer, 2024+ literature).** A recent line
  of work uses an LLM to generate candidate queries from a seed
  topic or a document — e.g., Meta's RTTP
  ([rttp_2601.17567.pdf](../../papers/rttp_2601.17567.pdf)) "Real-Time
  Trend Prediction via Continually-Aligned LLM Query Generation".
  For a pretraining corpus this is useful for high-recall expansion
  but introduces LLM-specific biases; treat LLM outputs as candidates
  to filter through the popularity filter in §3b rather than as
  finished lists.

### 3b. Popularity filter

TimesFM picks ~22k "head queries" because **below that point Google
Trends returns > 50% sparse (mostly-zero) series** (TimesFM §2
verbatim). The "head" cut-off is empirical, not magic:

1. Estimate each candidate query's absolute popularity using G-TAB's
   calibrated scale (§5).
2. Set a sparsity-rate threshold (e.g., a series with > 50% zeros
   at daily granularity over its time range is dropped).
3. Keep queries that clear the threshold at the granularities you
   care about — hourly is strictest (most sparse), monthly is
   loosest.

A practical target is **25k-50k queries** after filtering, to give
yourself margin: some queries will drop during later quality audits.

### 3c. Avoiding bias traps

- **English dominance.** Google Trends data is multi-language, but
  TimesFM used English-centric head queries. Decide whether to
  match that bias or extend to multilingual coverage (which roughly
  multiplies API cost by the language count).
- **Temporal bias.** Queries popular only during one news event
  (2020 "coronavirus") contribute a short signal. Mix in queries
  with stable long-run interest to balance the pretraining
  distribution.

### 3d. Published keyword-selection methodology

The nowcasting-with-Google-Trends literature has been working on
this exact problem since 2011. Key references:

- **Ross (2013), "Nowcasting with Google Trends: a keyword
  selection method"** — local PDF
  [ross-backward-induction_2013.pdf](../../papers/ross-backward-induction_2013.pdf).
  *Fraser of Allander Institute Economic Commentary*, 37(2).
  Introduces the **backward induction method**: start from Google
  Trends verticals as coarse seeds, test each candidate keyword's
  predictive power against a target series in a simple
  autoregressive model, iteratively prune keywords whose p-values
  fail a significance threshold. Retains only statistically
  significant contributors. Simple, interpretable, reproducible —
  and it anticipates most modern sparsity-plus-variance filters.
- **Choi & Varian (2011/2012), "Predicting the Present with Google
  Trends"** — local PDF
  [choi-varian_ptp.pdf](../../papers/choi-varian_ptp.pdf). The
  foundational paper that established verticals-as-seeds as the
  default entry point for GT-based nowcasting.
- **Ferrara & Simoni (2019/2020), "When are Google data useful to
  nowcast GDP? An approach via pre-selection and shrinkage"** —
  local PDF
  [ferrara-simoni_2007.00273.pdf](../../papers/ferrara-simoni_2007.00273.pdf)
  (arXiv:2007.00273). Uses pre-screening (targeting methods) plus
  shrunk bridge regressions to select a useful Google-Trends subset
  from a large candidate pool.
- **Scott & Varian (2013/2014), "Predicting the Present with
  Bayesian Structural Time Series"** — local PDF
  [scott-varian_2014_bsts.pdf](../../papers/scott-varian_2014_bsts.pdf).
  Introduces **spike-and-slab** priors for automated keyword
  selection inside a BSTS nowcasting model — the Bayesian
  formalization of "pick the ~20 most predictive keywords from a
  larger candidate pool".
- **Kohns & Bhattacharjee (2022), "Nowcasting Growth using Google
  Trends Data: A Bayesian Structural Time Series Model"** — local
  PDF [kohns-nowcast_2011.00938.pdf](../../papers/kohns-nowcast_2011.00938.pdf)
  (arXiv:2011.00938). Extends the Scott-Varian BSTS framework with
  **horseshoe** priors for heavier-tailed sparsity. Sensitivity
  analysis shows keyword-selection protocol matters roughly as much
  as model architecture for nowcasting accuracy.
- **Medeiros & Pires (2021), "The Proper Use of Google Trends in
  Forecasting Models"** — local PDF
  [gtrends-proper_2104.03065.pdf](../../papers/gtrends-proper_2104.03065.pdf)
  (arXiv:2104.03065). Warns that naïve keyword selection on a
  single GT sample can produce "arbitrary conclusions by chance"
  because of Bernoulli resampling noise; their §3 prescribes
  repeated-sample averaging *before* any keyword-selection
  statistics are computed.
- **Meta RTTP (Hui et al., 2024+), "Real-Time Trend Prediction via
  Continually-Aligned LLM Query Generation"** — local PDF
  [rttp_2601.17567.pdf](../../papers/rttp_2601.17567.pdf)
  (arXiv:2601.17567). LLM-driven query generation from social-media
  posts; scores candidates by engagement strength + creator
  authority. The closest modern analog to Google Correlate for
  discovering candidate queries that anticipate a target signal.
- **Google Correlate (deprecated 2019-12-15).** Historically the
  canonical tool: you fed in a target series, Google returned
  queries whose time-series correlated with it. Shut down for low
  usage. No direct replacement exists today; the closest
  substitutes are Google Trends autocomplete, topical seed
  expansion, and LLM-based query generation (see Meta RTTP above).
- **Topic-model–based coverage.** Seeded topic models (e.g.,
  `keyATM` in R) and embedding-based expansions (BERTopic) are
  standard NLP tools for ensuring a candidate pool has broad
  topical coverage before it goes through the popularity filter.
  Used less often than the econometric methods above, but
  complementary.

Pragmatic synthesis for a TimesFM-scale rebuild: **combine
vertical-walk (Ross 2013) + Wikipedia top-pageview list + LLM-based
expansion** to build a candidate pool of ~200k queries, then apply
G-TAB popularity calibration + sparsity filter (§3b) to prune down
to 25-50k head queries. This three-axis strategy covers more of the
topical space than any single published method.

## 4. Plan the time range and granularities

TimesFM's frequency mix:

- **Hourly:** Jan 2018 – Dec 2019 (1 year 11 months).
- **Daily / weekly / monthly:** Jan 2007 – Dec 2021.

For a 2026 rebuild, extend to present-day. Recommended targets:

- **Hourly:** last 3-5 years (e.g., Jan 2021 – today). Hourly data
  only exists for roughly this recent window in Google Trends
  anyway — it's not available for 2010-era requests.
- **Daily:** Jan 2007 – today. Requires sub-window stitching
  because Google Trends returns daily data only for requests ≤ 269
  days long; longer daily requests get auto-downgraded to weekly.
- **Weekly:** Jan 2007 – today. Google returns weekly for requests
  between 270 days and ~5 years.
- **Monthly:** Jan 2007 – today. For requests > 5 years, Google
  returns monthly.

The stitching problem (§7) is most severe at daily frequency,
moderate at weekly, minimal at monthly. You will run hundreds of
thousands of API requests; plan budget accordingly.

## 5. Build the G-TAB anchor bank

Full explanation in
[../datasets-benchmarks/google-trends-data.md § 4.2](../datasets-benchmarks/google-trends-data.md).
Recap: an anchor bank is a pre-calibrated set of queries spanning
the full popularity spectrum, so that any new query can be placed
on a common numeric scale by comparing it against the nearest
anchor. Without this step, a popular query and an unpopular query
cannot be put on the same scale, because Google Trends normalizes
within each request.

### 5a. Construction

Use `epfl-dlab/GoogleTrendsAnchorBank` directly rather than
reimplementing. The default anchor set it ships covers roughly 8
orders of magnitude of popularity and is adequate for English
queries. For non-English or domain-specific corpora, construct a
custom anchor set by:

1. Drawing a stratified sample of candidate queries at every
   popularity band (use a coarse pytrends pass to estimate).
2. Chaining them pairwise in 2-of-5-per-request calls until every
   anchor is connected transitively to a master reference.
3. Averaging over ≥ 10 independent samples per link to suppress
   Bernoulli noise.

### 5b. Per-query calibration

At inference time, G-TAB binary-searches the anchor bank for each
new query. Expect 3–6 API requests per calibrated series. For a
50k-query × 3-granularity corpus that is ~450k to 900k API calls —
not counting repeated sampling (§6) which multiplies it further.
Plan rate-limit strategy around this number.

## 6. Sampling protocol to suppress noise

Medeiros-Pires 2021 (`gtrends-proper_2104.03065.pdf`) demonstrates
that **each Google Trends request is a Bernoulli draw of the
underlying query logs**, so the returned series varies each time.
Two concrete observations from their paper:

- Variance can be substantial for mid-popularity queries.
- Running the *same* request twice can yield different conclusions
  about whether series A leads series B.

Mitigation: for each (query, granularity, sub-window) triple,
request N independent samples (separated by rate-limit-respecting
sleeps) and **average them pointwise**. `trendecon` uses N ≈ 10 for
its economic-sentiment index. A budget-aware default for a TS-FM
corpus is N = 5-10 for head queries, N = 20 for ambiguously-sparse
mid-popularity queries where the noise matters most.

This multiplies API cost 5–20×. Do it anyway — without averaging,
the corpus inherits Bernoulli noise that no downstream TS-FM can
compensate for.

## 7. Stitch temporal slices

Google Trends returns at most ~269 days of daily data per request.
To cover 2007 – today (~19 years) you need ~26 non-overlapping
sub-windows at daily granularity. Simple concatenation fails
because each sub-window is **re-normalized to its own peak = 100**,
so adjacent sub-windows are on different scales. Two fixes:

### 7a. Overlap-based stitching (simple, lossy)

Request sub-windows with an overlap of 10–30 days. For each adjacent
pair, compute the scale ratio `r = mean(overlap_window_A) /
mean(overlap_window_B)` and rescale B by `r` so it is on A's scale.
Walk chronologically from the most recent window backward, applying
cumulative ratios. Works when all sub-windows have non-trivial
signal in the overlap region; fails at query lifecycles (e.g., a
term that was dormant in 2008-2010 and popular after 2020 — the
overlap will be all-zeros and the ratio is undefined).

### 7b. Anchor-based stitching (robust, what G-TAB effectively does)

Use a *common anchor query* that has consistent popularity across
the entire 2007 – today range (classics: "weather", "news",
"google"). Request `[Q, common_anchor]` for each sub-window. The
anchor's peak within each window gives you an absolute scale to
divide by. This works even for dormant-then-popular queries because
the anchor provides a non-zero reference in every window.

**Recommended:** use 7b with an anchor drawn from the G-TAB bank,
so stitching and cross-query calibration share the same scale.

### 7c. Known edge cases

- **Post-2020 normalization change.** Google changed the relative
  scoring subtly around 2020 for some geographies; the pre/post
  boundary can produce a discontinuity even with proper stitching.
  Include a smoothing flag for this at 2020-01.
- **Hourly data boundary.** Hourly GT data is only available for
  requests within the last ~5 years. If you stitch hourly across
  years, you quickly hit the "request outside the allowed window"
  error. Constrain hourly to its natural recent window.

## 8. Reconcile across frequencies (trendecon-style)

Once you have (per query) daily, weekly, monthly, hourly series
that cover the full time range, one last step remains: make sure
the three long-horizon series are **mutually consistent**. If the
daily series says `sum_over_week ≈ 600` for a given week but the
weekly series says `400`, one of them is wrong — usually because
each was built by stitching a different set of sub-windows with
different scale factors.

`trendecon`'s method (Eichenauer et al. 2022) is the canonical
answer:

1. Treat the **monthly** series as the most reliable long-horizon
   anchor (it was computed in one or two requests for most years —
   little stitching error).
2. Apply **Chow-Lin disaggregation** to construct a weekly series
   that (a) matches the structure of the *observed* weekly data but
   (b) aggregates exactly to the monthly series.
3. Apply Chow-Lin *again* to disaggregate the reconciled weekly
   into a daily series that matches observed daily variation but
   aggregates exactly to weekly.

Implementation: `trendecon::ts_gtrends_mwd()` does this end-to-end.
Reimplementing in Python is straightforward — Chow-Lin is a
~50-line linear-GLS method, widely available in `statsmodels` or
in `tempdisagg` packages.

The result is a **frequency-consistent** daily series that
preserves short-term shape while honoring weekly and monthly
aggregates. This is what sets a trendecon-calibrated corpus apart
from a naive concatenation.

## 9. Quality audits

Before releasing the corpus, run:

1. **Across-sample variance check.** For a random 1% subsample of
   (query, granularity) pairs, collect 20 independent samples and
   plot the per-timestep standard deviation. If the std at any
   timestep exceeds a threshold (e.g., 15% of the local mean), the
   averaging in §6 was under-powered for that query — flag it.
2. **Anchor consistency.** Pick 10-20 anchors whose absolute
   popularity is well-established (e.g., `"weather"` should show
   seasonal structure, `"Super Bowl"` should have an annual
   February spike). Check that your corpus's anchor series exhibit
   the known structure.
3. **Frequency aggregation check.** Recompute monthly from
   reconciled daily; compare with raw monthly request. After
   Chow-Lin the residual should be near-zero.
4. **Dataset-level comparison.** For queries that appear in the
   Google Trends Datastore (`googletrends.github.io/data`), compare
   your series to Google's published version. Shape must match;
   scale differences are expected because the Datastore uses
   Google's internal absolute volumes.
5. **TS-FM-facing spec check.** Each series must be (a) padded to
   the full time range with missing-indicator flags, (b) in
   pandas-compatible format, (c) dated in UTC. These are the
   conventions LOTSA and GiftEvalPretrain use, and your corpus
   should match for tooling compatibility.

## 10. Storage, release, and leakage considerations

- **Format.** Sharded Parquet, one shard per ~500 queries, is the
  de facto TS-FM norm (matches LOTSA, Time-300B, GiftEvalPretrain).
  Include per-query metadata: query string, language, region,
  granularity, sample count N, G-TAB anchor(s) used, date range,
  sparsity rate.
- **Release channel.** HuggingFace Datasets. As of 2026-04 there
  is no Google-Trends TS corpus on HF at all (see
  [../datasets-benchmarks/google-trends-data.md § 6](../datasets-benchmarks/google-trends-data.md));
  publishing one fills a real gap.
- **License.** CC-BY-4.0 is the common choice; Google Trends terms
  allow redistribution of aggregated derivative data with
  attribution.
- **Leakage against the canonical TS-FM benchmarks.** Google Trends
  data is **not** in GIFT-Eval test, Monash, Chronos Benchmark II,
  or fev-bench (see
  [../datasets-benchmarks/leakage-map.md](../datasets-benchmarks/leakage-map.md)).
  A GT corpus built with this recipe can therefore be used for
  clean zero-shot claims against all four benchmarks.
- **Self-overlap for downstream GT tasks.** If your eventual
  evaluation includes a Google-Trends-based nowcasting task
  (economic index, flu forecasting), apply chronological splits
  inside that task so you don't leak forward.

## 11. Budget estimate

For 50k queries × 4 granularities × ~26 sub-windows (daily is the
dominant cost) × N = 10 samples × 5 G-TAB requests per calibration,
the total request count is in the low millions. With `pytrends`'s
~1,400-requests-per-4-hour-window rate-limit, that is 6–12 months
of continuous unattended collection on a single IP. Options:

- Accept the wall-clock and run it patiently.
- Use a commercial proxy-rotation service (see
  [../datasets-benchmarks/google-trends-data.md § 3](../datasets-benchmarks/google-trends-data.md)).
- Distribute across multiple IPs and coordinate rate-limiting.
- Restrict to 25k queries × 3 granularities for a v0 corpus, then
  expand later.

The Google-Trends team itself has not clarified the exact rate
limit, so expect to tune the retry/backoff schedule empirically.

## 12. What you cannot reproduce exactly

- **TimesFM's specific 22k query list** is not published (TimesFM
  §7 Reproducibility). Your query set will differ, and your corpus
  is therefore a TimesFM-style *sibling*, not a clone.
- **Google's internal differential-privacy noise** is applied
  server-side before returning data (TimesFM §8). It cannot be
  undone.
- **Absolute search volumes.** Google Trends never returns them.
  Your corpus is calibrated-relative, not absolute.

Document these limits in the dataset card so downstream users do
not mistake your corpus for a ground-truth measurement.

## Related wiki pages

- [../datasets-benchmarks/google-trends-data.md](../datasets-benchmarks/google-trends-data.md) — background on the API, calibration tools, and the published academic methodology this recipe draws on.
- [rebuilding-timebench.md](rebuilding-timebench.md) — the companion recipe for a broader TS pretraining corpus (LOTSA + Time-300B + synthetic + Chronos).
- [../datasets-benchmarks/scrub-tools.md](../datasets-benchmarks/scrub-tools.md) — the leakage-removal tooling catalog; scrub before release if your corpus overlaps a benchmark you care about.
- [../datasets-benchmarks/leakage-map.md](../datasets-benchmarks/leakage-map.md) — the corpus-versus-benchmark leakage cross-reference (spoiler: Google Trends is not in any canonical TS-FM benchmark).
- [../papers/timesfm.md](../papers/timesfm.md) — the original TimesFM paper this recipe is calibrated against.
- [../benchmarks/wikipedia-pageviews-leakage.md](../benchmarks/wikipedia-pageviews-leakage.md) — the sibling audit for Wikipedia pageviews, the other half of the TimesFM data recipe.
- [../concepts/synthetic-data-augmentation.md](../concepts/synthetic-data-augmentation.md) — synthetic alternatives if the API-collection cost is prohibitive.
