# Training a Small Time-Series Foundation Model

This page is the training-side companion to
[univariate-benchmarking.md](univariate-benchmarking.md). That page
answered *"where do I evaluate my model so its row lands next to 2024–
2026 TS-FM SOTA."* This page answers the question that comes
immediately before it: *"where do I train that model, and how big is
the smallest competitor I actually have to beat?"* It is written for
a reader who is not shipping a billion-parameter frontier model but
who wants a reproducible, defensible training setup inside the same
conversation as [Chronos](../papers/chronos.md),
[MOIRAI](../papers/moirai.md), [MOMENT](../papers/moment.md), and
[TTM](../papers/ttm.md).

## TL;DR recommendation

> If you are training a TS-FM in the 5M–100M parameter range and you
> want a competitive, open, reproducible setup, train on
> **[LOTSA](../datasets-benchmarks/lotsa.md)** (the [MOIRAI](../papers/moirai.md)
> corpus, ~27B observations across nine domains, public on HuggingFace
> as `Salesforce/lotsa_data`) with a **RevIN + patch tokenization +
> masked-encoder or decoder-only backbone**. The "smallest SOTA" tier
> you need to slot next to is **[TTM](../papers/ttm.md) at 1–5M** at
> the bottom, **[MOIRAI-Small](../papers/moirai.md) (14M) /
> [Lag-Llama](../papers/lag-llama.md) /
> [Mamba4Cast](../papers/mamba4cast.md) /
> [Chronos-Mini-Small](../papers/chronos.md) (20–46M)** in the tiny
> transformer tier, and **[MOMENT-Small](../papers/moment.md) (40M) /
> [MOIRAI-Base](../papers/moirai.md) (91M) /
> [MOMENT-Base](../papers/moment.md) (125M)** at the upper end of
> "small." If your model lands above ~200M it is no longer competing
> in the "small" bracket — it is in the base tier alongside
> [Chronos-Base](../papers/chronos.md),
> [TimesFM](../papers/timesfm.md), and
> [MOIRAI-Large](../papers/moirai.md). Apply the
> [Timer-S1](../papers/timer-s1.md) curation pipeline
> (imputation / outlier removal / ADF filter / GIFT-Eval leakage
> removal) on top of LOTSA before training if you want a clean
> zero-shot claim.

## What "not too big" means in this field

TS-FM parameter counts have drifted upward from the sub-10M
[TTM](../papers/ttm.md) tier to the multi-billion
[Time-MoE](../papers/time-moe.md) / [Timer-S1](../papers/timer-s1.md)
tier, and "small" has drifted with them. The brackets below are
calibrated to 2025–2026. Sizes come from each paper's reproducibility
section; "—" means not disclosed in the extracted leaves.

| Bracket | Parameter range | Representative models | Notes |
|---|---|---|---|
| Tiny | 1–10M | [TTM](../papers/ttm.md) (1M / 4M / 5M, TSMixer), [Lag-Llama](../papers/lag-llama.md) (d=144, L=8, 2.4M), [Mamba4Cast](../papers/mamba4cast.md) (d=1024, 2 SSM blocks, 27M) | CPU inference is realistic here; single consumer GPU for training. |
| Small | 10–50M | [Chronos-Mini](../papers/chronos.md) (d=384, L=4+4 enc-dec, 20M), [Chronos-Small](../papers/chronos.md) (d=512, L=6+6 enc-dec, 46M), [MOIRAI-Small](../papers/moirai.md) (d=384, L=6, 14M), [MOMENT-Small](../papers/moment.md) (d=512, L=6, 40M) | Single consumer GPU inference and training; the "interesting" regime for a solo researcher. |
| Base | 50–200M | [Chronos-Base](../papers/chronos.md) (d=768, L=12+12 enc-dec, 200M), [Chronos-2](../papers/chronos-2.md) (120M, d/L not disclosed), [MOMENT-Base](../papers/moment.md) (d=768, L=12, 125M), [MOIRAI-Base](../papers/moirai.md) (d=768, L=12, 91M), [TimesFM](../papers/timesfm.md) (d=1280, L=20, 200M), [Timer](../papers/timer.md)/[Timer-XL](../papers/timer-xl.md) (d=1024, L=8, 84M) | Single datacenter GPU or 2–4× consumer GPUs for training. |
| Large | 200M–1B | [Chronos-Large](../papers/chronos.md) (d=1024, L=24+24 enc-dec, 710M), [MOMENT-Large](../papers/moment.md) (d=1024, L=24, 385M), [MOIRAI-Large](../papers/moirai.md) (d=1024, L=24, 311M), [Sundial-Large](../papers/sundial.md) (d=1024, L=24, 444M) | Multi-GPU training. |
| Huge | 1B+ | [Time-MoE](../papers/time-moe.md) Ultra (d=1024, L=36, 2.4B total / 1.1B active), [Timer-S1](../papers/timer-s1.md) (d=1024, L=24, 8.3B total / 0.75B active) | Sparse MoE; no longer "small" by any definition. |

