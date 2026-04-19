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

The canonical caveat every paper repeats: Google Trends does **not**
return absolute search volumes. The platform returns a **normalized
index** in `[0, 100]`, rounded to **integer precision**, where the
normalization is:

1. Divide each data point by the total searches for the chosen
   location + time range.
2. Scale the resulting numbers from 0 to 100 based on the topic's
   proportion of all searches.

Two consequences:

- **Rounding destroys information for unpopular queries.** West 2020
  (G-TAB) is explicit: "entirely uninformative, all-zero time series
  may be returned for unpopular queries when requested together with
  more popular queries" (`gtab_2007.13861.pdf`, Abstract).
- **Each request is a fresh sample and the values vary.** Medeiros &
  Pires 2021 is explicit: "each sample of Google search data is
  different from the other, even if you set the same search term,
  data and location" — so naive use leads to conclusions "by chance"
  (`gtrends-proper_2104.03065.pdf`, Abstract).

Any reliable use of Google Trends requires either repeated sampling
with averaging, or calibration against an anchor query (G-TAB).

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
- **Paper.** Robert West, "Calibration of Google Trends Time Series",
  CIKM 2020 (arXiv:2007.13861). PDF at
  [../../papers/gtab_2007.13861.pdf](../../papers/gtab_2007.13861.pdf).
- **What it does.** Calibrates arbitrarily many queries onto a
  common scale *without rounding-loss collapse*. The method builds
  an offline "anchor bank" — a set of queries spanning the full
  popularity spectrum, all chained back to a common reference query.
  At query time, any new query is calibrated against the anchor bank
  via binary search (few requests).
- **Why it matters.** Solves the West 2020 all-zero-series problem
  for unpopular queries and the "5-queries-per-request" limit in a
  principled way.
- **Use this for.** Any Google-Trends-based pretraining corpus that
  ranges over queries of wildly different popularity — which is
  every realistic TS-FM use.

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
  replication code. Examples: Choi & Varian 2011 "Predicting the
  Present", Medeiros & Pires 2021 (`gtrends-proper_2104.03065`),
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

The two canonical methodology papers are downloaded into this wiki's
paper folder for reference:

- [gtab_2007.13861.pdf](../../papers/gtab_2007.13861.pdf) — West,
  "Calibration of Google Trends Time Series" (CIKM 2020).
- [gtrends-proper_2104.03065.pdf](../../papers/gtrends-proper_2104.03065.pdf) —
  Medeiros & Pires, "The Proper Use of Google Trends in Forecasting
  Models" (arXiv:2104.03065).

Additional reading not fetched as PDF leaves:

- Eichenauer, Indergand, Martínez, Sax, "Obtaining consistent time
  series from Google Trends", *Economic Inquiry* 60(2), 2022 —
  `trendecon`'s canonical paper. A related earlier working paper by
  the same team, "Constructing Daily Economic Sentiment Indices
  Based on Google Trends", circulated as a KOF ETH Zürich working
  paper in 2020 (identifier varies by index; cite the *Economic
  Inquiry* version for stability).
- Choi & Varian, "Predicting the Present with Google Trends",
  *Economic Record*, 2012 — foundational nowcasting paper.
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
