# Tiny Time Mixers (TTMs): Fast Pre-trained Models for Zero/Few-Shot Multivariate Forecasting

> **Short name:** `ttm` · **arXiv:** [2401.03955](https://arxiv.org/abs/2401.03955) · **PDF:** [local](../../papers/ttm_2401.03955.pdf) · **Date:** 2024-01 · **Venue:** NeurIPS 2024

**Authors:** Vijay Ekambaram, Arindam Jati, Pankaj Dayama, Sumanta Mukherjee, Nam H. Nguyen, Wesley M. Gifford, Chandra Reddy, Jayant Kalagnanam (IBM Research)

## Abstract
Tiny Time Mixers are compact non-transformer foundation models based on the TSMixer (MLP-Mixer) backbone. Despite starting at only 1M parameters, they match or beat much larger transformer TS foundation models on zero-shot and few-shot multivariate forecasting while running on CPU.

## Key contributions
- Non-transformer TSMixer backbone chosen for CPU-friendly deployment; the quadratic attention cost is avoided entirely.
- Adaptive patching (AP): different backbone levels operate at different patch lengths to generalise across dataset resolutions.
- Diverse resolution sampling (DRS): data augmentation that resamples high-resolution series into lower-frequency versions to boost resolution diversity.
- Resolution prefix tuning (RPT): a small prefix token encodes the sampling frequency so the model can condition on it.
- Multi-level channel head and explicit exogenous-variable mixer for cross-channel and covariate fusion during fine-tuning.
- 1M-5M parameter models (TTM-Q, TTM-B, TTM-E, TTM-A) that beat much larger TS foundation models zero/few-shot on 11 evaluation datasets.

## Architecture at a glance
TTM stacks TSMixer blocks that alternate feature-mixing and time-mixing MLPs over patched inputs. The backbone is channel-independent during pretraining, followed by an optional channel-mixing decoder and exogenous-variable mixer during fine-tuning. Adaptive patching rearranges the patch/feature dimensions across levels, resolution prefix tuning injects a frequency token, and RevIN handles distribution shift.

## Why it matters
TTM makes the case that parameter count is not destiny for TS foundation models. By exploiting TS-specific inductive bias and multi-resolution training, a tiny MLP-Mixer serves as a practical, edge-deployable foundation model. Together with Mamba4Cast it anchors the argument that the transformer-by-default assumption inherited from NLP is not justified in TS.

## Strengths
- Reports 4-40% accuracy improvements over Moment, [GPT4TS](./gpt4ts.md), [Time-LLM](./time-llm.md), Moirai, [TimesFM](./timesfm.md), Chronos, and [Lag-Llama](./lag-llama.md) on zero/few-shot forecasting, at 1-5M parameters rather than 100M-7B.
- Figure 1 of the paper gives a clean Pareto plot: TTM-B is 87× to 7000× faster than Chronos-T and Chronos-B on CPU batch inference while being more accurate.
- Explicit exogenous-variable support, which most TS foundation models lack; channel mixing is reintroduced only at fine-tuning time to keep pretraining cheap.
- Ablations isolate the contribution of each design choice: DRS alone gives ~37% improvement on zero-shot, and adding data from 250M to 1B samples adds another ~6%, showing data diversity matters more than model capacity.
- Practical: pretrained on public data, runs on CPU-only hardware, and the paper explicitly targets resource-constrained deployment.

## Limitations and open critiques
- TTM's benchmarks are weighted toward the TSLib long-horizon datasets (ETT, Weather, Electricity, Traffic); whether the efficiency gap holds on broader, more diverse evaluations such as [GIFT-Eval](../datasets-benchmarks/gift-eval.md) is less established.
- The model is deterministic point regression; there is no native probabilistic head. [Chronos-2](./chronos-2.md) and [Sundial](./sundial.md) target probabilistic calibration directly and TTM does not engage with that axis.
- Channel-mixing is bolted on at fine-tune time, so true multivariate zero-shot (without a target-domain tuning pass) remains limited.
- "Beats" larger models is measured at matched protocol, but the large TS-FMs were trained on different (often disjoint) corpora, so the comparison partially reflects training-data choice, not just architecture.
- Lightweight MLP-Mixer capacity caps the asymptote: on very long contexts or highly non-stationary regimes the inductive bias may not match what a 1B-parameter transformer can memorise.

## Follow-up work and dialogue
TTM is the direct counterweight to [time-moe](./time-moe.md): billion-parameter MoE vs. million-parameter Mixer, representing the two ends of the efficiency-versus-capacity trade-off. The paper argues that billion-parameter MoE models are overkill when lightweight Mixer models match or beat their zero-shot performance, and provides the Pareto evidence. [Mamba4Cast](./mamba4cast.md) is a parallel argument in the same cluster — non-transformer, efficient, single-pass — although trained on synthetic priors rather than real data. TTM is now the reference baseline for any paper that wants to claim efficient TS foundation-model performance, and for any discussion of CPU-only deployment.

## Reproducibility
- **Open weights:** yes — the paper states research-use weights are available, and multiple Apache-licensed enterprise-use variants (TTM-Q, TTM-B, TTM-E, TTM-A) are released; exact hub URLs referenced in paper but not extracted.
- **Code:** public repo stated as part of IBM's granite-tsfm project; URL referenced in paper but not extracted.
- **Training data:** 1B samples drawn from public TS datasets (subset of the [Monash](../datasets-benchmarks/monash-archive.md) collection plus additional public corpora), fully public.
- **Compute to retrain:** not disclosed as GPU-hours, but the paper emphasises that pretraining is "resource-constrained" and runs on a single GPU; fine-tuning is cheap because only the decoder head updates.
- **Deployment footprint:** 1M (TTM-Q), 4M (TTM-E), 5M (TTM-A) parameters; CPU inference measured around 10 ms per batch in Figure 1; supports CPU-only deployment.

## When to cite this paper
Cite TTM as the canonical reference for a sub-10M-parameter TS foundation model that matches much larger transformer-based models on zero/few-shot multivariate forecasting, and as the primary evidence that TS-specific inductive bias (adaptive patching, diverse resolution sampling, channel mixing) substitutes for raw scale. It is the obligatory baseline for any efficiency-versus-capacity claim.

## In the knowledge graph
- **Cluster:** [Lightweight / non-transformer FMs](../foundation-models/taxonomy.md#cluster-5--lightweight--non-transformer-fms)
- **Architecture family:** [Lightweight non-transformer](../architectures/lightweight-non-transformer.md)
- **Related concepts:** [patch tokenization](../concepts/patch-tokenization.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md), [multi-task universal](../concepts/multi-task-universal.md), [revin normalization](../concepts/revin-normalization.md)
- **See also:** [mamba4cast](./mamba4cast.md), [moirai](./moirai.md), [time-moe](./time-moe.md), [chronos](./chronos.md)
