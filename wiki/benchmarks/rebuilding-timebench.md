# Rebuilding a TimeBench-style Corpus from Public Sources

This page is a practical recipe for assembling a **TimeBench-equivalent
pretraining corpus** using only publicly released data, starting from
the `GIFT-Eval Pretrain` subset of [LOTSA](../datasets-benchmarks/lotsa.md).
[TimeBench](../datasets-benchmarks/timebench.md) itself — the ~1 trillion-point
corpus used by [Sundial](../papers/sundial.md) and [Timer-S1](../papers/timer-s1.md) —
has not been re-released in auditable form, but both its ingredient
list and the [Timer-S1](../papers/timer-s1.md) curation pipeline are
documented in enough detail that an independent rebuild is possible.

The reader has a target GIFT-Eval evaluation, already owns `GIFT-Eval Pretrain`,
and wants the missing pieces plus the filters to apply before training.

## 1. TL;DR — the finished mix

| Layer | Source | Approx. size (obs) | Leakage status vs GIFT-Eval test | Action |
|---|---|---|---|---|
| Core (have it) | `GIFT-Eval Pretrain` (LOTSA subset) | 230B obs / ~4.5M series | clean by construction (Salesforce curated) | use as-is |
| Real add-on | [Time-300B](../datasets-benchmarks/time-300b.md) | ~309B obs / ~48M series | unaudited; contains Wiki-Rolling + Kaggle Web Traffic + Extended Web Traffic per Time-MoE Table 10 | dedup vs Pretrain + GIFT-Eval scrub |
| Real add-on | Chronos public corpus (28 datasets) | ~84B obs / ~890K series (Chronos §5.1) | direct overlap: Chronos Benchmark I includes M4/ETT/Electricity/KDD Cup 2018/Temperature-Rain — all in GIFT-Eval test (Table 13) | dedup + scrub mandatory |
| Real add-on (optional) | [Time Series Pile](../datasets-benchmarks/time-series-pile.md) | smaller, multi-task | inherits Monash overlap (M4, Electricity, ETT) | skip for pure forecasting |
| Synthetic | KernelSynth (GP) | scale to taste | zero by construction | generate |
| Synthetic | TSMix (convex mixup) | scale to taste | inherits from inputs | generate after scrubbing inputs |
| Synthetic | Canonical signal families | scale to taste | zero by construction | generate |

**Crucial finding for this recipe.** GIFT-Eval test contains **zero
Wikipedia-derived series** (GIFT-Eval Table 13, arXiv:2410.10393) — all
Web/CloudOps test datasets are CloudOps (BizITObs + Bitbrains). This
means folding raw Wikimedia pageviews into your corpus does **not**
leak w.r.t. GIFT-Eval test. The real leakage pathways against
GIFT-Eval test are the Monash / M4 / ETT / Electricity entries that
Chronos-v1 and LOTSA both inherit. See
[wikipedia-pageviews-leakage.md](wikipedia-pageviews-leakage.md) for
the specific scrub procedure.

Expected real-data total after scrub/dedup: ~450–550B observations
(heavy overlap between Time-300B, Chronos, and LOTSA; the three share
M4, Electricity, Traffic, Weather, KDD Cup 2018, Wiki-Rolling,
Kaggle Web Traffic, and more). Remaining gap to reach the 1T
TimeBench headline is filled with synthetic generation.

## 2. Starting point — `GIFT-Eval Pretrain`

`GIFT-Eval Pretrain` is the LOTSA-derived, leakage-filtered subset used
by [Moirai 2.0](../papers/moirai-2.md). It contains 230B observations
and is already scrubbed against the [GIFT-Eval](../datasets-benchmarks/gift-eval.md)
test splits (see [lotsa.md § Related / derived corpora](../datasets-benchmarks/lotsa.md)).
Distributed via Salesforce's `uni2ts` tooling alongside `Salesforce/lotsa_data`.