Two caveats: [Chronos](../papers/chronos.md)'s four sizes are named
Mini 20M / Small 46M / Base 200M / Large 710M per §5.2 of the paper
(not Tiny / Mini / Small / Base / Large as sometimes misreported),
each mapping onto a `google/t5-efficient-{mini,small,base,large}`
config. [Lag-Llama](../papers/lag-llama.md)'s published checkpoint
has 2.45M params (`d=144, L=8, H=9`, Rasul et al. App. D);
[Mamba4Cast](../papers/mamba4cast.md) is 27M (`d=1024`, 2 stacked
Mamba2 encoder blocks, state N=128).

For concrete `(hidden_dim, num_layers)` combinations that land a
decoder-only transformer in each of these brackets — and for the
`12·d²·L` formula that underlies them — see
[model-sizing-cheatsheet.md](model-sizing-cheatsheet.md). The cheat
sheet is the answer to "how many layers at `d=512` / `768` / `1024` do
I need to match [Chronos-Mini](../papers/chronos.md) /
[MOMENT-Base](../papers/moment.md) / [TimesFM](../papers/timesfm.md)?"

## "Biggest of the smallest" — the actual answer

The user question this page is built around is: *what is the biggest
parameter count among the smallest SOTA models — the ceiling I have
to beat to be taken seriously in the small-TS-FM category?*

- The **floor** is set by [TTM](../papers/ttm.md) at **1–5M
  parameters**. If your model is bigger than 5M and loses to TTM
  zero-shot, TTM's Figure 1 Pareto plot (87× to 7000× faster than
  Chronos-T / Chronos-B on CPU while more accurate) is the exact
  figure a reviewer will point you at. The first hurdle is not
  MOIRAI; it is TTM.
- The **upper boundary** of "small" is the
  [MOIRAI-Base](../papers/moirai.md) (91M) /
  [MOMENT-Base](../papers/moment.md) (125M) line. Above this you are
  in the *base* tier, competing with [Chronos-Base](../papers/chronos.md),
  [TimesFM](../papers/timesfm.md), [MOMENT-Large](../papers/moment.md),
  and [MOIRAI-Large](../papers/moirai.md).
- The **biggest of the smallest** — the models you most need to beat
  to claim "competitive small TS-FM" — are **MOIRAI-Base at 91M** and
  **MOMENT-Base at 125M**.

If your model is under ~100M parameters with competitive zero-shot
numbers on [GIFT-Eval](../datasets-benchmarks/gift-eval.md) or Chronos
Benchmark II, you are in the interesting regime. Above ~200M, frame
your contribution as base-tier, not small.

## Training corpora — ranked for a small-model setting

Five corpora matter in the 2024–2026 literature. All but the last two
are publicly downloadable.

### 1. LOTSA — the recommended default

[LOTSA](../datasets-benchmarks/lotsa.md) is
[MOIRAI](../papers/moirai.md)'s pretraining corpus: ~27B observations
across 9 domains (energy, transport, climate, CloudOps, web, sales,
healthcare, nature, economics), sub-hourly to yearly. Distributed on
HuggingFace as `Salesforce/lotsa_data`, consumed via the `uni2ts`
training library. It is the single corpus on which the largest number
of 2024–2026 TS-FM numbers are reported under a transparent protocol:
used by [MOIRAI](../papers/moirai.md) (three sizes),
[Moirai-MoE](../papers/moirai-moe.md), and reused as a snapshot inside
[TimeBench](../datasets-benchmarks/timebench.md) per
[Timer-S1](../papers/timer-s1.md). **Recommended default for most
researchers.** Unless you have a specific reason to pick another
corpus, train on LOTSA.

### 2. Time Series Pile — multi-task default

