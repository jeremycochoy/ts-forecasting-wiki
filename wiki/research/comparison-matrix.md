# Comparison Matrix

This page puts the 20 papers side by side so that a researcher can see,
in one screen, which design choices have been tried and which
combinations are still empty. It is a map, not a ranking: the "Key
claim" column is the paper's own stated contribution, not a verdict on
it.

## How to read this matrix

- **Backbone family** is the architectural cluster
  ([../architectures/README.md](../architectures/README.md)) — decoder
  AR, masked encoder, encoder-decoder, MoE, LLM reprogramming,
  lightweight / SSM, flow matching.
- **Pretraining objective** is what the loss actually optimizes
  (next-patch point loss, next-token cross-entropy on a quantized vocab,
  masked reconstruction, flow matching, and so on).
- **Tokenization** is the input representation before the backbone
  sees it.
- **Param range** is the smallest to largest released variant.
- **Pretrain corpus** names the data; where the paper bundled a new
  corpus it is listed there.
- **Multivariate?** answers "can the model use multiple input channels
  natively, without per-channel re-training?"
- **Probabilistic?** answers "does the published model output a
  predictive distribution, not just a point?"
- **Open weights?** is Yes / No / Code only.
- **Key claim** is the paper's headline contribution in one phrase.

A dash ("—") means the paper does not state the value and it is not
general knowledge.

## The matrix

| Paper | Backbone family | Pretraining objective | Tokenization | Params | Pretrain corpus | Multivariate? | Probabilistic? | Open weights? | Key claim |
|---|---|---|---|---|---|---|---|---|---|
| [TimesFM](../papers/timesfm.md) | Decoder-only AR transformer | Next-patch point prediction, output patch > input patch | Continuous patches | ~200M | Google Trends + Wiki Pageviews + synthetic | No (univariate) | Point + quantile heads | Yes (HF) | Strong zero-shot from a clean decoder-only recipe |
| [Chronos](../papers/chronos.md) | Encoder-decoder (unmodified T5) | Next-token cross-entropy on quantized vocab | Value quantization to fixed vocab | 20M–710M | Public TS + TSMix + KernelSynth | No | Categorical sampling | Yes | Value quantization + vanilla T5 is enough |
| [Chronos-2](../papers/chronos-2.md) | Masked / encoder-only (patch-based) | Quantile regression with in-context group attention | Continuous patches | 120M | Heavy synthetic pretraining | Yes (group attention over related series) | Quantile decoder | Yes | In-context learning over related series beats scale |
| [MOMENT](../papers/moment.md) | Masked encoder (T5-init) + PatchTST | Masked reconstruction | Patches + RevIN | 40M / 125M / 385M | Time Series Pile | Channel-independent | Deterministic + quantile add-on | Yes (+ corpus) | One encoder for forecast / classify / impute / anomaly |
| [MOIRAI](../papers/moirai.md) | Masked encoder with any-variate attention | Masked reconstruction | Multi-patch-size projections | — base / large / small | LOTSA (~27B obs) | Yes (any-variate attention) | Mixture of Student-t | Yes | Any-variate attention + mixture head for true zero-shot |
| [Moirai-MoE](../papers/moirai-moe.md) | Masked encoder + token-level sparse MoE | Masked reconstruction with sparse routing | Unified single projection | — | LOTSA | Yes | Mixture-of-Student-t | Yes | Sparse MoE replaces multi-patch-size hack, 39-dataset SOTA |
| [Timer](../papers/timer.md) | Decoder-only AR (GPT-style) | Next-token autoregressive | S3 single-series format | ~1B data points scale, model mid-size | Unified S3 corpus | No | Point | Yes | Clean GPT-on-time-series recipe with strong few-shot |
| [Timer-XL](../papers/timer-xl.md) | Decoder-only AR, long-context | Next-token autoregressive | Flattened multivariate tokens (TimeAttention) | — | — | Yes (flattened) | Point | Yes | Universal TimeAttention handles multivariate + long context |
| [Lag-Llama](../papers/lag-llama.md) | Decoder-only AR probabilistic | Next-step NLL with Student-t head | Lag features | — | Assembled open corpus | No | Student-t head | Yes | First open probabilistic TS-FM with empirical scaling curves |
| [TimeGPT-1](../papers/timegpt.md) | Encoder-decoder (commercial) | Supervised forecast | — | Closed | Proprietary | Yes | Conformal prediction intervals | No (API) | Production-grade zero-shot via API |
| [Time-MoE](../papers/time-moe.md) | Decoder-only sparse MoE | Next-token autoregressive | Patches | Up to 2.4B (activated much smaller) | Time-300B | No (primary) | Point + quantile | Yes | Scaling a sparse MoE TS-FM to billion params on 300B tokens |
| [Time-LLM](../papers/time-llm.md) | LLM reprogramming (frozen Llama) | Reprogrammed forecast head with Prompt-as-Prefix | Patches aligned to LLM word embeddings | Llama-sized, adapter-small | — | Channel-independent | Point | Code only | Frozen LLM backbone can forecast with the right reprogramming |
| [GPT4TS / FPT](../papers/gpt4ts.md) | LLM reprogramming (frozen GPT-2/BERT/BEiT) | Task-specific forecast, fine-tuning only LN + pos | Patches | GPT-2 sized | — | Channel-independent | Point | Code only | Freezing almost everything of a pretrained LM still works for TS |
| [LLMTime](../papers/llmtime.md) | LLM reprogramming (vanilla GPT-3 / Llama-2) | None (pure zero-shot prompting) | Numbers as text | LLM-sized | None (zero-shot) | No | Discrete-to-continuous sampling | Code only | A plain LLM is already a zero-shot forecaster if numbers are tokens |
| [TTM](../papers/ttm.md) | Lightweight TSMixer (MLP-Mixer) | Supervised forecast | Adaptive patches + resolution prefix + exogenous head | 1–5M | Multi-source | Exogenous head only | Point + quantile add-on | Yes (HF) | Million-parameter CPU-deployable FM competitive with billion-param transformers |
| [UniTS](../papers/units.md) | Variable-length transformer with task tokens | Mixed generative + predictive | Task tokens + patches | — | 38-dataset multi-task | Yes | Point (+ task-specific) | Yes | One model for generative and predictive TS tasks across 38 datasets |
| [TOTEM](../papers/totem.md) | Cross-domain VQ-VAE + transformer | Reconstruction + downstream | VQ-VAE codebook | — | Cross-domain | Channel-independent | Point | Yes | A shared VQ codebook is a cross-domain tokenizer |
| [Sundial](../papers/sundial.md) | Continuous / flow matching (TimeFlow) | Flow matching over horizon | Continuous | — | TimeBench (~1T points) | — | Generative (flow sampling) | Yes | Flow matching as the generative head for TS-FMs at scale |
| [Mamba4Cast](../papers/mamba4cast.md) | SSM (Mamba) + PFN | Prior-fitted-network loss on synthetic priors | Continuous | Small / mid | Synthetic only | No | Yes (PFN predictive distribution) | Yes | Synthetic-only SSM backbone with single-pass horizon |
| [Timer-S1](../papers/timer-s1.md) | Decoder-only sparse MoE + TimeSTP blocks | Serial-Token Prediction (stacked shift-by-one heads) | Continuous patches (P=16) | 8.3B total / 0.75B active | TimeBench (~1T pts) curated + augmented | No (primary) | Yes (9-quantile CRPS head) | Yes (Tsinghua/ByteDance) | Serial Scaling in architecture, data, and training beats autoregressive rollout |

