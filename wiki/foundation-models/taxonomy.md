# TS-FM Taxonomy — Seven Clusters

This page is the canonical taxonomy the rest of the wiki anchors into.
Each cluster has its own H2 heading (do not rename them — other pages
link directly to the emitted GitHub anchors shown in the section
index), a two-sentence description, a pointer to the primary
architecture page, and a bullet list of member papers with a one-line
hook each.

## Summary table

| Paper       | Year    | Cluster | Backbone                         | Pretraining                   | Params                      |
|-------------|---------|---------|----------------------------------|-------------------------------|-----------------------------|
| TimesFM     | 2023-10 | 1       | Decoder-only transformer         | Next-patch prediction         | ~200M                       |
| Timer       | 2024-02 | 1       | Decoder-only transformer         | Autoregressive next-token     | ~100M class                 |
| Timer-XL    | 2024-10 | 1 (+6)  | Decoder-only, long context       | Autoregressive, multivariate  | —                           |
| Lag-Llama   | 2023-10 | 1       | Decoder-only (Llama-style)       | Autoregressive w/ lag tokens  | small                       |
| TimeGPT-1   | 2023-10 | 1       | Transformer (closed)             | Autoregressive (undisclosed)  | —                           |
| Chronos     | 2024-03 | 2       | Encoder-decoder (T5)             | Next-token on quantized bins  | 20M – 710M                  |
| Chronos-2   | 2025-10 | 2 (+6)  | Encoder-decoder (T5), multivar.  | Next-token, multivariate      | 120M                        |
| MOMENT      | 2024-02 | 2 (+6)  | Masked encoder (T5-style)        | Masked reconstruction         | 40M / 125M / 385M           |
| MOIRAI      | 2024-02 | 2       | Masked encoder-decoder           | Masked reconstruction         | small / base / large        |
| Time-MoE    | 2024-09 | 3       | Decoder-only MoE transformer     | Autoregressive                | up to 2.4B (MoE)            |
| Moirai-MoE  | 2024-10 | 3       | MoE encoder-decoder              | Masked reconstruction         | —                           |
| Time-LLM    | 2023-10 | 4       | Frozen Llama + reprogramming     | Prompt + patch embedding      | Llama backbone              |
| GPT4TS      | 2023-02 | 4       | Frozen GPT-2 base (OFA)          | Light fine-tuning on TS       | GPT-2 base                  |
| LLMTime     | 2023-10 | 4       | Off-the-shelf GPT-3 / Llama-2    | None (text-as-number)         | vanilla LLM                 |
| TTM         | 2024-01 | 5       | TSMixer (MLP) backbone           | Masked reconstruction         | 1 – 5M                      |
| Mamba4Cast  | 2024-10 | 5       | Mamba (SSM)                      | Synthetic-only pretraining    | small SSM                   |
| UniTS       | 2024-03 | 6       | Unified transformer              | Multi-task joint training     | —                           |
| TOTEM       | 2024-02 | 6       | VQ tokenizer + transformer       | Multi-task on VQ tokens       | —                           |
| Sundial     | 2025-02 | 7       | Transformer + flow matching      | Flow matching on cont. tokens | ~1T pretraining points      |
| Timer-S1    | 2026-03 | 1 (+3)  | Decoder-only sparse MoE          | Serial-Token Prediction (STP) | 8.3B total / 0.75B active   |

## Cluster 1 — Decoder-only autoregressive TS-FMs

These models adopt the GPT-style recipe most directly: patch the
history, feed it to a decoder-only transformer, and predict the next
patch (or the next quantized value). They are the purest "LLM for time
series" bet and dominated the first wave of TS-FMs.

Primary architecture page:
[../architectures/decoder-only-autoregressive.md](../architectures/decoder-only-autoregressive.md).

- [../papers/timesfm.md](../papers/timesfm.md) — Google's patch-based
  decoder trained on a large multi-domain corpus; the prototypical
  TS-FM.
- [../papers/timer.md](../papers/timer.md) — Tsinghua's decoder-only
  model trained with generative pretraining across heterogeneous
  series.
- [../papers/timer-xl.md](../papers/timer-xl.md) — Timer's long-context,
  multivariate successor; doubles as a Cluster-6 unified model.
- [../papers/timer-s1.md](../papers/timer-s1.md) — 8.3B sparse-MoE
  decoder with Serial-Token Prediction, trained on [TimeBench](../datasets-benchmarks/timebench.md); state of
  the art on [GIFT-Eval](../datasets-benchmarks/gift-eval.md) among pre-trained models. Doubles as a Cluster-3
  MoE member.
- [../papers/lag-llama.md](../papers/lag-llama.md) — a small
  Llama-style architecture that explicitly feeds lag features as
  tokens.
- [../papers/timegpt.md](../papers/timegpt.md) — Nixtla's closed-source
  transformer API, historically important as the first commercial
  TS-FM offering.

## Cluster 2 — Masked-encoder / encoder-decoder TS-FMs

These models pretrain by masking parts of the input (random patches,
random spans, or a future block) and reconstructing them, reusing the
T5 / BERT playbook. Several of them naturally extend to classification
and imputation, which is why their members also appear as secondaries
in Cluster 6.

