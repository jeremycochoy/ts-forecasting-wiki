# Leakage Map — Corpora × Benchmarks

This page is the cross-reference matrix. For every public TS-FM
pretraining corpus and every canonical evaluation benchmark, it
records whether the corpus contains series that appear in the
benchmark, and — where audited — the source of the evidence.

Every claim on this page is verified against a specific table or
figure in a primary source. Where no audit exists, the cell is
marked "unaudited" and the implication is "assume the worst."

## 1. Headline matrix

Rows are pretraining corpora. Columns are evaluation benchmarks.

| Corpus ↓ / Benchmark → | GIFT-Eval test | Monash | Chronos Bench II | fev-bench | LTSF (ETT/Weather/ECL) |
|---|---|---|---|---|---|
| [LOTSA](lotsa.md) (original) | **yes** — shared Monash / M4 / ETT / Electricity entries; Salesforce re-trained Moirai on `GiftEvalPretrain` for a fair comparison and renamed the original "Moirai-Leakage" (GIFT-Eval §F.2) | yes | held out by Chronos; LOTSA is separate | no formal audit | in-corpus (ETT/ECL are in LOTSA) |
| [GIFT-Eval Pretrain](gift-eval.md) | **clean by construction** (GIFT-Eval Table 14 excludes all Table 13 test series) | partial overlap (shares some Monash not in GIFT-Eval test) | no formal audit | no formal audit | in-corpus for ETT/ECL |
| [Time-300B](time-300b.md) | unaudited; Time-MoE pre-dates GIFT-Eval | in-corpus (Monash datasets imported via Time-MoE Table 10) | no formal audit | no formal audit | in-corpus (ETT/ECL imported) |
| [Time Series Pile](time-series-pile.md) | unaudited; includes Monash | **yes** (MOMENT uses Monash forecasting subset) | no formal audit | no formal audit | in-corpus (ETT/ECL in TSLib subset) |
| [Chronos](../papers/chronos.md) v1 corpus (28 datasets) | **yes** — Chronos Benchmark I includes M4 Daily/Hourly/Monthly/Weekly, Electricity, ETT, KDD Cup 2018, Temperature-Rain, Pedestrian Counts, London Smart Meters, which are in GIFT-Eval Table 13 | **yes** — Moirai-MoE Figure 3 asterisks Chronos-Small/Base/Large* | zero-shot by design (Chronos §5.1 footnote 5) | no formal audit | in-corpus |
| [Chronos-2](../papers/chronos-2.md) corpus | "partial overlap with training portions of some GIFT-Eval datasets; no overlap with test portions at any sampling frequency" (Chronos-2 §5.1) | partial; not explicitly audited | strict zero-shot (re-held-out during Chronos-2 pretraining) | audited: 0% leakage (Chronos-2 Table 3) | — |
| [TimesFM](../papers/timesfm.md) v1 corpus | **yes** — explicit M4 (all granularities) / Electricity / Traffic / Weather inclusion (TimesFM Table 1), all in GIFT-Eval test | **yes** — Moirai-MoE Figure 3 asterisks TimesFM* | no formal audit; predates Chronos Benchmark II | audited: 8% leakage (Chronos-2 Table 3, for TimesFM-2.5 evaluated against fev-bench) | in-corpus (ETT/Weather/ECL via LibCity inclusion) |
| [TimeBench](timebench.md) (Sundial + Timer-S1) | **clean as of Timer-S1 release** — explicitly scrubs GIFT-Eval leakage (Timer-S1 §4.2); paper confirms "test-data leakage removal against GIFT-Eval" | partial | — | — | in-corpus for LTSF sources |

Reading the matrix:

- **"yes"** means the corpus contains series that literally appear in
  the benchmark's test split. This breaks zero-shot claims.
- **"no formal audit"** means the paper does not audit this overlap.
  In practice, any corpus that aggregates public data from 2021-2024
  sources should be assumed to overlap with Monash-derived
  benchmarks.
- **"held out" / "clean by construction"** means the corpus was
  specifically built to exclude the benchmark test windows.

## 2. Per-source evidence

### GIFT-Eval test (Aksu et al., arXiv:2410.10393)

- The 23 test datasets are listed in Table 13. Key datasets with
  leakage implications: `M4 Yearly/Quarterly/Monthly/Weekly/Daily/
  Hourly` (Monash source), `Electricity` (UCI), `ETT1/ETT2` (Informer),
  `KDD Cup 2018` (Monash), `Temperature Rain` (Monash), `COVID Deaths`
  (Monash), `Hospital` (Monash), `Saugeen` (Monash), `Car Parts`
  (Monash), `US Births` (Monash). The Web/CloudOps test subset is
  exclusively CloudOps (`BizITObs × 3`, `Bitbrains × 2`) — no
  Wikipedia.
