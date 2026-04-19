# Datasets and Benchmarks — the Data Section

This section is the **data layer** of the wiki. It documents every
named dataset relevant to modern time-series foundation models: what
it is, how big it is, who uses it, whether it is still used for
evaluation, what is known about leakage against the canonical
benchmarks, and what public tooling exists for working with it
responsibly.

If you are new here, read section 1 first. If you already know the
field, skip straight to the cross-cutting pages in section 4.

## 1. Quick orientation

There are two kinds of datasets in TS-FM research:

- **Pretraining corpora.** Large, heterogeneous collections that a
  foundation model trains on. Examples: LOTSA, Time-300B, TimeBench,
  Time Series Pile.
- **Evaluation benchmarks.** Curated, held-out datasets with fixed
  splits and standardized metrics used to compare models. Examples:
  GIFT-Eval, Monash Archive, Chronos Benchmark II, fev-bench, LTSF.

These two categories are **not disjoint**. The field's single biggest
methodology problem is that most public pretraining corpora were
built by aggregating public benchmark datasets, so a model trained on
LOTSA or Time-300B and evaluated on Monash or GIFT-Eval is often not
strictly zero-shot. This is the *leakage problem*, and it drives much
of the structure of this section.

## 2. The named pretraining corpora

| Corpus | Assembled for | ~# Obs | ~# Series | Domain mix | Released | Leakage vs GIFT-Eval test |
|---|---|---|---|---|---|---|
| [LOTSA](lotsa.md) | [MOIRAI](../papers/moirai.md) | ~27B | — | 9 domains | yes (`Salesforce/lotsa_data`) | yes, via shared Monash/ETT/Electricity — see [leakage-map.md](leakage-map.md) |
| [GIFT-Eval Pretrain](gift-eval.md) (LOTSA subset) | [Moirai 2.0](../papers/moirai-2.md) | ~230B | ~4.5M | 7 domains | yes (`Salesforce/GiftEvalPretrain`) | **clean by construction** |
| [Time-300B](time-300b.md) | [Time-MoE](../papers/time-moe.md) | ~309B | ~48M | Nature 90.5%, 9 domains nominally | yes | unaudited; overlaps public Monash/ETT/Wikipedia |
| [Time Series Pile](time-series-pile.md) | [MOMENT](../papers/moment.md) | < LOTSA | — | multi-task (forecast/class/anomaly/impute) | yes (`AutonLab/Timeseries-PILE`) | yes, via Monash inclusion |
| [TimeBench](timebench.md) | [Sundial](../papers/sundial.md) / [Timer-S1](../papers/timer-s1.md) | ~1T | — | real + synthetic, 9+ domains | **no** (inputs are public; full mix not released) | Timer-S1 explicitly scrubs GIFT-Eval test |

For how the field got here (27B → 230B → 309B → 1T) and which
corpus to pick for your own training, see
[../benchmarks/training-a-small-model.md](../benchmarks/training-a-small-model.md)
and [../benchmarks/rebuilding-timebench.md](../benchmarks/rebuilding-timebench.md).

## 3. The named evaluation benchmarks

| Benchmark | Year | Datasets / Series | Zero-shot status | Current use |
|---|---|---|---|---|
| [Monash Archive](monash-archive.md) | 2021 | dozens of datasets | only strict for models that held out the specific test splits | **still reported** for historical continuity — every TS-FM paper has a Monash row, usually asterisked for leakage |
| [GIFT-Eval](gift-eval.md) | 2024 | 23 test datasets / 144k series / 177M obs | zero-shot by construction; paired with `GIFT-Eval Pretrain` for fair comparison | **current SOTA yardstick** |
| Chronos Benchmark II | 2024 | 27 zero-shot datasets | held out during [Chronos](../papers/chronos.md) and [Chronos-2](../papers/chronos-2.md) pretraining | still reported by successors |
| fev-bench | 2025 | 98 tasks (32 univariate / 26 multivariate / 42 covariates) | leakage-audited per model | **current leaderboard** for multivariate + covariate claims |
| LTSF (ETT, Weather, ECL) | 2021-2023 | 6 datasets × 4 horizons | in-corpus for models that trained on the underlying raw data | still reported for historical continuity |

See [evaluation-benchmarks.md](evaluation-benchmarks.md) for the
consolidated breakdown of what each benchmark measures, when it was
introduced, and which benchmarks are active vs. retired.

## 4. Cross-cutting pages

- **[dataset-types.md](dataset-types.md)** — taxonomy of TS datasets:
  pretraining vs. evaluation, real vs. synthetic, univariate vs.
  multivariate, generic vs. domain-specific.
- **[evaluation-benchmarks.md](evaluation-benchmarks.md)** —
  consolidated index of evaluation suites, what they measure, their
  status (active / historical / deprecated).
- **[leakage-map.md](leakage-map.md)** — cross-reference matrix of
  pretraining corpora × evaluation benchmarks with leakage status,
  verified against primary sources.
- **[scrub-tools.md](scrub-tools.md)** — code and tooling for
  removing leakage: Salesforce `uni2ts`, `GiftEvalPretrain`,
  Timer-S1's documented curation pipeline, and external libraries.
- **[google-trends-data.md](google-trends-data.md)** — every known
  way to obtain Google Trends time-series data, with the TS-FM angle.
- **[../benchmarks/rebuilding-timebench.md](../benchmarks/rebuilding-timebench.md)** —
  step-by-step recipe for assembling a TimeBench-style corpus from
  public sources.
- **[../benchmarks/wikipedia-pageviews-leakage.md](../benchmarks/wikipedia-pageviews-leakage.md)** —
  Wikipedia pageview data: which benchmarks include it, which don't,
  and how to scrub.

## 5. Per-corpus pages

- [Monash Archive](monash-archive.md)
- [LOTSA](lotsa.md)
- [Time Series Pile](time-series-pile.md)
- [Time-300B](time-300b.md)
- [TimeBench](timebench.md)
- [GIFT-Eval](gift-eval.md) — both the evaluation suite and the
  `GiftEvalPretrain` pretraining subset.

## Related wiki pages

- [../benchmarks/benchmarks.md](../benchmarks/benchmarks.md) — head-to-head results layer.
- [../benchmarks/leaderboard.md](../benchmarks/leaderboard.md) — normalized result tables.
- [../benchmarks/state-of-the-art.md](../benchmarks/state-of-the-art.md) — who wins where.
- [../evaluation/protocols.md](../evaluation/protocols.md) — zero-shot semantics and leakage-case catalog.
- [../evaluation/comparability-checklist.md](../evaluation/comparability-checklist.md) — seven-item checklist for valid cross-paper comparison.
