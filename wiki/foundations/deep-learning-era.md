# The Deep Learning Era

Between roughly 2017 and 2023, neural networks moved from curiosity to
default in the time-series forecasting literature. This page sketches
the main stepping stones and explains why, despite their successes,
none of them quite became *foundation* models in the sense the field now
uses the term.

## DeepAR (Amazon, 2017)

DeepAR trained an autoregressive recurrent network (LSTM / GRU) to
predict the parameters of a per-step output distribution (Gaussian,
negative binomial, Student-t) rather than a point value. Key ideas:
- Probabilistic output by construction, so confidence intervals come
  for free.
- Global training across many related series, with per-series
  embeddings — a precursor to the "one model, many series" mindset of
  later foundation models.
- Teacher-forcing at train time, ancestral sampling at inference.

DeepAR was the first widely deployed neural forecaster at industrial
scale (AWS SageMaker), and most later "global" models trace their
lineage to it.

## N-BEATS and N-HiTS (2019–2022)

N-BEATS stacked fully-connected residual blocks and learned
interpretable "basis expansion" decompositions (trend and seasonality
sub-networks). It beat statistical baselines on the M4 competition
*without* hand-crafted features, demonstrating that an MLP with the
right inductive bias could compete with ensembles of ETS and ARIMA.
N-HiTS later added hierarchical down-sampling for long horizons.

## Transformer adaptations (2020–2022)

As transformers took over NLP, several groups tried them on long-horizon
forecasting:
- **Informer** (AAAI 2021) — ProbSparse attention to scale the
  Transformer to thousands of steps.
- **Autoformer** (NeurIPS 2021) — decomposition-aware
  auto-correlation attention block.
- **FEDformer** (ICML 2022) — frequency-domain attention.

These papers pushed the state of the art on the canonical long-horizon
benchmarks (ETT, Weather, ECL, Traffic) but also invited a productive
backlash: *DLinear* (2022) showed that a simple linear model often
matched or beat them, casting doubt on whether the complexity of
attention was buying anything without the right tokenization.

## PatchTST (2022)

PatchTST resolved that tension. Its two simple ideas — **channel
independence** (one shared model applied per channel) and **patching**
(splitting the history into non-overlapping windows of length `P` that
become the tokens the Transformer consumes) — gave transformers a much
higher signal-per-token ratio and let them scale meaningfully. Patching
quickly became the default tokenization for subsequent TS-FMs such as
TimesFM, MOIRAI, MOMENT, and [Timer](../papers/timer.md).

## TSMixer and MLP-Mixer approaches (2023)

TSMixer applied the MLP-Mixer recipe from vision to time series,
alternating time-mixing and channel-mixing MLPs. Alongside DLinear and
N-BEATS, it is part of the ongoing evidence that transformer-scale
performance can be matched by far cheaper architectures when the data
are not enormous — a thread that reappears later in lightweight
foundation models like TTM (see
[../papers/ttm.md](../papers/ttm.md)).

## TiDE (2023)

[TiDE](../papers/tide.md) (Das, Kong, Leach, Mathur, Sen, Yu, TMLR 2023)
formalized the post-DLinear consensus: a pure-MLP encoder-decoder with a
covariate "feature projection" path, a per-step "temporal decoder" that
re-injects future covariates, and a global linear residual that
subsumes DLinear as a degenerate sub-model. On the seven canonical LTSF
benchmarks (ETT ×4, Weather, Traffic, Electricity) TiDE matches or beats
PatchTST while being 5–10× faster, and on the M5 retail competition it
beats DeepAR by 20% on WRMSSE thanks to its covariate-aware design.
TiDE is a per-dataset supervised model, not a foundation model — but
the same Google team (Das, Kong, Sen) carried channel independence,
covariate handling, and the linear residual into [TimesFM](../papers/timesfm.md)
the following year, so it sits directly on the lineage from DLinear to
the first TS-FM cohort.

## Contrastive representation learning (2018–2022)

A parallel track to the forecasting-first lineage above is
self-supervised representation learning, which trains an encoder to
discriminate "related" segments from "unrelated" ones rather than to
forecast or reconstruct. CPC (van den Oord et al., 2018) introduced
the InfoNCE recipe with a strided-CNN encoder and a GRU autoregressive
aggregator; T-Loss, TNC, TS-TCC, and especially TS2Vec (AAAI 2022)
adapted it for time series with hierarchical and timestamp-level
contrasts. These methods dominate the UCR/UEA classification
leaderboards and the Yahoo / KPI anomaly-detection benchmarks but were
overtaken on zero-shot forecasting by the 2024 TS-FM wave.
See [../concepts/contrastive-representation-learning.md](../concepts/contrastive-representation-learning.md)
for the full treatment.

## Why these fell short of being foundation models

- **Per-dataset training.** With the partial exception of DeepAR's
  global setup, these models were trained afresh for each benchmark,
  losing any hope of cross-domain transfer.
- **Limited cross-domain generalization.** Inductive biases tuned for
  ETT did not necessarily work on traffic or retail; hyperparameters
  had to be re-swept.
- **No zero-shot use.** Deployment required labels and compute for
  every new problem, exactly the cost that the NLP foundation-model
  playbook was eliminating elsewhere.

By late 2023 the ingredients were in place: patch tokenization, global
training, probabilistic heads, long-context attention, and — crucially —
the emergence of large multi-domain corpora such as the [Monash](../datasets-benchmarks/monash-archive.md) archive,
[LOTSA](../datasets-benchmarks/lotsa.md), and the [Time-Series Pile](../datasets-benchmarks/time-series-pile.md). The field was ready for the
TimesFM / Chronos / MOMENT / MOIRAI wave. See
[../foundation-models/foundation-models.md](../foundation-models/foundation-models.md) for
what came next, and
[../papers/timesfm.md](../papers/timesfm.md),
[../papers/chronos.md](../papers/chronos.md),
[../papers/moment.md](../papers/moment.md), and
[../papers/moirai.md](../papers/moirai.md) for the leaf pages of the
first TS-FM cohort.

## Related wiki pages

- [../foundation-models/foundation-models.md](../foundation-models/foundation-models.md)
- [classical-methods.md](classical-methods.md)
- [../concepts/patch-tokenization.md](../concepts/patch-tokenization.md)