The [Time Series Pile](../datasets-benchmarks/time-series-pile.md) is
[MOMENT](../papers/moment.md)'s corpus, distributed on HuggingFace as
`AutonLab/Timeseries-PILE`. Smaller than LOTSA but explicitly
multi-task: aggregates Informer/LTSF datasets, the
[Monash Archive](../datasets-benchmarks/monash-archive.md), UCR/UEA
classification, and the TSB-UAD anomaly archive, with documented
disjoint splits. Pick this if your model targets classification,
imputation, or anomaly detection in addition to forecasting. For pure
forecasting, LOTSA is still the default.

### 3. Time-300B — scale beyond LOTSA

[Time-300B](../datasets-benchmarks/time-300b.md) is
[Time-MoE](../papers/time-moe.md)'s ~309B-point corpus, roughly ten
times LOTSA. Open, released with the Time-MoE weights. Two caveats:
(i) the domain mix is heavily skewed — "Nature" is ~90% of
observations — so effective diversity is lower than the headline; (ii)
iterating over 300B points demands real data-loading engineering
(sharded parquet, hybrid loaders). Pick it only if you have saturated
LOTSA and specifically want to study data-axis scaling.

### 4. TimeBench — the 2026 frontier recipe

[TimeBench](../datasets-benchmarks/timebench.md) is the ~1T-point
corpus introduced by [Sundial](../papers/sundial.md) and documented in
detail by [Timer-S1](../papers/timer-s1.md): a curated aggregation
built on Chronos's corpus and LOTSA with an explicit pipeline
(imputation, k-sigma/IQR, ADF filter, GIFT-Eval leakage removal,
resampling and value-flipping augmentation). As of the Timer-S1
preprint, the full mixture has not been re-released in auditable
form — only the public snapshots it reuses. The pipeline itself is
applicable on top of LOTSA or Time-300B even if you are not training
on TimeBench. The contribution is the pipeline, not the name.

### 5. Synthetic-only

[Chronos](../papers/chronos.md) introduced **KernelSynth** (GP-generated
series) and **TSMix** (convex mixtures of real series) as augmentation,
with a 10M mixed + 1M GP-synthesized 9:1 default mix.
[Mamba4Cast](../papers/mamba4cast.md) goes further: it trains entirely
on synthetic priors (70% GP, 30% ForecastPFN) and is competitive with
Chronos-Small on 17 real-world datasets while sidestepping licensing
and leakage entirely. Synthetic-only is attractive under storage/compute
pressure, but there is an open question about whether it hits a ceiling
real-data training does not (see
[../research/open-problems.md](../research/open-problems.md)). The
conservative choice is: train on LOTSA, augment with KernelSynth and
TSMix, report both rows.

### 6. Rolling your own corpus

Possible but strongly discouraged. Data curation is the contribution
of Timer-S1 precisely because it is hard — deduplication, outlier
handling, stationarity filtering, and eval-leakage removal are each
nontrivial and compound unpleasantly. If you roll your own, document
every filter and expect reviewers to demand an ablation isolating data
quality from architecture.

## The Timer-S1 curation pipeline as a recipe

The [Timer-S1](../papers/timer-s1.md) paper is the most detailed
public writeup of TS-FM data curation as of 2026-04. The pipeline is:

1. **Causal mean imputation** for missing values (no leakage from
   future observations).
2. **Outlier removal** via **k-sigma** or **IQR** filters on shifting
   windows.
3. **ADF-based stationarity / predictability filter** — series that
   fail the Augmented Dickey-Fuller test are discarded or reweighted,
   on the argument that un-predictable noise degrades pretraining.
4. **Eval-leakage removal** — explicit deduplication against the
   [GIFT-Eval](../datasets-benchmarks/gift-eval.md) and Chronos
   Benchmark II test splits, inherited and extended from Sundial.
5. **Resampling augmentation** — downsampling and Fourier
   interpolation to expose the model to multiple resolutions.
6. **Value-flipping augmentation** — multiplying a series by -1 to
   invert its trend while preserving temporal dependencies (do not
   apply this to strictly non-negative series like counts or prices).

Apply this pipeline to LOTSA (or Time-300B) before training, even if
you never touch TimeBench. It is currently the best-documented
curation recipe in the field and is citable as a stand-alone
contribution.

## Recipe for a competitive small model

A concrete, opinionated stack that stays inside the small bracket:

- **Normalization:** RevIN per-series (instance normalization with a
  learnable affine, applied before patching and reversed after the
  head). See [../concepts/revin-normalization.md](../concepts/revin-normalization.md).
  Every small TS-FM in the wiki uses RevIN or a close variant.
