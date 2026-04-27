# Knowledge Wiki — Time-Series Forecasting Foundation Models

This wiki is a hand-curated knowledge graph about time-series forecasting,
with a strong focus on the recent wave of *foundation models* (TS-FMs)
that learn a single pretrained model transferable across many datasets,
frequencies, and tasks. It is organized general-to-specific: high-level
orientation pages sit at the top, architecture families and cross-cutting
concepts in the middle, and individual research papers form the leaves.
Each page is short, links forward and backward, and ends with a list of
related nodes so that readers can move through the graph rather than
reading it linearly.

The wiki pairs with the `papers/` directory at the repo root, which holds
the actual PDFs. A typical traversal starts here, narrows down through
`foundation-models/taxonomy.md` to a cluster, clicks into an architecture
or concept page for context, and finally opens the corresponding leaf in
`papers/` to read the paper itself. Cross-links are always relative paths,
so the wiki is fully browsable on GitHub or offline.

## First stop at query time

- [index.md](index.md) — flat catalog of every wiki page with a
  one-line gloss; the schema-mandated first stop when answering a
  question.
- [log.md](log.md) — chronological append-only record of ingests,
  queries-filed-back, lint passes, refactors, and schema changes.

## Start here as a researcher

If you arrived here wanting to contribute — to the wiki or to the
field — read these first, in order:

- [research/reading-roadmap.md](research/reading-roadmap.md) — a
  beginner / intermediate / advanced path through the 26 TS-FM papers.
- [research/open-problems.md](research/open-problems.md) — the
  frontier questions as of 2026-04, with concrete directions.
- [research/comparison-matrix.md](research/comparison-matrix.md) — all
  26 TS-FMs (plus the JEPA cluster and pre-FM precursors) side by
  side, with cross-paper takeaways.
- [benchmarks/leaderboard.md](benchmarks/leaderboard.md) — current
  public numbers on the main evaluation suites.
- [evaluation/evaluation.md](evaluation/evaluation.md) — the methodology
  layer: metrics, baselines, protocols, and a per-paper summary of
  what each wiki paper actually evaluated.
- [benchmarks/univariate-benchmarking.md](benchmarks/univariate-benchmarking.md)
  — *"I want to benchmark my own model."* Practical recommendation
  for picking a comparison suite when your model is univariate-only
  (one series at a time, no covariates).
- [benchmarks/training-a-small-model.md](benchmarks/training-a-small-model.md)
  — *"I want to train my own TS-FM."* Practical recommendation for
  which corpus to pretrain on, which small TS-FMs set the competitive
  ceiling, and the 2026 data-curation recipe.
- [benchmarks/decision-guide.md](benchmarks/decision-guide.md) —
  *"I have to ship something on Monday."* The 1–3-model
  recommendation for each axis (univariate vs multivariate, point
  vs probabilistic, CPU vs big GPU, short vs long horizon).
- [research/timeline.md](research/timeline.md) — *"Who did what,
  when?"* Chronological walk through the 26 TS-FM papers, 2023 to
  2026, with the signature contribution of each.
- [research/training-recipes.md](research/training-recipes.md) and
  [research/failure-modes.md](research/failure-modes.md) — the
  consolidated training hyperparameters and documented weaknesses
  across the 26 papers, one table each.

The full research corner, including the reproducibility table,
contributing guide and glossary, is at
[research/research.md](research/research.md).

## Reading paths

- **15-minute overview.** Start with
  [foundations/time-series-forecasting.md](foundations/time-series-forecasting.md),
  then read [foundation-models/foundation-models.md](foundation-models/foundation-models.md), then
  skim [foundation-models/taxonomy.md](foundation-models/taxonomy.md).
- **Compare architectures.** Read
  [foundation-models/taxonomy.md](foundation-models/taxonomy.md) for the
  cluster map, then walk through
  [architectures/architectures.md](architectures/architectures.md) and the individual
  architecture pages it links to.
- **Pick a model for a use case.** Start from
  [foundation-models/taxonomy.md](foundation-models/taxonomy.md) to find
  the right cluster, consult the cross-cutting pages under
  [concepts/concepts.md](concepts/concepts.md) (zero-shot, probabilistic,
  [patch tokenization](./concepts/patch-tokenization.md)), then open the relevant leaves in
  [papers/papers.md](papers/papers.md).

## Top-level sections

- [foundations/](foundations/foundations.md) — the classical and deep-learning
  precursors that set the stage for TS-FMs.
- [foundation-models/](foundation-models/foundation-models.md) — the TS-FM paradigm,
  its brief history, and the eight-cluster taxonomy.