Because it is pre-scrubbed, **no additional GIFT-Eval filtering is
required on this layer** — but every subsequent addition must pass
through the full pipeline in section 6.

## 3. Layer 1 — Time-300B

[Time-300B](../datasets-benchmarks/time-300b.md) is [Time-MoE](../papers/time-moe.md)'s
corpus, ~309B observations across 9 domains (Time-MoE Table 1),
released alongside Time-MoE's weights. It is the single biggest
public addition.

Caveats (verified against Time-MoE Tables 1 and 10):

- **No published GIFT-Eval leakage audit.** Time-MoE was assembled
  before GIFT-Eval and is not part of the Timer-S1 scrub
  documentation. You must apply the scrub yourself.
- **Heavily skewed domain mix.** Nature is **90.50%** of observations
  (279.7B / 309.1B) per Time-MoE Table 1, so the effective diversity
  is much lower than the 309B headline.
- **Overlap with LOTSA is substantial.** Time-MoE Table 10 lists
  per-dataset sources; the overlap with LOTSA includes at least
  `Alibaba Cluster Trace 2018`, `Azure VM Traces 2017`,
  `Borg Cluster Data 2011`, `Kaggle Web Traffic Weekly`
  (133,388 series), `Extended Web Traffic` (161,890 series),
  `Wiki-Rolling` (47,675 series), and `Wiki Daily (100k)`
  (100,001 series, sourced from Chronos/Ansari et al. 2024).
  A dataset-level dedup pass against `GIFT-Eval Pretrain` is
  mandatory before counting observations.

Expected clean yield after dedup + scrub: **on the order of 200–280B
observations**, depending on how aggressive the fuzzy dedup is.

## 4. Layer 2 — Chronos public corpus snapshot

[Chronos](../papers/chronos.md)'s pretraining corpus is 28 datasets
(13 pretraining-only + 15 Benchmark I in-domain), **~890K univariate
series totaling ~84B observations** per Chronos §5.1. Distributed via
the HuggingFace dataset card `autogluon/chronos_datasets`, plus the
`amazon-science/chronos-forecasting` repo.
[TimeBench](../datasets-benchmarks/timebench.md) explicitly reuses
"public dataset snapshots from Chronos"
(see [datasets-benchmarks/timebench.md § Overview](../datasets-benchmarks/timebench.md)).

Leakage status against GIFT-Eval test (verified against Chronos
Table 3 / Appendix B and GIFT-Eval Table 13):

- **Chronos Benchmark II (27 datasets) is zero-shot by Chronos's own
  design** — the paper's footnote 5 says the leakage risk is "minimal
  given that the datasets bear no overlap beyond high-level conceptual
  categorization." So Benchmark II is not the leakage source.
- **Chronos Benchmark I (15 in-domain datasets) is the leakage
  source.** It includes `M4 (Daily/Hourly/Monthly/Weekly)`,
  `Electricity (15 Min/Hourly/Weekly)`, `ETT (15 Min/Hourly)`,
  `KDD Cup 2018`, `London Smart Meters`, `Pedestrian Counts`,
  `Rideshare`, `Taxi (30 Min)`, `Temperature-Rain`, and
  `Uber TLC (Daily/Hourly)`. Several of these are in **GIFT-Eval test**
  (GIFT-Eval Table 13): `M4 Yearly/Quarterly/Monthly/Weekly/Daily/Hourly`,
  `Electricity` (UCI), `ETT1/ETT2`, `KDD Cup 2018`,
  `Temperature Rain`.
- Chronos pretraining-only set also includes `Wiki Daily (100k)` —
  100,001 Wikipedia articles daily — but this does not leak against
  GIFT-Eval test, which has no Wikipedia data. See
  [wikipedia-pageviews-leakage.md](wikipedia-pageviews-leakage.md).