- **Tokenization:** patch tokenization, patch size 16–32 — standard
  across [MOMENT](../papers/moment.md) (P=8),
  [Timer-S1](../papers/timer-s1.md) (P=16),
  [Sundial](../papers/sundial.md) (P=16). See
  [../concepts/patch-tokenization.md](../concepts/patch-tokenization.md).
- **Backbone.** Pick one: (a) T5 encoder-decoder à la
  [Chronos](../papers/chronos.md); (b) masked encoder à la
  [MOMENT](../papers/moment.md) / [MOIRAI](../papers/moirai.md) for
  multi-task or variable-variate support; (c) decoder-only next-patch
  à la [TimesFM](../papers/timesfm.md) / [Timer](../papers/timer.md)
  for LLM-mainstream generative; (d) TSMixer / MLP-Mixer à la
  [TTM](../papers/ttm.md) to stay under 10M and target CPU.
- **Loss / head** (decides which leaderboard column you enter):
  cross-entropy over quantized values
  ([Chronos](../papers/chronos.md) — MASE + WQL via sampling);
  parametric mixture-of-Student-t
  ([MOIRAI](../papers/moirai.md)) or single Student-t
  ([Lag-Llama](../papers/lag-llama.md)) for native CRPS; quantile-CRPS
  ([Timer-S1](../papers/timer-s1.md), nine quantiles with weighted
  quantile loss, what GIFT-Eval actually scores); flow matching
  ([Sundial](../papers/sundial.md)); or point regression
  ([TTM](../papers/ttm.md), [Mamba4Cast](../papers/mamba4cast.md),
  [MOMENT](../papers/moment.md) — MASE/MSE only). Pick a probabilistic
  head if you plan to report WQL on GIFT-Eval.
- **Context length:** 512 is standard (Chronos, MOMENT); 1024–2048 for
  long-horizon work ([Sundial](../papers/sundial.md) 2880,
  [Timer-S1](../papers/timer-s1.md) extends to 11,520 via RoPE
  post-training). See
  [../evaluation/protocols.md](../evaluation/protocols.md).
- **Optimizer:** AdamW + cosine + warmup. Batch size by VRAM.
  [MOMENT](../papers/moment.md) reports batch 2048, LR 1e-4→1e-5 as a
  reference.
- **Augmentation:** KernelSynth + TSMix from Chronos; value-flipping
  + resampling from Timer-S1. All four are cheap and compose.

## Hardware and compute reality check

Most TS-FM papers under-disclose training compute (see
[../research/reproducibility.md](../research/reproducibility.md) —
that column is almost all dashes). Order-of-magnitude intuitions
only, extrapolated from the papers that disclose something and from
standard transformer scaling:

- **Tiny (1–10M):** single consumer GPU (one RTX 4090 or equivalent),
  hours to a day for LOTSA-scale. [Mamba4Cast](../papers/mamba4cast.md)
  reports ~3 days on a single RTX 2080 Ti for its synthetic corpus;
  [TTM](../papers/ttm.md) is described as "resource-constrained,
  single GPU."
- **Small (10–50M):** 1× A100 80GB or H100, on the order of a day for
  LOTSA-scale at P=16, context 512. [Lag-Llama](../papers/lag-llama.md)
  famously trained on a single Tesla P100 12GB — one of the smallest
  reproducible footprints in the literature.
- **Base (50–200M):** 1× H100 in a few days, or 4× consumer GPUs.
  [Chronos](../papers/chronos.md) reports 200K optimizer steps at
  batch 256 on 8× A100 40GB per size, with per-size wall-clock in
  Appendix Table 6.
- **Large (200M–1B):** 4–8× datacenter GPUs, multi-day. No longer
  "small."
- **Huge (1B+):** industrial cluster. [Time-MoE](../papers/time-moe.md)
  and [Timer-S1](../papers/timer-s1.md) do not disclose GPU-hours;
  Timer-S1's 44 TB parquet footprint implies a ByteDance-scale cluster.

Do not invent specific hour counts. "On the order of days on an H100"
is fine; "34.2 GPU-hours" is not unless your own run measured it.

Inference: CPU is realistic under ~10M ([TTM](../papers/ttm.md)
explicitly measures ~10 ms per batch); consumer GPU through ~200M
(the Chronos family, [MOMENT](../papers/moment.md) small/base,
[MOIRAI](../papers/moirai.md) small/base); datacenter GPU above that.

