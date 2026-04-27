# Google Trends as a Time-Series Data Source

Google Trends exposes search-interest data for virtually any query,
any region, and several time granularities going back to 2004. This
page catalogs every practical way to obtain Google Trends time
series for TS-FM pretraining or nowcasting, along with the
well-documented pitfalls.

This is a significant data source for TS-FM work because
[TimesFM](../papers/timesfm.md) pretrained on ~22k Google Trends
head queries across four granularities (~0.5B time points), making
Google Trends one of the two biggest public real-world contributions
to a released TS-FM corpus (alongside Wikimedia pageviews; see
[../benchmarks/wikipedia-pageviews-leakage.md](../benchmarks/wikipedia-pageviews-leakage.md)).

## 1. What Google Trends actually returns

Google Trends does **not** return absolute search volumes. For any
request, Google picks the largest value in the requested
(query, time, region) window and calls that 100, then scales every
other value as a percentage of that maximum and **rounds to integer
precision** in `[0, 100]`. Two additional facts matter:

- **One request accepts at most 5 queries.** All 5 are normalized
  together: their values are relative to the peak value across the
  entire set of 5 in that time window.
- **Each request is a fresh Bernoulli sample of the underlying
  query logs.** Ask the same question again with the same inputs
  and the returned numbers will differ, sometimes substantially
  ([Medeiros & Pires 2021](../papers/gtrends-proper.md), Abstract).

The integer-rounding + "peak is 100" rules interact badly when one
request mixes popular and unpopular queries. Concrete example: ask
for `["coronavirus", "my-niche-hobby"]` over 2020-2022. Coronavirus
peaks at 100 during the March 2020 spike. Every `my-niche-hobby`
value, even if actually non-zero, is a small fraction of that
coronavirus peak — so once it gets rounded to the integer grid it
collapses to **all zeros**. You cannot recover the niche hobby's
real time-series from this request at all. West 2020 states the
consequence plainly: "entirely uninformative, all-zero time series
may be returned for unpopular queries when requested together with
more popular queries" ([West 2020](../papers/gtab.md), Abstract).

The two fixes for reliable use are: (a) **repeated-sampling with
averaging** to smooth out the Bernoulli noise (Medeiros-Pires
recommendation) and (b) **calibration via chained requests** so
that queries of very different popularity never end up in the same
request — this is what G-TAB automates (§4 below).

## 2. Official Google channels

### Google Trends Datastore

- **URL.** `https://googletrends.github.io/data/`.
- **What it is.** A curated repository of downloadable datasets
  prepared by Google's Trends Data Team. Includes election-related
  searches, annual top-chart CSVs, and other Google-selected
  snapshots. **Not** a programmatic bulk-access tool.
- **License.** Public use with attribution.
- **Use this for.** Pre-processed topical datasets that Google has
  already cleaned. Limited in scope; do not expect comprehensive
  bulk access here.

### Google Trends on BigQuery

- **Dataset.** `bigquery-public-data:google_trends` and
  `bigquery-public-data:international_google_trends` (launched March
  2022).
- **What it is.** Top-25-overall and top-25-rising search queries for
  the past 30 days in the US and ~50 international regions, updated
  daily.
- **Scale.** **Only top-25**. Not a full query-space dataset. Access
  via SQL at no additional cost within BigQuery's free tier (1 TB /
  month queries, 10 GB storage).
- **Use this for.** Quick trending-topic analysis. Not suitable for
  pretraining-scale TS-FM work (the top-25 cap rules out the
  long-tail millions of queries you would need).

### Official Google Trends web UI

- **URL.** `https://trends.google.com`.
- **Manual CSV download** per-query is available.
- **Use this for.** One-off exploratory plots; not at scale.

## 3. Unofficial Python APIs

### `pytrends`

- **Repo.** `https://github.com/GeneralMills/pytrends`.
- **What it does.** Unofficial scraping wrapper around the Google
  Trends web interface. Supported methods:
  - `interest_over_time` — historical indexed data for up to 5 terms.
  - `multirange_interest_over_time` — multiple date ranges in one
    call.
  - `get_historical_interest` — hourly-level, multi-request.
  - `interest_by_region` — country / region / metro / city
    breakdown.
  - `related_topics` / `related_queries` — discovery of associated
    terms.
  - `trending_searches`, `top_charts`, `suggestions`.