Expected clean yield after dedup + scrub: **a non-trivial fraction of
the 84B** — the 13 pretraining-only datasets + the Benchmark II
datasets (zero-shot for Chronos, but most of those overlap GIFT-Eval
via common Monash sources, so the scrub still removes them). The
remaining clean content is dominated by the pretraining-only set
(Brazilian Cities Temperature, Mexico City Bikes, Solar, Spanish
Energy and Weather, Taxi, USHCN, Weatherbench, Wiki Daily, Wind
Farms) minus whatever overlaps with Pretrain's existing entries.

## 5. Layer 3 — Synthetic generators

Synthetic generation is the only knob that produces arbitrarily many
clean points. Three named recipes plus an optional fourth are
documented in [concepts/synthetic-data-augmentation.md](../concepts/synthetic-data-augmentation.md):

- **KernelSynth (Chronos).** Gaussian-process samples with random
  kernel compositions. Cheap, license-free, **zero leakage by
  construction**. Scale to whatever point count you need. Original
  Chronos used 1M series / ~1.02B points (see [lotsa.md § Moirai-2 derivative mixture](../datasets-benchmarks/lotsa.md)).
- **TSMix (Chronos).** Convex Dirichlet-weighted mixtures of real
  series. Inherits leakage from its inputs: only apply **after** the
  GIFT-Eval scrub of layers 1–2. Chronos used 10M mixed series as a
  default.
- **Canonical signal families.** The primitives named in the Sundial
  / Timer-S1 TimeBench description: linear, sinusoidal, exponential,
  power, impulse, step (see [datasets-benchmarks/timebench.md § Overview](../datasets-benchmarks/timebench.md)).
  Exact parameterization is not published — you match the spirit, not
  the bits.
- **Optional: ForecastPFN priors.** Used by [Mamba4Cast](../papers/mamba4cast.md)
  (70% GP, 30% ForecastPFN). Adds a second synthetic prior family if
  you want more diversity than KernelSynth alone.

Use synthetic generation to fill the gap between your cleaned real
total and the 1T target.

## 6. The mandatory curation pipeline

Apply the full [Timer-S1](../papers/timer-s1.md) pipeline to **every
real-data layer you add** (layers 3 and 4; not needed on
`GIFT-Eval Pretrain` which is pre-scrubbed). Source:
[training-a-small-model.md § The Timer-S1 curation pipeline as a recipe](training-a-small-model.md)
and [papers/timer-s1.md § Key contributions](../papers/timer-s1.md).

1. **Causal mean imputation** for missing values. No future-into-past
   leakage through the imputer.
2. **Outlier removal** via k-σ or IQR on shifting windows.
3. **ADF predictability filter.** Series that fail the Augmented
   Dickey-Fuller test are discarded or down-weighted. The argument is
   that unpredictable-noise series degrade pretraining signal.
4. **Eval-leakage scrub.** Explicit series-level dedup against the
   GIFT-Eval test splits *and* against Chronos Benchmark II. Do this
   with dataset-name match first, then a fuzzy-hash or nearest-neighbor
   check on series windows to catch renamed redistributions.
5. **Resampling augmentation.** Downsample plus Fourier interpolation
   to expose the model to multiple temporal resolutions from a single
   source.
6. **Value-flipping augmentation.** Multiply by −1 to invert trends
   while preserving dependencies. **Do not apply to strictly
   non-negative series** (counts, prices, utility demand).

Run steps 1–4 on layers 3 and 4 before counting observations. Run
steps 5–6 as a final augmentation pass across the whole corpus.

## 7. Size budget worksheet

Cumulative observations at each stage, assuming typical dedup /
scrub overheads. Source numbers: GIFT-Eval Pretrain §3 / Table 14
(230B); Time-MoE Table 1 (309B); Chronos §5.1 (84B); paper-stated
augmentation counts for KernelSynth/TSMix.

