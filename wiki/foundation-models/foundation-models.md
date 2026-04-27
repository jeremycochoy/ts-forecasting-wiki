# Time-Series Foundation Models (TS-FMs)

## Definition

A **time-series foundation model** is a single pretrained model that
transfers — typically zero-shot or with minimal fine-tuning — across
heterogeneous time-series datasets, frequencies, and (often) tasks. The
emphasis is on the *transfer*: rather than training a new forecaster
for each problem, a TS-FM is trained once on a large, diverse corpus
and then applied off the shelf, in the same spirit as an LLM used for
NLP downstream tasks.

## Motivation

- **LLM precedent.** Large pretrained language models showed that a
  single model, given enough data and parameters, can handle a long
  tail of tasks without per-task training. The time-series community
  had every reason to ask whether the same recipe would work for
  numerical sequences.
- **Cost of per-dataset training.** Industrial forecasting problems
  routinely involve millions of distinct series. Training a bespoke
  neural model per series, or even per dataset, does not scale.
- **Zero-shot and few-shot demand.** Many real deployments have very
  little history — a newly launched product, a brand-new sensor, a
  patient on day one. A pretrained prior is worth more than any
  per-series model when the history is too short to fit one at all.
- **Universal evaluation.** Corpora like the [Monash](../datasets-benchmarks/monash-archive.md) archive, [LOTSA](../datasets-benchmarks/lotsa.md),
  the [Time-Series Pile](../datasets-benchmarks/time-series-pile.md), [Time-300B](../datasets-benchmarks/time-300b.md), [TimeBench](../datasets-benchmarks/timebench.md) and [GIFT-Eval](../datasets-benchmarks/gift-eval.md) exist
  precisely to measure cross-domain transfer, making the foundation
  model paradigm both tractable and falsifiable.

## A brief history (2023–2026)

- **Early 2024 cohort.** [TimesFM](../papers/timesfm.md) (Google), [Chronos](../papers/chronos.md) (Amazon), [MOMENT](../papers/moment.md)
  (CMU), [MOIRAI](../papers/moirai.md) (Salesforce) and [Timer](../papers/timer.md) (Tsinghua) appeared within a
  few months of each other. They established the main design axes —
  decoder-only autoregressive, value-quantized, masked reconstruction,
  and patched encoder-decoder — and set competitive zero-shot
  benchmarks.
- **MoE scaling (late 2024).** [Time-MoE](../papers/time-moe.md) and [Moirai-MoE](../papers/moirai-moe.md) pushed total
  parameter counts above a billion while keeping activated parameters
  manageable, echoing the mixture-of-experts move in NLP.
- **Long context and true multivariate (late 2024 to 2025).** [Timer-XL](../papers/timer-xl.md)
  and [Chronos-2](../papers/chronos-2.md) extended context lengths and added first-class
  multivariate conditioning, a significant step beyond the
  channel-independent defaults of the early cohort.
- **Continuous / generative (2025).** [Sundial](../papers/sundial.md) introduced flow-matching
  on continuous tokens, breaking with the value-quantization lineage
  and reframing forecasting as continuous generative modeling.
- **JEPA / latent-target prediction (2024–2026).** A parallel line of
  work — [LaT-PFN](../papers/lat-pfn.md), [TS-JEPA](../papers/ts-jepa.md),
  [MTS-JEPA](../papers/mts-jepa.md) — ports LeCun's joint-embedding
  predictive architecture to time series, replacing input-space
  reconstruction with latent-space prediction under an EMA target
  encoder. The objective specializes for representation quality and
  noise-robustness rather than raw forecast accuracy, and currently
  dominates UCR / UEA classification and multivariate anomaly
  prediction more than forecasting leaderboards.

Alongside these "forecast-first" models, a parallel thread repurposed
frozen LLMs for time series ([Time-LLM](../papers/time-llm.md), [GPT4TS](../papers/gpt4ts.md), [LLMTime](../papers/llmtime.md)), and a
lightweight thread ([TTM](../papers/ttm.md), [Mamba4Cast](../papers/mamba4cast.md)) showed that very small models can
still be "foundation" in the transfer sense.

## Key design axes

- **Tokenization.** Patch embeddings (TimesFM, MOIRAI, MOMENT,
  Timer), value quantization into a discrete vocabulary (Chronos,
  Chronos-2), text-as-token reprogramming (LLMTime, Time-LLM), and
  learned vector-quantized tokens ([TOTEM](../papers/totem.md), Sundial). See
  [../concepts/patch-tokenization.md](../concepts/patch-tokenization.md)
  and [../concepts/value-quantization.md](../concepts/value-quantization.md).
- **Pretraining objective.** Next-token prediction on quantized or
  continuous tokens, masked reconstruction, or flow-matching / score
  matching for continuous generative variants.
- **Backbone.** Decoder-only transformer, encoder-decoder transformer,
  masked encoder, mixture-of-experts transformer, MLP-Mixer,
  state-space model (Mamba), or a frozen LLM reused as the core.
- **Task scope.** Forecast-only (the majority) versus multi-task
  unified models that also do classification, imputation and anomaly
  detection from a single pretrained checkpoint ([UniTS](../papers/units.md), TOTEM, MOMENT,
  Timer-XL, Chronos-2).

## Navigating the rest of the wiki

- [taxonomy.md](taxonomy.md) — the canonical eight-cluster taxonomy,
  which every other page anchors into.
- [../architectures/architectures.md](../architectures/architectures.md) — one page
  per architecture family, each linked from its matching cluster in
  the taxonomy.
- [../concepts/concepts.md](../concepts/concepts.md) — cross-cutting
  technical ideas (patching, VQ, zero-shot, probabilistic,
  [in-context learning](../concepts/in-context-learning.md), scaling
  laws, [RevIN](../concepts/revin-normalization.md), multi-task,
  synthetic data, contrastive and JEPA-style self-supervision).
- [../papers/papers.md](../papers/papers.md) — the 26 TS-FM paper
  leaves plus 1 pre-FM precursor (TiDE).

## Related wiki pages

- [taxonomy.md](taxonomy.md)
- [../foundations/deep-learning-era.md](../foundations/deep-learning-era.md)
- [../architectures/architectures.md](../architectures/architectures.md)
