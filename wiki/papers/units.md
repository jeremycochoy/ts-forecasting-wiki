# UniTS: A Unified Multi-Task Time Series Model

> **Short name:** `units` · **arXiv:** [2403.00131](https://arxiv.org/abs/2403.00131) · **PDF:** [local](../../papers/units_2403.00131.pdf) · **Date:** 2024-02 · **Venue:** NeurIPS 2024

**Authors:** Shanghua Gao, Teddy Koker, Owen Queen, Thomas Hartvigsen, Theodoros Tsiligkaridis, Marinka Zitnik (Harvard / MIT)

## Abstract
UniTS unifies generative and predictive time-series tasks in a single model via a task-token interface. A modified transformer accepts variable-length inputs and uses learnable task tokens (GEN and CLS) to select between forecasting, classification, anomaly detection, and imputation, supporting strong few-shot transfer across a 38-dataset benchmark.

## Key contributions
- Task-token interface: a shared backbone is steered toward a given task by appending a learnable prompt token, a GEN token (for forecasting, imputation, anomaly detection), or a CLS token (for classification).
- Modified transformer block that factorises attention along the sequence and variable dimensions, handling heterogeneous (T, C) shapes without padding tricks.
- Unified treatment of generative and predictive tasks with one shared set of weights.
- 38-dataset benchmark spanning human activity, healthcare, engineering, and finance, evaluated against 12 forecasting, 20 classification, 18 anomaly detection, and 16 imputation baselines including adapted text LLMs.
- Prompt-learning mode where only the prompt tokens are tuned while the backbone is frozen, reaching few-shot transfer performance comparable to full fine-tuning.

## Architecture at a glance
UniTS wraps a transformer encoder whose blocks contain factorised sequence MHSA and variable MHSA with gated dynamic MLPs. Patched TS inputs are concatenated with learnable prompt tokens and task tokens; GEN tokens are replicated along the horizon direction and unpatchified at the output for direct multi-step forecasting, while CLS tokens are matched against class embeddings via distance matching for classification. The same backbone weights serve every task and dataset. Input is variable-length patched tokens; for forecasting, GEN task tokens determine the output format and the model produces the full forecast horizon in a single forward pass (direct, not autoregressive) (Gao et al. Section 3).

The **factorised attention over sequence and variable dimensions** (paper §4.2) is what lets one set of weights handle datasets with arbitrary numbers of channels and arbitrary lengths: a standard transformer's flat self-attention requires per-dataset reshaping (or padding to a global maximum) when channel count or sequence length varies, while factorising into separate sequence-MHSA and variable-MHSA passes keeps each attention call at a fixed shape per dataset. This is what makes the "one model across 38 datasets" claim architectural rather than a per-dataset fine-tuning trick.

### Sizes (Gao et al. Appendix D.1 + Table 14)

| Variant | UniTS blocks | d_model | d_ff | Heads | d_kv | Params | Patch | Context |
|---|---|---|---|---|---|---|---|---|
| UniTS-PMT ×32 (smallest) | 3 | 32 | not disclosed | not disclosed | — | 1.57M | 16 | dataset-dependent |
| UniTS-SUP ×64 / UniTS-PMT ×64 (default supervised) | 3 | 64 | not disclosed | not disclosed | — | 3.41M | 16 | dataset-dependent |
| UniTS-PMT ×96 | 3 | 96 | not disclosed | not disclosed | — | 5.67M | 16 | dataset-dependent |
| UniTS-PMT ×128 (default prompt-tuning) | 3 | 128 | not disclosed | not disclosed | — | 8.24M | 16 | dataset-dependent |

Each UniTS block contains time MHSA + variable MHSA + Dynamic FFN (first linear replaced by 3-kernel 1D conv). Heads count, `d_ff`, and `d_kv` are not tabulated in the paper — they must be read from the released config in `mims-harvard/UniTS`. Patch size and stride are fixed at 16 for both UniTS and patch-based baselines.

## Why it matters
UniTS advances the multi-task-universal direction by showing that a single backbone and task-token prompt can match specialist models across a broad benchmark. Its design makes it easy to add new tasks and datasets without changing the architecture, and it is the reference comparison point for any unified-model claim that is not based on masked reconstruction.

## Strengths
- Genuinely unified: a single set of weights handles forecasting, classification, anomaly detection, and imputation, versus MOMENT's task-specific heads on a masked-encoder backbone.
- Factorised attention over sequence and variable dimensions lets the model handle variable numbers of channels per dataset without per-dataset reshaping.
- Prompt-learning ablation shows that tuning only the prompt tokens (while freezing the backbone) recovers most of the multi-task performance, which is a strong sign that the backbone generalises.
- Reports 12.4% MSE improvement on imputation and 2.3% F1 improvement on anomaly detection in few-shot transfer to held-out datasets.
- Out-of-domain few-shot transfer is evaluated explicitly, not just in-domain interpolation.

## Limitations and open critiques
- The benchmark is heterogeneous but the per-task improvements over specialised SOTAs are incremental (forecasting MSE from ~0.4 to ~0.376); the main claim is versatility, not accuracy dominance.
- UniTS is evaluated mostly on the TSLib-style benchmark suite rather than on the large-scale zero-shot benchmarks used by Chronos, [Moirai](./moirai.md), or [TimesFM](./timesfm.md), so its position on the zero-shot leaderboard is unclear.
- No probabilistic forecasting head; task coverage is broad but uncertainty quantification is outside scope.
- The paper's "unification" is achieved by distinct heads (GEN tokens unpatchified, CLS tokens distance-matched), which is arguably a softer form of unification than a single generative objective.
- Scaling behaviour is not characterised — UniTS is evaluated at a single model size rather than across the Chinchilla-style grid that [Time-MoE](./time-moe.md) and [Sundial](./sundial.md) use.

## Follow-up work and dialogue
UniTS and [MOMENT](./moment.md) are the two canonical approaches to multi-task TS modelling: UniTS uses task tokens that steer a shared encoder toward each task, while MOMENT uses dedicated task heads on a masked-encoder backbone pretrained with the T5 masked-reconstruction objective. Later multi-task work has to choose between these two templates. [TOTEM](./totem.md) is an orthogonal entry in the same cluster, arguing that a learned VQ-VAE codebook should be the common substrate rather than raw patches. [Timer](./timer.md) and [Chronos](./chronos.md) are generative alternatives that unify by collapsing every task into next-token prediction on a discretised vocabulary, sidestepping the multi-head design entirely.

## Reproducibility
- **Open weights:** yes — the paper states code and datasets are released.
- **Code:** public repo stated as `https://github.com/mims-harvard/UniTS`.
- **Training data:** 38 public TS datasets across human activity, healthcare, engineering, finance, enumerated in the paper appendix; fully public.
- **Compute to retrain:** not disclosed in the main text; the paper emphasises single-GPU accessibility and does not report GPU-hours.
- **Deployment footprint:** parameter count and context length are reported per backbone variant in the appendix but not extracted here; the paper does not demonstrate CPU inference.

## When to cite this paper
Cite UniTS as the canonical reference for the task-token multi-task recipe in time series — a single shared backbone steered by learnable GEN/CLS tokens that handles forecasting, classification, anomaly detection, and imputation with one set of weights. It is the natural counterpart to MOMENT whenever the question is how to unify predictive and generative TS tasks.

## In the knowledge graph
- **Cluster:** [Multi-task / universal unified TS models](../foundation-models/taxonomy.md#cluster-6--multi-task--universal-unified-ts-models)
- **Architecture family:** [Masked encoder](../architectures/masked-encoder.md)
- **Related concepts:** [multi-task universal](../concepts/multi-task-universal.md), [patch tokenization](../concepts/patch-tokenization.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md)
- **See also:** [moment](./moment.md), [totem](./totem.md), [timer](./timer.md), [chronos](./chronos.md)
