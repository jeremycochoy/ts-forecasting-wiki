# Wikipedia Pageviews and GIFT-Eval: What to Scrub and Why

Wikimedia pageview data is one of the largest freely available sources
of real-world time series. A natural question before folding it into a
pretraining corpus is whether it leaks with the canonical TS-FM
benchmarks. This page is the concrete answer for
[GIFT-Eval](../datasets-benchmarks/gift-eval.md), verified against the
benchmark's own paper (Aksu et al., "GIFT-Eval", arXiv:2410.10393).

## 1. Headline finding

**The GIFT-Eval test (evaluation) split contains zero Wikipedia-derived
series.** All 23 test datasets are listed in Appendix D / Table 13 of
the paper: the Web/CloudOps domain is covered by `BizITObs` (three
subsets) and `Bitbrains` (two subsets), all CloudOps telemetry from
AutoMixer (Palaskar et al., 2024) and the Grid Workloads Archive
(Shen et al., 2015). No `Wiki-Rolling`, no `Kaggle Web Traffic`, no
Wikipedia pageviews in the evaluation.

This means: **if your evaluation is GIFT-Eval test, training on raw
Wikimedia pageviews does not produce direct test-set leakage.** It is
precisely why Salesforce's `Salesforce/GiftEvalPretrain` dataset can
include Wikipedia data while still being called "non-leaking".

## 2. The three Wikipedia-derived datasets inside GIFT-Eval Pretrain

GIFT-Eval Pretrain (the 88-dataset, 4.5M-series, 230B-point corpus
released as `Salesforce/GiftEvalPretrain`) lists its full contents in
Appendix E / Table 14 of the paper. Three entries are Wikipedia-derived:

| Dataset name in Table 14 | Upstream source | Freq | # Series | # Obs |
|---|---|---|---|---|
| `Wiki-Rolling` | GluonTS (Alexandrov et al., 2020) | D | 47,675 | 40,619,100 |
| `Kaggle Web Traffic Weekly` | Monash (Godahewa et al., 2021) | W | 145,063 | 16,537,182 |
| `Extended Web Traffic` | Monash (Godahewa et al., 2021) | D | 145,063 | 370,926,091 |

Combined: ~338k distinct article series (the two Monash entries almost
certainly share the same underlying 145,063-article list from the
Kaggle 2017 Web Traffic Time Series Forecasting competition, at two
different aggregations — daily + weekly). Together they contribute
~428M observations to the pretraining corpus, ~0.2% of its 230B-point
total.

## 2b. Wikipedia data across the other major TS-FM corpora

For context, Wikipedia pageview data is already present in most other
large TS-FM corpora as well — verified against the papers' own tables:

| Corpus | Wikipedia-derived entries |
|---|---|
| [LOTSA](../datasets-benchmarks/lotsa.md) (Moirai) | `Kaggle Web Traffic Weekly` (133,388 series), `Extended Web Traffic` (161,890 series), `Wiki-Rolling` (47,675 series), `Wiki Daily (100k)` (100,001 series; sourced from Chronos) — Moirai Table 14 |
| [Time-300B](../datasets-benchmarks/time-300b.md) (Time-MoE) | Same four datasets as LOTSA — Time-MoE Table 10 |
| [Chronos](../papers/chronos.md) v1 pretraining-only | `Wiki Daily (100k)` (100,001 series × 2,741 obs/series) — Chronos Table 3 / Appendix B |
| [TimesFM](../papers/timesfm.md) v1 pretraining | "all Wikimedia pageview data" from Jan 2012 – Nov 2023, ~300B points before dataset-mixing; exact page list not released — TimesFM §2 |

Consequence: Wikipedia pageview data is already in virtually every
large public TS-FM pretraining corpus. Folding raw Wikimedia on top
is primarily a *deduplication* concern (covered in section 3b), not a
fresh leakage vector against GIFT-Eval test.

## 3. Practical implications

### 3a. If you use `Salesforce/GiftEvalPretrain` as-is

Do nothing. The corpus is already leakage-free against the GIFT-Eval
test split by construction. The Wikipedia data it contains is a
*feature*, not a bug — it is one of the domain channels the pretraining
corpus exposes.

### 3b. If you add your own Wikimedia pageview dump on top of `GiftEvalPretrain`

The risk is not leakage (the test split has no Wiki data) but
**double-counting**: your dump will overlap with the ~145k articles
already represented in Wiki-Rolling + Kaggle Web Traffic. Without
deduplication you inflate the effective weight of those specific
articles in your training distribution.

Dedup procedure (see section 4 for how to extract the article lists):

1. Extract the Wikipedia article titles from each of the three
   datasets listed in section 2.
2. Union the three lists; normalize (URL-decode, lowercase, strip
   language/access/agent suffixes).
3. Drop every row in your raw Wikimedia dump whose article title is in
   the union set.
4. (Optional) drop the 2015-07 through 2017-09 window for all articles
   even outside the union, since that is the Kaggle snapshot window.

### 3c. If your evaluation includes anything beyond GIFT-Eval test

This is the case where leakage actually matters:

- [Monash Archive](../datasets-benchmarks/monash-archive.md) includes
  `kaggle_web_traffic_dataset` as a first-class entry. If you evaluate
  on Monash web-traffic tasks, Wikimedia data must be scrubbed.
- Chronos Benchmark II overlaps with Monash at several entries; verify
  the specific subset you target.
- fev-bench is audited per model (`evaluation/protocols.md:138-140`),
  with per-baseline leakage percentages published.