- **Rate limits.** Documented: ~1,400 sequential requests on 4-hour
  granularity hits the limit; recover with 60-second sleep between
  requests after rate-limiting. The exact threshold is not public.
- **Caveats.** Google may change aggregation level for very-popular
  or very-unpopular terms without warning; monthly Top Charts are
  broken; current-year data is often unavailable. The project is
  explicitly "not an official or supported API."
- **Use this for.** Small-to-mid-scale academic and research
  pipelines. Not for continuous production use.

### Other scraper implementations

Several open-source scrapers exist on GitHub with varying maintenance
activity. Independent evaluation would be required before picking any
of them for production use; the wiki does not endorse a specific
choice. Representative examples at time of writing:

- `dballinari/GoogleTrends-Scraper` — splits long time ranges into
  sub-periods and stitches results, a lighter-weight partial fix for
  the frequency-inconsistency problem more rigorously solved by
  `trendecon`.
- `clintonboys/trendy-scraper` — similar split-and-stitch approach.
- Commercial wrappers exist around paid SERP / proxy services
  (luminati, scrapeless, oxylabs). They trade cost for rate-limit
  headroom and are out of scope for research-only pipelines.

## 4. The academic calibration tools

Two research projects address the reliability problems identified
in section 1. Both are load-bearing if you want Google Trends data
for anything beyond exploration.

### `trendecon` (R package)

- **Repo.** `https://github.com/trendecon/trendecon`.
- **Paper.** Eichenauer, Indergand, Martínez, Sax, "Obtaining
  consistent time series from Google Trends", *Economic Inquiry*
  60(2), April 2022. Preprint arXiv variant: Eichenauer et al.,
  "Constructing Daily Economic Sentiment Indices Based on Google
  Trends", KOF Working Paper 20-484.
- **What it does.** Builds frequency-consistent daily series by
  querying Google at daily / weekly / monthly resolution, averaging
  across multiple independent samples to reduce Medeiros-style
  sampling noise, then combining the three frequencies with Chow-Lin
  (1971) disaggregation applied twice (monthly → weekly → daily).
- **Why it matters.** Raw Google Trends daily series do not
  preserve long-run trends correctly; `trendecon` demonstrates this
  and fixes it. The fix is the basis for the "Swiss Economic Daily
  Index" published weekly by KOF.
- **Use this for.** Nowcasting and long-horizon economic forecasting
  with Google Trends. The right tool when you need a series that is
  both daily-frequency and long-run-consistent.

### Google Trends Anchor Bank (G-TAB)

- **Repo.** `https://github.com/epfl-dlab/GoogleTrendsAnchorBank`.
- **Paper.** [Robert West, "Calibration of Google Trends Time Series"](../papers/gtab.md),
  CIKM 2020 (arXiv:2007.13861). Local PDF:
  [../../papers/gtab_2007.13861.pdf](../../papers/gtab_2007.13861.pdf).

**The problem G-TAB solves.** Directly from §1 of this page: if you
want to compare a popular query (say "football") with an unpopular
one (say "my local park's name") on a single scale, you cannot put
them in the same Google Trends request — the unpopular one gets
rounded to all-zeros. Asking them separately doesn't work either:
each request normalizes its own peak to 100, so the two resulting
series live on *different* scales that cannot be combined.

**How G-TAB solves it.** Think of queries as needing a "ruler". If
you cannot fit them all onto one ruler in a single request, **chain
requests transitively**:

- Request 1: `[A, B]` tells you A's peak relative to B's peak.
- Request 2: `[B, C]` tells you B's peak relative to C's peak.
- Combined: you now know A relative to C, even though A and C were
  never in the same request.

G-TAB industrializes this chaining in two phases:

