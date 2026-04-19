# LOTSA

LOTSA (Large-scale Open Time Series Archive) is the pretraining corpus assembled by Salesforce for MOIRAI. It contains roughly 27 billion observations drawn from 9 broad domains, making it one of the largest publicly described open time-series corpora and a key enabler of frequency-aware masked-encoder pretraining.

## Overview

LOTSA was built specifically to support a single foundation model that can handle a wide range of frequencies, domains, and variate structures without per-domain specialization. The 9-domain coverage is chosen to span the kinds of dynamics TS models have to handle in production: energy, transport, climate/weather, sales, web metrics, and other operational-data categories. Series are available at multiple sampling frequencies, from sub-hourly to yearly, and the corpus is assembled as an open resource so that downstream work can reuse it.

MOIRAI uses LOTSA as its sole pretraining corpus and leverages the multi-frequency coverage by attaching multiple input projections keyed by patch size, so that the model learns distinct parameters for different frequency regimes. The masked-encoder objective, together with LOTSA's breadth, yields strong zero-shot numbers on [Monash](./monash-archive.md) and GIFT-Eval. Moirai-MoE then inherits LOTSA for pretraining but drops the explicit frequency projections in favor of token-level mixture-of-experts routing, arguing that the corpus is large and diverse enough to let the router discover frequency specialization on its own.

LOTSA's main contribution relative to older TS corpora is scale-per-domain: earlier benchmarks like Monash contain diverse datasets but only hundreds of millions of observations, and many published FM pretraining mixtures are opaque blends of private data. LOTSA is explicit, large, and reproducible.

## Key ideas / variants

- ~27 billion observations across 9 domains.
- Multi-frequency coverage from sub-hourly to yearly.
- Open and reproducible pretraining corpus.
- Used to train MOIRAI and Moirai-MoE.

## Wikipedia-derived entries

Per MOIRAI Table 14 (Appendix), LOTSA's Web domain includes four
Wikipedia-pageview datasets plus three CloudOps datasets. The
Wikipedia entries:

- `Kaggle Web Traffic Weekly` — 133,388 series / ~15M obs (Monash/Godahewa et al., 2021)
- `Extended Web Traffic` — 161,890 series / ~333M obs (Monash)
- `Wiki-Rolling` — 47,675 series / 40,619,100 obs (GluonTS/Alexandrov et al., 2020)
- `Wiki Daily (100k)` — 100,001 series / ~274M obs (sourced from [Chronos](../papers/chronos.md)/Ansari et al., 2024)

These are the same four Wikipedia-derived datasets that appear in
[Time-300B](time-300b.md), and the same three (ex-`Wiki Daily`) that
appear in [GIFT-Eval](gift-eval.md) Pretrain. GIFT-Eval **test**
contains zero Wikipedia-derived series (GIFT-Eval Table 13), so
LOTSA's Wikipedia content does not produce GIFT-Eval test leakage by
itself; the leakage concern with original LOTSA is via shared Monash /
M4 / ETT / Electricity entries that GIFT-Eval test does include —
which is precisely what `GIFT-Eval Pretrain` scrubs out. See
[../benchmarks/wikipedia-pageviews-leakage.md](../benchmarks/wikipedia-pageviews-leakage.md).

## Papers that exemplify this (or use this)

- [MOIRAI](../papers/moirai.md) — created LOTSA and used it as the sole pretraining corpus.
- [Moirai-MoE](../papers/moirai-moe.md) — reuses LOTSA with token-level sparse MoE routing.

## Related / derived corpora

- **Moirai-2 derivative mixture.** [Moirai 2.0](../papers/moirai-2.md)
  uses a 36M-series, ~295B-observation mixture that combines a
  non-leaking subset of LOTSA (`GIFT-Eval Pretrain`, 230B observations)
  with Chronos-Mixup (30M synthetic series), KernelSynth
  (1M / 1.02B points), and Salesforce-internal CloudOps telemetry
  (~2.15M series / ~1.48B daily observations, proprietary). The
  mixture is unnamed in the paper and not redistributable as a whole
  because of the CloudOps component; the Moirai-2 leaf documents the
  individual sources.

## Related wiki pages

- [Masked encoder](../architectures/masked-encoder.md)
- [Mixture-of-experts](../architectures/mixture-of-experts.md)
- [GIFT-Eval](gift-eval.md)
- [../benchmarks/training-a-small-model.md](../benchmarks/training-a-small-model.md)
  — see this page for how to use LOTSA as a default pretraining
  corpus for a small TS-FM, including the [Timer-S1](../papers/timer-s1.md) curation pipeline
  to apply on top of it.