| Stage | Cumulative real obs | Synthetic budget to add | Notes |
|---|---|---|---|
| After `GIFT-Eval Pretrain` | 230B | 0 | already clean |
| + Time-300B (post-scrub) | ~450–510B | 0 | dedup vs Pretrain drops shared LOTSA/Monash entries; ~90% of Time-300B is Nature-domain |
| + Chronos corpus (post-scrub) | ~460–530B | 0 | Benchmark I overlaps GIFT-Eval test directly; after scrub only the 13 pretraining-only datasets + Wiki Daily + synthetic-incompatible entries remain |
| + KernelSynth | same | ~100–300B | generate to any target |
| + TSMix | same | ~100–300B | inputs must be already scrubbed |
| + Canonical signal families | same | ~50–200B | cheap, composable |
| **Target** | — | — | **~1T points** to match TimeBench |

The synthetic layers are where you close the gap. This is also why
TimeBench's per-source weighting is "not published in auditable form"
(see [papers/timer-s1.md § Limitations and open critiques](../papers/timer-s1.md)):
the 1T headline is not necessarily 1T of *real* observations.

## 8. What you cannot faithfully reproduce

Be explicit about the limits in any paper that builds on this recipe:

- **TimeBench's exact source mixture and per-domain weights** are not
  published (see [papers/timer-s1.md § Limitations and open critiques](../papers/timer-s1.md)).
  Your rebuild will match the *composition* of TimeBench but not the
  exact sampling ratios.
- **The CloudOps telemetry** used in the Moirai-2 mixture
  (~2.15M series / ~1.48B daily observations; see
  [lotsa.md § Related / derived corpora](../datasets-benchmarks/lotsa.md))
  is proprietary and unavailable.
- **Timer-S1's exact ADF threshold, k-σ multiplier, IQR bounds, and
  fuzzy-dedup threshold** are not stated numerically in the paper.
  You set them and document them.
- **Any paper published against your rebuild should label results as
  "trained on a TimeBench-style corpus," not "trained on TimeBench,"**
  and ship the full source list, weights, and filter parameters
  alongside the model.

## 9. Why this is worth doing

From [benchmarks/training-a-small-model.md § 6 Rolling your own corpus](training-a-small-model.md):
*"Data curation is the contribution of [Timer-S1](../papers/timer-s1.md)
precisely because it is hard."* A transparent, reproducible rebuild
is itself a contribution — it is the one thing Timer-S1 does not
deliver, and the one thing the field most needs for trustworthy
zero-shot claims on [GIFT-Eval](../datasets-benchmarks/gift-eval.md).

## Related wiki pages

- [wikipedia-pageviews-leakage.md](wikipedia-pageviews-leakage.md) — concrete audit of Wikipedia data in GIFT-Eval (test has none; Pretrain has three datasets), with scrub procedure if you add raw Wikimedia pageviews on top.
- [datasets-benchmarks/timebench.md](../datasets-benchmarks/timebench.md) — the target corpus description.
- [datasets-benchmarks/lotsa.md](../datasets-benchmarks/lotsa.md) — base corpus and `GIFT-Eval Pretrain` provenance.
- [datasets-benchmarks/time-300b.md](../datasets-benchmarks/time-300b.md) — biggest public add-on.
- [datasets-benchmarks/time-series-pile.md](../datasets-benchmarks/time-series-pile.md) — optional multi-task layer.
- [datasets-benchmarks/gift-eval.md](../datasets-benchmarks/gift-eval.md) — the evaluation benchmark everything is scrubbed against.
- [concepts/synthetic-data-augmentation.md](../concepts/synthetic-data-augmentation.md) — KernelSynth, TSMix, ForecastPFN recipes.
- [papers/timer-s1.md](../papers/timer-s1.md) — the curation pipeline's canonical source.
- [papers/sundial.md](../papers/sundial.md) — the paper that introduced the TimeBench name.
- [benchmarks/training-a-small-model.md](training-a-small-model.md) — companion guide on architecture, head, and compute choices.
- [evaluation/protocols.md](../evaluation/protocols.md) — known leakage cases and leakage-free cross-paper benchmarks.
- [evaluation/comparability-checklist.md](../evaluation/comparability-checklist.md) — the seven-item checklist a reviewer will apply to your results.