- The GIFT-Eval `Pretrain` set (Table 14) is engineered to be
  disjoint from the test set. Wikipedia-derived series
  (`Wiki-Rolling`, `Kaggle Web Traffic Weekly`, `Extended Web
  Traffic`) live here, not in the test split.
- §F.2 of the GIFT-Eval paper runs a formal leakage-vs-no-leakage
  comparison on MOIRAI: "Moirai-Leakage" denotes the original
  LOTSA-trained checkpoint; the retrained-on-`GiftEvalPretrain`
  variant is the fair baseline. Leakage effect increases with
  prediction length.

### Monash Archive (Godahewa et al., NeurIPS 2021)

- [Moirai-MoE](../papers/moirai-moe.md) Figure 3 caption is the
  clearest public leakage statement: "We use asterisks (*) to mark
  the methods that used the evaluation datasets here in their
  pretraining corpora. … TimesFM* Chronos-Small* Chronos-Base*
  Chronos-Large*". Table 2 in the same paper repeats the
  asterisks for Chronos + TimesFM on the 10-dataset held-out split.
- Monash's own dataset does not label pretraining leakage — the
  archive was published in 2021 before any TS-FM existed.

### Chronos Benchmark II

- Chronos paper §5.1 footnote 5: "we consider the risk [of leakage]
  to be minimal given that the datasets bear no overlap beyond
  high-level conceptual categorization."
- Chronos-2 §5.1 restates the held-out guarantee.
- Benchmark II is the closest the field has to a genuinely
  zero-shot benchmark for models in the Chronos lineage.

### fev-bench (Shchur et al., 2025)

- Chronos-2 Table 3 is the authoritative leakage audit: **Chronos-2
  0%, TiRex 1%, TimesFM-2.5 8%, Toto-1.0 8%, Moirai-2.0 28%,
  COSMIC 0%, Chronos-Bolt 0%, TabPFN-TS 0%, Sundial 1%**.
- The baseline scores and the imputation strategy for handling
  leakage in individual tasks are documented in Chronos-2 §5.1 and
  in Shchur et al. 2025.

## 3. Implications for your training

If you want a clean zero-shot GIFT-Eval claim:

1. Train on `Salesforce/GiftEvalPretrain`. Do not manually add raw
   Monash, M4, ETT, or Electricity data — they all overlap the
   test split.
2. Synthetic additions (KernelSynth, TSMix, canonical signal
   families) are safe to layer on top — they generate clean data
   by construction.
3. Raw Wikimedia pageviews are safe against GIFT-Eval test (none
   of GIFT-Eval's test series come from Wikipedia). They do
   overlap with Monash's `Kaggle Web Traffic` entries, so if you
   also evaluate on Monash you need the scrub described in
   [../benchmarks/wikipedia-pageviews-leakage.md](../benchmarks/wikipedia-pageviews-leakage.md).
4. If you want strict Monash zero-shot, you need to scrub against
   the Kaggle 2017 Web Traffic competition article list plus the
   M4 / Electricity / ETT / Weather / KDD Cup 2018 series.

See [scrub-tools.md](scrub-tools.md) for the code.

## 4. Gaps and open audits

The following cells in the headline matrix are documented as
"unaudited" — a published leakage audit does not exist:

- Time-300B vs GIFT-Eval (Time-MoE predates GIFT-Eval).
- Time-300B vs Chronos Benchmark II.
- Time-300B vs fev-bench.
- Time Series Pile vs GIFT-Eval / Chronos II / fev-bench.

A community contribution of these audits would close the field's
largest remaining leakage gap. See
[../research/contributing.md](../research/contributing.md).

## Related wiki pages

- [datasets-benchmarks.md](datasets-benchmarks.md) — data-section hub.
- [evaluation-benchmarks.md](evaluation-benchmarks.md) — per-benchmark detail.
- [scrub-tools.md](scrub-tools.md) — code for leakage removal.
- [../benchmarks/wikipedia-pageviews-leakage.md](../benchmarks/wikipedia-pageviews-leakage.md) — Wikipedia-specific scrub.
- [../benchmarks/rebuilding-timebench.md](../benchmarks/rebuilding-timebench.md) — how to rebuild a clean corpus.
- [../evaluation/protocols.md](../evaluation/protocols.md) — zero-shot definitions.
- [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md) — §1 "Zero-shot is paper-relative".