1. **Offline anchor-bank construction (done once).** Pick a set of
   "anchor" queries spanning every popularity level — from ultra-
   popular ("weather") through medium-popular ("taxi") down to
   extremely-rare ("rare-scientific-term"). Chain them pairwise
   through many requests so every anchor's peak is known on a
   single common scale, relative to one master reference query.
   The result — the **anchor bank** — is a lookup table of
   absolute (uncalibrated-but-consistent) peak values for every
   anchor.
2. **Online calibration (for each new query at query time).**
   Given a new query `Q`, binary-search the anchor bank for the
   anchor closest to `Q`'s popularity. You only need a handful of
   Google Trends requests (each comparing `Q` against one or two
   anchors) to pin `Q` onto the common scale without rounding loss.

**What you get.** Any number of queries — of any popularity level —
placed onto one consistent scale, with the rounding-collapse
problem eliminated. This is what "calibration" means in the G-TAB
paper title.

**Use this for.** Any Google-Trends-based pretraining corpus that
covers queries of wildly different popularity — which is every
realistic TS-FM use. If you only ever query a narrow popularity
band, you can probably skip G-TAB. If you want a TimesFM-scale
corpus over heterogeneous topics, you essentially cannot avoid it.

## 5. The TimesFM Google Trends contribution

[TimesFM](../papers/timesfm.md) is the primary TS-FM that uses
Google Trends at scale. The paper's §2 "Data" describes:

- **Query selection.** "Around 22k head queries based on their
  search interest over 15 years from 2007 to 2022. Beyond these head
  queries the time-series become more than 50% sparse."
- **Granularities.** Hourly (2018-01 to 2019-12), daily / weekly /
  monthly (2007-01 to 2021-12).
- **Total contribution to corpus.** ~0.5 billion time points across
  the four granularities.
- **Release status.** The exact 22k-query list is **not released**.
  The TimesFM paper §7 "Reproducibility" says the aggregation is
  public but the specific slice is not.
- **Differential privacy.** TimesFM §8 "Data Privacy" (verbatim):
  *"Note that most of our data sources are publicly available and
  are aggregated i.e no individual user activity constitutes a
  time-point. Further the Google Trends data is differentially
  private."* This DP property applies only to Google Trends, not to
  Wikipedia Pageviews; the latter is aggregate pageview counts
  without a published DP mechanism.

If you want to reproduce the TimesFM Google Trends component
ingredient-for-ingredient: you need a seed list of 22k head queries
(the top-searched English queries over 2007–2022), `pytrends` or
`G-TAB` for calibrated retrieval, and enough API budget to run
~22k × 4 = 88k requests across four granularities.

## 6. Published Google Trends corpora (as of 2026-04)

No "LOTSA for Google Trends" exists. The field lacks a released,
calibrated, leakage-audited Google Trends *time-series* corpus.
Available alternatives:

- **TimesFM internal corpus** — described but not released.
- **Google Trends Datastore** (`googletrends.github.io/data/`) —
  small curated topical datasets only.
- **HuggingFace `ronantakizawa/trending-words-google`.** As of
  2026-04, this is the only Google-Trends-derived dataset located on
  the HuggingFace Hub via keyword search. Contents: top-ranked
  Google "Year in Search" terms from 2001–2024, 93 categories, 2,784
  entries. **Columns:**
  `word`, `year`, `tag` (category), `rank`. **Granularity is
  yearly**, and the dataset is a **ranked top-terms list**, not a
  search-interest time series. License: CC-BY-4.0. For TS-FM
  pretraining this dataset is **not useful** — the granularity and
  format do not match what a forecasting model consumes. It is
  useful for topical trend analysis and NLP work on trending-term
  vocabularies.
- **Per-paper microcorpora** — numerous economics / epidemiology
  papers publish small Google Trends datasets alongside their
  replication code. Examples: [Choi & Varian 2011](../papers/choi-varian.md) "Predicting the
  Present", [Medeiros & Pires 2021](../papers/gtrends-proper.md),
  nowcasting literature.

Building a new, releasable Google Trends corpus at TimesFM scale is
an **open contribution opportunity** the field would benefit from.
The technical prerequisites — G-TAB + `trendecon` — are solved; the
missing pieces are API budget and a committed maintainer. As of the
2026-04 snapshot of the HuggingFace Hub, no such dataset exists.