Primary architecture pages:
[../architectures/masked-encoder.md](../architectures/masked-encoder.md)
and [../architectures/encoder-decoder-t5.md](../architectures/encoder-decoder-t5.md).

- [../papers/chronos.md](../papers/chronos.md) — tokenize series by
  binning values, then train a T5 on the resulting sequences.
- [../papers/chronos-2.md](../papers/chronos-2.md) — adds first-class
  multivariate inputs and covariates while keeping the T5 backbone.
- [../papers/moment.md](../papers/moment.md) — masked encoder
  pretrained for forecasting, classification, imputation and anomaly
  detection jointly.
- [../papers/moirai.md](../papers/moirai.md) — Salesforce's
  any-frequency, any-variate masked transformer with patch-size
  adaptation.

## Cluster 3 — Mixture-of-experts TS-FMs

Scaling laws suggest that more parameters help, but training cost is
the practical bottleneck. These models adopt sparsely activated MoE
layers to scale total capacity while keeping per-token compute bounded.

Primary architecture page:
[../architectures/mixture-of-experts.md](../architectures/mixture-of-experts.md).

- [../papers/time-moe.md](../papers/time-moe.md) — decoder-only MoE
  scaled to 2.4B total parameters on a very large TS corpus.
- [../papers/moirai-moe.md](../papers/moirai-moe.md) — MoE variant of
  MOIRAI with experts that specialize across frequencies and domains.
- Secondary member: [../papers/timer-s1.md](../papers/timer-s1.md) —
  pushes sparse MoE (top-2 of 32 experts) to 8.3B total parameters in
  the Timer family.

## Cluster 4 — LLM-adapted / reprogramming approaches

Rather than training a transformer *on* time series, these methods
reuse an existing LLM as the core and adapt its interface. They are
cheap to build and benefit from the fluency of pretrained language
models, but depend on the LLM backbone's idiosyncrasies.

Primary architecture page:
[../architectures/llm-reprogramming.md](../architectures/llm-reprogramming.md).

- [../papers/time-llm.md](../papers/time-llm.md) — reprograms a frozen
  Llama by mapping patches to text-prototype tokens and prompting.
- [../papers/gpt4ts.md](../papers/gpt4ts.md) — the "one fits all" OFA
  paper: freeze most of GPT-2 and fine-tune only a few layers for TS
  tasks.
- [../papers/llmtime.md](../papers/llmtime.md) — feed the numbers as
  plain text to an off-the-shelf LLM and decode the completion;
  surprisingly competitive zero-shot.

## Cluster 5 — Lightweight / non-transformer FMs

Evidence from DLinear and TSMixer that simple architectures can match
transformers at forecasting has carried into the foundation-model era.
These models aim for the "foundation" property — broad transfer — while
remaining tiny and cheap to run.

Primary architecture page:
[../architectures/lightweight-non-transformer.md](../architectures/lightweight-non-transformer.md).

- [../papers/ttm.md](../papers/ttm.md) — IBM's Tiny Time Mixers:
  1 – 5M parameter MLP-Mixer-style model with strong zero-shot
  transfer.
- [../papers/mamba4cast.md](../papers/mamba4cast.md) — a Mamba SSM
  pretrained entirely on synthetic data, achieving zero-shot
  performance without touching real series.

## Cluster 6 — Multi-task / universal unified TS models

These models extend the TS-FM idea beyond forecasting to a unified
interface for forecasting, classification, imputation, and anomaly
detection. MOMENT, Timer-XL and Chronos-2 also belong here as
secondaries because their architectures and training setups were
designed to support multiple downstream tasks.

Primary architecture pages: see the masked-encoder and decoder-only
pages above; Cluster 6 is defined by task scope rather than backbone.

- [../papers/units.md](../papers/units.md) — a single transformer
  trained jointly across tasks with task-specific prompt tokens.
- [../papers/totem.md](../papers/totem.md) — VQ-VAE tokenizer plus
  transformer, yielding a discrete universal representation reusable
  across tasks.
- Secondary members: [../papers/moment.md](../papers/moment.md),
  [../papers/timer-xl.md](../papers/timer-xl.md),
  [../papers/chronos-2.md](../papers/chronos-2.md).

## Cluster 7 — Continuous / flow-matching FMs

The most recent branch: rather than quantizing values into a discrete
vocabulary, these models treat tokens as continuous vectors and use
flow-matching or diffusion-style objectives to train a generative
forecaster.

Primary architecture page:
[../architectures/flow-matching-continuous.md](../architectures/flow-matching-continuous.md).

- [../papers/sundial.md](../papers/sundial.md) — flow-matching on
  continuous patch tokens; pretrained on ~1T points and treats
  forecasting as continuous generative modeling.

## Related wiki pages

- [foundation-models.md](foundation-models.md)
- [../architectures/architectures.md](../architectures/architectures.md)
- [../papers/papers.md](../papers/papers.md)
- [../benchmarks/leaderboard.md](../benchmarks/leaderboard.md) —
  head-to-head performance of the models in this table on
  GIFT-Eval, fev-bench, Chronos Benchmark II, Monash and LTSF.
- [../benchmarks/state-of-the-art.md](../benchmarks/state-of-the-art.md)
  — narrative "which cluster wins for which regime."
