# Scrub Tools — Code for Leakage Removal and Corpus Curation

This page is a catalog of the public code you would actually run to
remove leakage from a TS-FM pretraining corpus, or to curate a raw
time-series dump into training-ready data. It is organized by
increasing level of bespoke effort required.

## 1. Ready-to-use: curated leakage-free corpora

These are **datasets**, not tools — the leakage removal has already
been done and the result is published. Pick one of these if your
goal is simply to train a model against a named evaluation benchmark
without re-doing the curation work.

### `Salesforce/GiftEvalPretrain`

- **What it is.** An 88-dataset, ~4.5M-series, ~230B-observation
  subset of LOTSA, engineered to be **disjoint from GIFT-Eval's
  test split** (GIFT-Eval Aksu et al., 2024, §3 and Appendix E /
  Table 14).
- **Leakage status.** Clean by construction against
  [GIFT-Eval](gift-eval.md) test.
- **Access.** `https://huggingface.co/datasets/Salesforce/GiftEvalPretrain`.
- **Loader.** `uni2ts` library (see below).
- **Use this if.** You want the simplest path to a legitimate
  zero-shot GIFT-Eval claim.

### `Salesforce/lotsa_data` (original LOTSA)

- **What it is.** The full ~27B-observation LOTSA corpus assembled
  for [MOIRAI](../papers/moirai.md).
- **Leakage status.** Contains Monash / M4 / ETT / Electricity
  datasets that overlap GIFT-Eval test. The GIFT-Eval paper §F.2
  formally evaluates "Moirai-Leakage" (the original MOIRAI
  checkpoint) to show the leakage effect.
- **Access.** `https://huggingface.co/datasets/Salesforce/lotsa_data`.
- **Use this if.** You intentionally want the original LOTSA for
  replication of MOIRAI / Moirai-MoE, and plan to evaluate on a
  different (non-GIFT-Eval) benchmark.

### `Maple728/TimeMoE` (Time-300B)

- **What it is.** The ~309B-observation Time-300B corpus released
  alongside [Time-MoE](../papers/time-moe.md) weights.
- **Leakage status.** Unaudited against GIFT-Eval / Chronos Benchmark
  II / fev-bench. Time-MoE predates GIFT-Eval.
- **Domain skew.** Nature = 90.50% of observations (Time-MoE Table 1).
- **Use this if.** You need more data than LOTSA offers and are
  willing to run your own leakage scrub. Start from
  [rebuilding-timebench.md](../benchmarks/rebuilding-timebench.md)
  section 3.

### `AutonLab/Timeseries-PILE` (Time Series Pile)

- **What it is.** [MOMENT](../papers/moment.md)'s multi-task
  pretraining corpus. Includes forecasting, classification, anomaly,
  and imputation tasks with documented disjoint splits for each.
- **Access.** `https://huggingface.co/datasets/AutonLab/Timeseries-PILE`.
- **Use this if.** Your model targets multi-task capability beyond
  forecasting.

### `autogluon/chronos_datasets`

- **What it is.** The 28-dataset public backbone of
  [Chronos](../papers/chronos.md) v1's pretraining corpus (13
  pretraining-only + 15 Benchmark I in-domain), redistributed on
  HuggingFace.
- **Leakage status.** The 15 Benchmark I datasets overlap GIFT-Eval
  test directly (M4, Electricity, ETT, KDD Cup 2018,
  Temperature-Rain, etc.).
- **Use this if.** You need the raw Chronos corpus as a reference
  point; apply aggressive dedup + scrub before training.

## 2. Training libraries with leakage-aware loaders

### `SalesforceAIResearch/uni2ts`

- **What it is.** The training and evaluation framework for MOIRAI,
  Moirai-MoE, and Moirai 2.0. Loads LOTSA, `GiftEvalPretrain`, and
  GIFT-Eval out of the box.
- **Leakage helpers.** The `GiftEvalPretrain` branch of the LOTSA
  loader applies the GIFT-Eval test-disjoint filter.
- **Repo.** `https://github.com/SalesforceAIResearch/uni2ts`.
- **Use this if.** You want a battle-tested data pipeline for
  LOTSA-family work.

### `SalesforceAIResearch/gift-eval`

- **What it is.** The evaluation harness for the GIFT-Eval benchmark.
  Handles dataset loading, windowing, metrics, and leaderboard
  submission.
- **Repo.** `https://github.com/SalesforceAIResearch/gift-eval`.
- **Use this if.** You are submitting to GIFT-Eval.

## 3. Documented but unreleased: the Timer-S1 pipeline

The [Timer-S1](../papers/timer-s1.md) paper documents the most
detailed public TS-FM data-curation pipeline — but the code is not
released. You can reconstruct it from the paper text. The steps are:

1. **Causal mean imputation** for missing values (no future-into-past
   leakage through the imputer). Timer-S1 §4.2 does not specify the
   window length.
2. **Outlier removal** via k-σ or IQR on shifting windows. Specific
   thresholds are not stated.