## Licensing, leakage, and contamination

[LOTSA](../datasets-benchmarks/lotsa.md), the
[Time Series Pile](../datasets-benchmarks/time-series-pile.md), and
[Time-300B](../datasets-benchmarks/time-300b.md) all include subsets
of public benchmarks (Monash, ETT, ECL, Traffic, Weather, CloudOps,
and more). If you train on any of them and then evaluate on the same
benchmarks, **you are not zero-shot — you are in-corpus**. This is
the hazard flagged in [methodology-caveats.md](methodology-caveats.md)
section 1, where Moirai-MoE Figure 3 asterisks both TimesFM and
Chronos on Monash for exactly this reason.

Mitigations: (1) apply the [Timer-S1](../papers/timer-s1.md)
leakage-removal step — explicit deduplication against GIFT-Eval and
Chronos Benchmark II — before training, and document what you
removed; (2) for a clean zero-shot claim, train on LOTSA with
Timer-S1 curation and evaluate on **Chronos Benchmark II** (held out
by construction in the [Chronos](../papers/chronos.md) paper) and/or
**[GIFT-Eval](../datasets-benchmarks/gift-eval.md) with the leakage
filter applied**. Report Monash for historical continuity but label
it as "in-corpus against in-corpus competitors."

Licensing: each corpus is released under per-dataset licenses, not a
single monolithic license — read the HuggingFace cards before
redistribution.

## Where to cite your training setup

- LOTSA + masked encoder + any-variate attention →
  [MOIRAI](../papers/moirai.md).
- Quantization + cross-entropy + KernelSynth / TSMix →
  [Chronos](../papers/chronos.md).
- Masked reconstruction + patched encoder + Time Series Pile →
  [MOMENT](../papers/moment.md).
- Timer-S1 curation pipeline (imputation, k-sigma, ADF, leakage
  removal, value-flipping) and quantile-CRPS head →
  [Timer-S1](../papers/timer-s1.md).
- Student-t head + decoder-only probabilistic →
  [Lag-Llama](../papers/lag-llama.md).
- Flow-matching probabilistic head →
  [Sundial](../papers/sundial.md).
- TSMixer / MLP-Mixer sub-10M backbone → [TTM](../papers/ttm.md).
- Synthetic-only Prior-data Fitted Network training →
  [Mamba4Cast](../papers/mamba4cast.md).

## Related wiki pages

- [univariate-benchmarking.md](univariate-benchmarking.md) — the
  evaluation-side companion; pair reading.
- [leaderboard.md](leaderboard.md) — head-to-head tables across
  Monash, Chronos Benchmark II, GIFT-Eval, fev-bench, and LTSF.
- [state-of-the-art.md](state-of-the-art.md) — which model wins where
  by regime.
- [efficiency-and-cost.md](efficiency-and-cost.md) — parameter counts,
  inference latency, memory, CPU deployability.
- [methodology-caveats.md](methodology-caveats.md) — leakage,
  normalization, metric semantics. Read section 1 before claiming
  zero-shot.
- [../evaluation/metrics.md](../evaluation/metrics.md) — MASE, CRPS,
  WQL definitions and skill-score aggregation.
- [../evaluation/protocols.md](../evaluation/protocols.md) —
  zero-shot / in-domain / fine-tuned / full-shot definitions.
- [../datasets-benchmarks/lotsa.md](../datasets-benchmarks/lotsa.md),
  [../datasets-benchmarks/time-series-pile.md](../datasets-benchmarks/time-series-pile.md),
  [../datasets-benchmarks/time-300b.md](../datasets-benchmarks/time-300b.md),
  [../datasets-benchmarks/timebench.md](../datasets-benchmarks/timebench.md)
  — the four corpus-level pages.
- [../research/reproducibility.md](../research/reproducibility.md) —
  the per-paper table of weights, code, data, and disclosed compute.
- Relevant paper leaves:
  [../papers/moirai.md](../papers/moirai.md),
  [../papers/chronos.md](../papers/chronos.md),
  [../papers/moment.md](../papers/moment.md),
  [../papers/ttm.md](../papers/ttm.md),
  [../papers/lag-llama.md](../papers/lag-llama.md),
  [../papers/mamba4cast.md](../papers/mamba4cast.md),
  [../papers/timer-s1.md](../papers/timer-s1.md),
  [../papers/sundial.md](../papers/sundial.md),
  [../papers/time-moe.md](../papers/time-moe.md).