## Derived takeaways

- **Every billion-parameter TS-FM released so far is a sparse MoE.**
  Time-MoE and Moirai-MoE are the only two that clearly live above the
  billion-param mark; every dense open model is below 1B. That is a
  strong hint about what training budgets the dense recipes can afford.
- **Open-weights models are almost always masked-encoder or
  decoder-only.** Flow-matching (Sundial) has open weights but is
  alone in its family; the continuous generative direction is still
  one paper wide.
- **LLM reprogramming is code-only, not weight-only.** None of
  Time-LLM, GPT4TS or LLMTime ships a new pretrained artifact, because
  their contribution is a protocol on top of an existing LLM. This
  means "open code" does not give you something you can deploy
  off-the-shelf the way the HF checkpoints do.
- **Multivariate support is young and uneven.** Before Timer-XL,
  MOIRAI, Moirai-MoE and Chronos-2, every major TS-FM treated channels
  as independent. The multivariate recipes are also the ones that
  introduce the most idiosyncratic input interfaces, which is why
  cross-paper comparison is so hard in that regime.
- **Probabilistic output is no longer optional, but the head is not
  settled.** Categorical sampling, Student-t mixtures, quantile heads
  and flow matching all coexist, and different papers report different
  metrics for them.
- **Synthetic-only training is still a minority bet.** Mamba4Cast is
  the clearest pure example; Chronos's KernelSynth augments real data;
  every other model relies on real corpora. This is one of the
  highest-leverage unexplored combinations in the matrix.

## Related wiki pages

- [reading-roadmap.md](reading-roadmap.md)
- [open-problems.md](open-problems.md)
- [reproducibility.md](reproducibility.md)
- [../foundation-models/taxonomy.md](../foundation-models/taxonomy.md)