- [architectures/](architectures/architectures.md) — architecture families:
  decoder-only autoregressive, masked encoder, encoder-decoder, MoE,
  LLM reprogramming, lightweight non-transformer, continuous / flow
  matching, and JEPA / latent-target prediction.
- [concepts/](concepts/concepts.md) — cross-cutting technical ideas: patch
  tokenization, [value quantization](./concepts/value-quantization.md), zero-shot forecasting,
  probabilistic forecasting, [in-context learning](./concepts/in-context-learning.md),
  scaling laws, [RevIN](./concepts/revin-normalization.md), multi-task universal
  models, synthetic data, and the two non-reconstructive
  self-supervision families:
  [contrastive representation learning](./concepts/contrastive-representation-learning.md)
  and [JEPA](./concepts/joint-embedding-predictive-architecture.md).
- [datasets-benchmarks/](datasets-benchmarks/datasets-benchmarks.md) — the corpora and
  evaluation suites that TS-FMs are trained and judged on ([Monash](./datasets-benchmarks/monash-archive.md), [LOTSA](./datasets-benchmarks/lotsa.md),
  [Time-Series Pile](./datasets-benchmarks/time-series-pile.md), [Time-300B](./datasets-benchmarks/time-300b.md), [TimeBench](./datasets-benchmarks/timebench.md), [GIFT-Eval](./datasets-benchmarks/gift-eval.md)).
- [benchmarks/](benchmarks/benchmarks.md) — head-to-head performance
  comparisons across those suites: a normalized
  [leaderboard](benchmarks/leaderboard.md), a
  [state-of-the-art analysis](benchmarks/state-of-the-art.md) by regime,
  [methodology caveats](benchmarks/methodology-caveats.md), and an
  [efficiency and cost](benchmarks/efficiency-and-cost.md) page.
- [evaluation/](evaluation/evaluation.md) — the methodology layer: every
  metric ([MASE](./evaluation/metrics.md#17-mase--mean-absolute-scaled-error), [CRPS](./evaluation/metrics.md#21-crps--continuous-ranked-probability-score), [WQL](./evaluation/metrics.md#23-wql--weighted-quantile-loss), MSE, pinball...), seasonal-period and
  baseline conventions, evaluation protocols, a per-paper summary
  of what each wiki paper evaluated, and a comparability checklist.
- [papers/](papers/papers.md) — the 26 TS-FM leaf pages plus 1 pre-FM
  precursor (TiDE), one per paper.

## Knowledge-graph sketch

```
ts-forecasting-wiki
|
+-- foundations/
|   +-- time-series-forecasting
|   +-- classical-methods
|   +-- deep-learning-era
|
+-- foundation-models/
|   +-- taxonomy
|       +-- Cluster 1 decoder-only AR ---> TimesFM, Timer, Timer-XL,
|       |                                  Timer-S1, Lag-Llama, TimeGPT, Moirai-2
|       +-- Cluster 2 masked / enc-dec --> Chronos, Chronos-2, MOMENT, MOIRAI
|       +-- Cluster 3 mixture-of-experts-> Time-MoE, Moirai-MoE
|       |                                  (+ Timer-S1 secondary)
|       +-- Cluster 4 LLM reprogramming -> Time-LLM, GPT4TS, LLMTime
|       +-- Cluster 5 lightweight / SSM -> TTM, Mamba4Cast, SEMPO
|       |                                  (+ TSPulse secondary)
|       +-- Cluster 6 multi-task unified-> UniTS, TOTEM, TSPulse
|       |                                  (+ MOMENT, Timer-XL, Chronos-2 secondary)
|       +-- Cluster 7 continuous / flow -> Sundial
|       +-- Cluster 8 JEPA / latent --->   LaT-PFN, TS-JEPA, MTS-JEPA
|       +-- Pre-FM precursor ----------->  TiDE
|
+-- architectures/   (one page per family above; jepa-latent-prediction
|                     mirrors Cluster 8)
+-- concepts/        (patching, VQ, zero-shot, CRPS, ICL, scaling, RevIN,
|                     contrastive learning, JEPA, ...)
+-- datasets-benchmarks/ (Monash, LOTSA, TS-Pile, Time-300B, TimeBench, GIFT)
+-- papers/          (26 TS-FM leaves + 1 pre-FM precursor)
```

## Related wiki pages

- [foundation-models/taxonomy.md](foundation-models/taxonomy.md)
- [benchmarks/leaderboard.md](benchmarks/leaderboard.md)
- [benchmarks/state-of-the-art.md](benchmarks/state-of-the-art.md)
- [papers/papers.md](papers/papers.md)