3. **ADF predictability filter.** Series that fail the Augmented
   Dickey-Fuller test are discarded or down-weighted. P-value
   threshold not stated.
4. **Eval-leakage scrub.** Explicit series-level dedup against the
   GIFT-Eval test splits *and* against Chronos Benchmark II. This
   is the step with the most practical leverage; section 4 below
   is the reconstructable recipe.
5. **Resampling augmentation.** Downsample + Fourier interpolation
   to expose the model to multiple temporal resolutions.
6. **Value-flipping augmentation.** Multiply by −1 to invert trends.

The Timer team uses ByteDance's internal `VeOmni` framework for the
training infrastructure (Timer-S1 §4.4). Neither the scrubber code
nor the final TimeBench mixture has been published as of 2026-04.

## 4. Reconstructable recipe — the scrub

None of the 1-2 libraries above provides a general "scrub arbitrary
TS data against benchmark X" utility. The following recipe is the
typical open-source reconstruction. It is ~50 lines of Python.

```python
# Pseudocode: scrub raw TS data against benchmark X
import pandas as pd, hashlib
from pathlib import Path

def load_benchmark_series(benchmark_path):
    """Returns a set of (series_id, start_date, end_date) for benchmark test."""
    # e.g. for GIFT-Eval, iterate through the HF dataset's test split:
    # each row is a series with a stable id column.
    ...

def series_signature(values, k=20):
    """First-k + last-k + length fingerprint for fuzzy matching."""
    h = hashlib.sha256()
    h.update(str(len(values)).encode())
    for v in values[:k]: h.update(f"{v:.6f}".encode())
    for v in values[-k:]: h.update(f"{v:.6f}".encode())
    return h.hexdigest()

# 1. Build exclusion index from benchmark
bench_ids = {row.series_id for row in load_benchmark_series("gift-eval/test")}
bench_hashes = {series_signature(row.values) for row in load_benchmark_series("gift-eval/test")}

# 2. Iterate your raw corpus
for series in iter_raw_corpus("my_wikimedia_dump.parquet"):
    if series.id in bench_ids: continue        # exact id match
    if series_signature(series.values) in bench_hashes: continue  # fuzzy match
    write_out(series)
```

For Wikipedia pageview data specifically, the title-based scrub is
documented in
[../benchmarks/wikipedia-pageviews-leakage.md](../benchmarks/wikipedia-pageviews-leakage.md)
section 4. For general Monash/M4/ETT scrub, iterate the Monash TSF
metadata and exclude by dataset name + series-id.

## 5. External libraries worth knowing

- **`pytrends`** (unofficial Python Google Trends API). Rate-limited;
  aggregates vary over time. See
  [google-trends-data.md](google-trends-data.md).
- **`trendecon`** (R package). Consistent daily-weekly-monthly Google
  Trends series via Chow-Lin disaggregation. Published in Economic
  Inquiry 2022 by Eichenauer et al.
- **`G-TAB`** (Google Trends Anchor Bank, Robert West, EPFL).
  Calibrates multiple Google Trends queries onto a common scale by
  chaining requests against anchor queries. Paper:
  [papers/gtab_2007.13861.pdf](../../papers/gtab_2007.13861.pdf).
- **`gluonts.dataset.repository`** (AWS). Loads Wiki-Rolling,
  electricity, traffic, and other standard TS datasets with
  consistent interfaces.
- **`darts`** (Unit8). Has dataset loaders for ETT, Electricity,
  Weather; useful for sanity-check pipelines.
- **`functime`** (polars-based forecasting library). Curated dataset
  loaders with licensing metadata.

## 6. Rough cost of reproducing TimeBench

[../benchmarks/rebuilding-timebench.md](../benchmarks/rebuilding-timebench.md)
is the full recipe. Dependency cost at a glance:

- **Data download.** LOTSA ~4 TB decompressed; Time-300B ~300 GB;
  Chronos corpus ~tens of GB; Wikimedia pageviews ~TBs if you mirror
  all languages; Google Trends is API-only (hundreds of gigs if
  bulk-scraped).
- **Disk format.** Parquet with per-shard metadata is standard.
- **Compute for curation.** ADF filtering + fuzzy dedup over 500B
  observations is a multi-day job on a single workstation; add
  sharding to run it on a small cluster.

## Related wiki pages

- [datasets-benchmarks.md](datasets-benchmarks.md) — data-section hub.
- [leakage-map.md](leakage-map.md) — which corpus overlaps which benchmark.
- [../benchmarks/rebuilding-timebench.md](../benchmarks/rebuilding-timebench.md) — the end-to-end corpus rebuild recipe.
- [../benchmarks/wikipedia-pageviews-leakage.md](../benchmarks/wikipedia-pageviews-leakage.md) — the Wikipedia-specific scrub.
- [google-trends-data.md](google-trends-data.md) — tools for Google Trends specifically.
- [../concepts/synthetic-data-augmentation.md](../concepts/synthetic-data-augmentation.md) — the leakage-free-by-construction alternative.
- [../papers/timer-s1.md](../papers/timer-s1.md) — the canonical description of the full curation pipeline.
