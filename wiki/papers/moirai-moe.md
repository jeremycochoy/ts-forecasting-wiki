# Moirai-MoE: Empowering Time Series Foundation Models with Sparse Mixture of Experts

> **Short name:** `moirai-moe` · **arXiv:** [2410.10469](https://arxiv.org/abs/2410.10469) · **PDF:** [local](../../papers/moiraimoe_2410.10469.pdf) · **Date:** 2024-10 · **Venue:** preprint

**Authors:** Xu Liu, Juncheng Liu, Gerald Woo et al. (Salesforce AI Research)

## Abstract
Moirai-MoE replaces MOIRAI's frequency-specific input projection layers with a single unified projection and moves specialization into sparse mixture-of-experts feed-forward blocks inside the transformer. Expert routing happens at the token level rather than the frequency level, letting the model learn its own specialization structure. It also switches from a masked-encoder to a decoder-only training objective to enable parallel learning across context lengths.

## Key contributions
- Unified input projection replacing MOIRAI's hand-designed multi-patch-size frequency projections.
- Sparse MoE feed-forward layers with 32 experts and top-2 routing, delegating specialization to learned token-level assignment.
- A novel gating function seeded by cluster centroids extracted from a pretrained MOIRAI model (mini-batch k-means over pretrained self-attention outputs), which outperforms a randomly initialized linear gate under the same load-balancing loss.
- Decoder-only training with a causal normalizer and masking-ratio hyperparameter r for per-patch normalization without losing parallelism.
- Configurations: MOIRAI-MoE-Small at 11M activated / 117M total parameters and MOIRAI-MoE-Base at 86M activated / 935M total parameters.
- State-of-the-art in-distribution and zero-shot performance across 39 datasets.

## Architecture at a glance
Moirai-MoE keeps MOIRAI's any-variate flattening across variates but switches to a decoder-only causal backbone and injects sparse MoE feed-forward blocks with top-2 routing over 32 experts. Each token is routed to two experts, so total parameter count grows while per-token activated compute stays bounded. The frequency-specific input and output projection heads are removed and replaced with one shared residual-MLP projection.

## Why it matters
Moirai-MoE argues that specialization in TS foundation models is better handled by learned routing than by hand-designed frequency heuristics. It delivers stronger accuracy than dense MOIRAI at lower activated compute, making MoE a natural scaling axis for universal TS models.

## Strengths
- Direct critique of a specific prior design choice: the paper explicitly shows cases where time series at different frequencies share patterns (e.g. monthly vs daily seasonality) and cases where same-frequency series diverge, motivating token-level instead of frequency-level routing. Figure 1 gives concrete visual evidence.
- Reports up to 17% improvement over MOIRAI at the same size and up to 65x fewer activated parameters than dense baselines at comparable accuracy on 39 datasets. Comparisons are run against MOIRAI-Small/Base/Large, Chronos-Small/Base/Large, TimesFM and classical baselines.
- Introduces a new gating mechanism (cluster-centroid gate) and ablates it against pure linear gating plus load-balancing loss, showing the cluster-seeded gate is consistently superior across expert configurations — a non-trivial contribution to the MoE-for-time-series literature.
- Probes the inner workings: frequency-invariant representations and progressive denoising through the layer stack are demonstrated via expert-activation analysis, which is rare in TS-FM papers.
- Total parameter count grows to ~935M while activated compute stays at ~86M per token, giving a concrete recipe for scaling without the inference cost of dense MOIRAI-Large.

## Limitations and open critiques
- Still relies on LOTSA for pretraining; any LOTSA biases and leakage into downstream datasets carry over — see [../research/reproducibility.md](../research/reproducibility.md).
- Expert count (32) and top-k (2) are fixed by design choice; no ablation in the main text over expert count or sparsity ratio at multiple scales, so the MoE-specific scaling law for time series is still not mapped.
- Evaluation is dominated by normalized aggregate MAE across 39 datasets; probabilistic metrics ([CRPS](../evaluation/metrics.md#21-crps--continuous-ranked-probability-score), [WQL](../evaluation/metrics.md#23-wql--weighted-quantile-loss)) are reported less thoroughly than point metrics, which makes head-to-head probabilistic comparisons with [chronos](./chronos.md) or [lag-llama](./lag-llama.md) less clean.
- The cluster-centroid gate is seeded from a pretrained MOIRAI model, so MOIRAI-MoE inherits a dependency on a two-stage pretraining pipeline that a greenfield practitioner cannot reproduce in a single run.
- Switch to a decoder-only backbone is a substantial architectural change on top of the MoE story, and the ablation does not fully isolate how much of the reported gain is due to MoE vs. the training-objective change from masked-encoder to causal decoder.

## Follow-up work and dialogue
Moirai-MoE is a direct argument with [moirai](./moirai.md) (its own predecessor) and with [timesfm](./timesfm.md), which maintains a frequency embedding dictionary: both use frequency as a first-class input feature, and Moirai-MoE replaces that with token-level sparse routing. It sits in the same MoE-for-TS design space as [time-moe](./time-moe.md); both papers independently argue MoE is the right scaling axis for universal TS forecasters, though they differ on training objective and corpus ([time-moe](./time-moe.md) uses a different decoder-only recipe). On the corpus side, Moirai-MoE inherits LOTSA from MOIRAI, which puts it in dialogue with [chronos](./chronos.md) and [moment](./moment.md), which use other open corpora. See [../architectures/mixture-of-experts.md](../architectures/mixture-of-experts.md).

## Reproducibility
- **Open weights:** referenced as released via Salesforce's `uni2ts` pipeline; specific HuggingFace identifiers are not pinned in the extracted main text.
- **Code:** public — extends the `SalesforceAIResearch/uni2ts` library used for MOIRAI.
- **Training data:** fully public — LOTSA, same as MOIRAI.
- **Compute to retrain:** small model trained for 50,000 steps and base for 250,000 steps on LOTSA; exact GPU count and wall-clock time not disclosed in the main text.
- **Deployment footprint:** MOIRAI-MoE-Small 11M activated / 117M total; MOIRAI-MoE-Base 86M activated / 935M total. Inference speed is dominated by activated parameters but memory by total parameters, so deployment is more demanding than dense MOIRAI at equivalent activated budget.

## When to cite this paper
Cite Moirai-MoE as the canonical reference for token-level sparse mixture-of-experts as a replacement for frequency-based input specialization in time series foundation models, and for the cluster-centroid gating function. It is also the right citation for the first clear demonstration that MoE routing beats hand-designed frequency heads at equal or lower activated compute on LOTSA-scale pretraining.

## In the knowledge graph
- **Cluster:** [Mixture-of-experts TS-FMs](../foundation-models/taxonomy.md#cluster-3--mixture-of-experts-ts-fms)
- **Architecture family:** [Mixture of experts](../architectures/mixture-of-experts.md)
- **Related concepts:** [patch tokenization](../concepts/patch-tokenization.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md), [scaling laws](../concepts/scaling-laws.md)
- **Dataset / corpus:** [LOTSA](../datasets-benchmarks/lotsa.md)
- **See also:** [moirai](./moirai.md), [time-moe](./time-moe.md)