## 7. Leakage considerations for Google Trends data

- **Google Trends series do not appear in GIFT-Eval test.**
  GIFT-Eval's Web/CloudOps test subset is CloudOps telemetry only
  (BizITObs + Bitbrains; see [gift-eval.md](gift-eval.md)).
- **Nor in Monash or the Chronos corpus.** The Monash Archive's web
  entries are Wikipedia pageviews from the Kaggle 2017 competition,
  not Google Trends.
- **So Google Trends is essentially leakage-free against the
  canonical TS-FM benchmarks.** Folding Google Trends into a
  pretraining corpus does not produce test-split leakage against
  GIFT-Eval / Monash / Chronos II / fev-bench.
- **The caveat is self-overlap.** If you train on Google Trends AND
  evaluate on a downstream nowcasting task that also uses Google
  Trends, you have classic train-test leakage inside the target
  task. Fix: chronological splits per the `trendecon` paper
  methodology.

See [leakage-map.md](leakage-map.md) for the full corpus ×
benchmark matrix.

## 8. Reference papers (in `papers/`)

The methodology papers below are ingested into this wiki as paper
leaves under `wiki/papers/`; the leaves carry the per-paper detail
that this section uses inline:

- [GTAB](../papers/gtab.md) (West, CIKM 2020) — anchor-bank
  cross-batch calibration.
- [Medeiros-Pires "Proper Use of Google Trends"](../papers/gtrends-proper.md)
  (PUC-Rio, 2021) — per-request sampling-noise documentation and
  repeated-sampling stitching procedure.
- [Choi-Varian "Predicting the Present"](../papers/choi-varian.md)
  (Google, *Economic Record* 2012) — foundational nowcasting paper.
- [Scott-Varian BSTS](../papers/scott-varian.md) (Google, 2014) —
  Bayesian Structural Time Series + spike-and-slab on GT predictors.
- [Ferrara-Simoni](../papers/ferrara-simoni.md) (SKEMA / CREST, 2020)
  — preselection-and-shrinkage theoretical framework.
- [Kohns-Bhattacharjee](../papers/kohns-nowcast.md) (Heriot-Watt,
  2022) — COVID-era BSTS extension with mixed-frequency variable
  selection.
- [Ross "keyword selection"](../papers/ross-backward-induction.md)
  (Strathclyde, 2013) — game-theoretic backward-reasoning algorithm
  for selecting GT keywords.
- [RTTP](../papers/rttp.md) (Meta, 2026) — continually-aligned LLM
  query generation for cold-start trend detection.

Additional reading not yet ingested as paper leaves:

- Eichenauer, Indergand, Martínez, Sax, "Obtaining consistent time
  series from Google Trends", *Economic Inquiry* 60(2), 2022 —
  `trendecon`'s canonical paper. A related earlier working paper by
  the same team, "Constructing Daily Economic Sentiment Indices
  Based on Google Trends", circulated as a KOF ETH Zürich working
  paper in 2020 (identifier varies by index; cite the *Economic
  Inquiry* version for stability).
- Nowcasting-with-Google-Trends literature more broadly: applied
  macro / epidemiology papers that use GT as exogenous features.

## Related wiki pages

- [datasets-benchmarks.md](datasets-benchmarks.md) — data-section hub.
- [dataset-types.md](dataset-types.md) — Axis 7 "Release form" where
  API-only sources sit.
- [leakage-map.md](leakage-map.md) — corpus × benchmark matrix.
- [scrub-tools.md](scrub-tools.md) — code for leakage removal.
- [../benchmarks/wikipedia-pageviews-leakage.md](../benchmarks/wikipedia-pageviews-leakage.md) — Wikipedia's analogous data-source audit.
- [../papers/timesfm.md](../papers/timesfm.md) — the canonical TS-FM user of Google Trends.
- [../concepts/synthetic-data-augmentation.md](../concepts/synthetic-data-augmentation.md) — synthetic alternatives if you cannot access GT at scale.