For these, the full Timer-S1 scrub
([rebuilding-timebench.md § 6](rebuilding-timebench.md)) applies. The
series-level article list from the three GIFT-Eval Pretrain datasets
is the starting exclusion set; extend it with Monash's own web-traffic
enumeration and any additional Wikipedia entries in the benchmark you
use.

## 4. Extracting the article list from each source

Each Wikipedia-derived dataset ships the article identifiers as part
of its schema. Mechanical extraction:

### 4a. `Wiki-Rolling` (GluonTS)

Distributed through the GluonTS dataset loader. Series identifiers are
URL-encoded Wikipedia article titles. 47,675 entries.

```python
from gluonts.dataset.repository import get_dataset
ds = get_dataset("wiki-rolling_nips", regenerate=False)
titles = {e["item_id"] for e in ds.train}
```

(Verify the exact dataset handle against the current GluonTS release;
the historical identifier is `wiki-rolling_nips`.)

### 4b. `Kaggle Web Traffic Weekly` and `Extended Web Traffic` (Monash)

Both are Monash redistributions of the Kaggle 2017 Web Traffic Time
Series Forecasting competition, which covered **145,063 Wikipedia
articles** with daily series from **2015-07-01 to 2017-09-10**. The
column identifiers follow the Kaggle convention
`<url_encoded_title>_<lang>.wikipedia.org_<access>_<agent>`, where
`access ∈ {desktop, mobile-web, all-access}` and
`agent ∈ {all-agents, spider}`.

Extraction from the Monash TSF file:

```python
# Using the `tsf_reader` utility or pandas:
import pandas as pd
df = pd.read_csv("kaggle_web_traffic_dataset.csv")  # or the equivalent TSF
identifiers = df["series_name"].unique()           # or the series-id column
```

Parse each identifier into `(title, lang, access, agent)` by splitting
on the underscore convention. Keep `title` as the scrub key.

Authoritative raw source: the Kaggle competition files
(`train_1.csv`, `train_2.csv`) list the articles in a `Page` column.
The Zenodo redistribution (`zenodo.org/records/4656075` for the
daily-no-missing version, `zenodo.org/records/7371038` for the
extended version) packages the same article set.

### 4c. Unioning the three lists

Expect ~145k unique articles after normalization — Wiki-Rolling is a
smaller subset of the same Kaggle article universe, so the union is
dominated by the Monash 145,063-article list.

## 5. Edge cases the scrub does not catch

- **Related-article correlation.** Excluding "Barack Obama" pageviews
  does not neutralize "Michelle Obama" pageviews; their dynamics
  covary. No published mitigation in the TS-FM literature.
- **Event-driven correlation.** Major news events produce correlated
  spikes across hundreds of otherwise-unrelated articles. Title-level
  scrubbing does not remove the event signal in the benchmark test
  window.
- **Cross-language correlation.** The Kaggle competition included
  multiple language editions (en, fr, de, ja, ru, es, zh). Decide
  whether to scrub titles across all languages or only the languages
  actually represented in the benchmark.
- **Article renames and redirects.** Articles renamed between 2017
  (Kaggle snapshot) and your scrape date appear under different titles
  in your dump. Expand the exclusion list via the MediaWiki API's
  redirect query for every title to catch renamed aliases.

## 6. Is there existing code for the scrub?

**For the GIFT-Eval-clean pretraining case, yes — but it is the
dataset, not a filter.** Salesforce's `Salesforce/GiftEvalPretrain` on
HuggingFace *is* the "existing code": they already assembled the
88-dataset, Wikipedia-inclusive, GIFT-Eval-test-disjoint corpus and
published it. The `uni2ts` library (github.com/SalesforceAIResearch/uni2ts)
loads it directly.

**For a raw-Wikimedia-scrub-against-GIFT-Eval case, no dedicated
public tool is documented in this wiki.** Timer-S1 says it performs a
GIFT-Eval leakage scrub during TimeBench curation
(`papers/timer-s1.md:13`) but does not release the code
(`papers/timer-s1.md:35`). The procedure in section 4 is the
reconstructable recipe.

If you need a runnable implementation, the pragmatic path is:

1. Download `Wiki-Rolling`, `Kaggle Web Traffic Weekly`, and
   `Extended Web Traffic` from their sources (GluonTS loader, Monash
   archive, or GiftEvalPretrain directly).
2. Extract the identifier columns as in section 4.
3. Apply the union as a filter on your Wikimedia dump.

That is ~50 lines of Python and does not exist as a packaged utility
in any TS-FM repository I am aware of — a small, useful open-source
contribution.

## 7. Related wiki pages

- [rebuilding-timebench.md](rebuilding-timebench.md) — companion recipe where this scrub applies.
- [../datasets-benchmarks/gift-eval.md](../datasets-benchmarks/gift-eval.md) — the benchmark this page audits.
- [../datasets-benchmarks/monash-archive.md](../datasets-benchmarks/monash-archive.md) — the sibling benchmark with first-class Wikipedia entries.
- [../datasets-benchmarks/lotsa.md](../datasets-benchmarks/lotsa.md) — where `GIFT-Eval Pretrain` fits in the LOTSA / Moirai-2 lineage.
- [../evaluation/protocols.md](../evaluation/protocols.md) — the wider leakage-cases catalog.
- [methodology-caveats.md](methodology-caveats.md) — section 1 on pretraining-vs-evaluation overlap.
- [../papers/timesfm.md](../papers/timesfm.md) — the canonical user of Wikipedia Pageviews in a TS-FM corpus.
- [../papers/timer-s1.md](../papers/timer-s1.md) — the paper that documents (but does not release) a GIFT-Eval scrub procedure.
