# Sundial: A Family of Highly Capable Time Series Foundation Models

> **Short name:** `sundial` · **arXiv:** [2502.00816](https://arxiv.org/abs/2502.00816) · **PDF:** [local](../../papers/sundial_2502.00816.pdf) · **Date:** 2025-02 · **Venue:** ICML 2025

**Authors:** Yong Liu, Guo Qin, Zhiyuan Shi, Zhi Chen, Caiyin Yang, Xiangdong Huang, Jianmin Wang, Mingsheng Long (Tsinghua)

## Abstract
Sundial is a family of native time-series foundation models built around TimeFlow, a flow-matching objective that predicts each next-patch distribution directly in continuous value space without discrete tokenisation. It is pretrained on TimeBench, a corpus of roughly one trillion points, and delivers probabilistic multi-horizon forecasts with fewer parameters than Time-MoE.

## Key contributions
- TimeFlow Loss: an autoregressive flow-matching objective that parameterises a velocity field over continuous value trajectories, so the model can sample probabilistic predictions from Gaussian noise via K-step push-forward rather than from a categorical distribution.
- Continuous (not discrete) patch tokenisation, avoiding the out-of-vocabulary and quantisation-artifact problems of Chronos/LLMTime.
- Multi-patch prediction during pretraining to reduce autoregressive inference steps, with shared lookback representations that enable fast repeated sampling for probabilistic forecasts.
- Modern Transformer adaptations that prior TS-FMs often skip: RoPE, Pre-LN, FlashAttention, KV-cache — shown in ablations to improve zero-shot accuracy and reduce memory by 14.8% and inference time by 43.6%.
- TimeBench pretraining corpus of ~1 trillion time points, mixing real-world and synthetic data.
- Reported roughly 4.7% average MSE reduction relative to Time-MoE with fewer parameters.

## Architecture at a glance
Sundial pairs a decoder-style transformer (RoPE, Pre-LN, FlashAttention, KV-cache) with a small flow-matching MLP head (FM-Net) that learns a velocity field conditioned on the transformer's per-position representation via AdaLN. Training samples a noised target along the conditional optimal-transport path between Gaussian noise and the ground-truth patch, and the network regresses the velocity. Inference starts from Gaussian noise and advances with a K-step uniform trajectory to produce each predicted patch; repeated noise samples give probabilistic forecasts and test-time calibration knobs (sample count, step count). Input patch size is 16, output is next-patch (16); forecasting is autoregressive rollout where each step predicts the next-patch distribution via TimeFlow flow-matching, and multiple noise samples yield sampled trajectories (Liu et al. Table 5).

Three design rationales are made explicit in the paper. (a) **Flow-matching over discrete tokens** (paper §3.1): any discretisation ([Chronos](./chronos.md), [LLMTime](./llmtime.md)) commits to a fixed vocabulary that fails on out-of-vocabulary trends and forces *mode collapse* on heterogeneous distributions, while continuous flow-matching learns a velocity field over real-valued space along the conditional optimal-transport path — preserving the full distributional shape without any parametric prior commitment. (b) **Multi-patch prediction during pretraining** (paper §3.2) reduces the number of autoregressive rollout steps needed at inference, with shared lookback representations that enable fast repeated noise sampling for probabilistic forecasts. (c) **RoPE / Pre-LN / FlashAttention / KV-cache** are inherited from modern LM stacks rather than the older absolute-position transformer pattern most TS-FMs use; the paper's ablation attributes 14.8% memory and 43.6% inference-time savings to this engineering and treats them as essential for trillion-token-scale training stability.

### Sizes (Liu et al. Tables 5 + 6)

| Variant | Layers (L) | d_model | d_ff | Heads | d_kv | Params | Patch (P) | Context (T) | Pred (F) | TimeFlow head (D_tf, L_tf) |
|---|---|---|---|---|---|---|---|---|---|---|
| Sundial-Small | 6 | 512 | 2048 | 8 | not disclosed | 32M | 16 | 2880 | {16, 720} | (512, 3) |
| Sundial-Base (`thuml/sundial-base-128m`) | 12 | 768 | 3072 | 12 | not disclosed | 128M | 16 | 2880 | {16, 720} | (768, 3) |
| Sundial-Large | 24 | 1024 | 4096 | 16 | not disclosed | 444M | 16 | 2880 | {16, 720} | (1024, 6) |

Backbone: decoder-only transformer with RoPE + Pre-LN + FlashAttention + KV-cache. Standard `d_ff = 4 × d_model`. `d_kv` is not split out; implied as `d_model / heads`. TimeFlow head (FM-Net) is a small MLP parameterizing the velocity field on the conditional optimal-transport path; conditioning is via AdaLN. Inference uses K-step push-forward with sampled Gaussian noise per call.

## Why it matters
Sundial is the leading exponent of the continuous, flow-matching branch of TS foundation models. It shows that avoiding value quantisation can both improve accuracy and reduce parameter requirements, and provides the clearest argument to date that parametric probabilistic heads (Gaussian, Student-t, quantile) limit the capacity of foundation models on heterogeneous TS distributions.

## Strengths
- Ablation directly compares TimeFlow against MSE and diffusion at matched model size and data: TimeFlow achieves 0.290 average MSE vs. 0.296 (MSE loss) and 0.314 (diffusion) on the six TSLib benchmarks, and also wins on [CRPS](../evaluation/metrics.md#21-crps--continuous-ranked-probability-score).
- Generative forecasting gives genuine probabilistic outputs with test-time calibration: more samples at inference monotonically tighten the confidence interval, more flow-matching steps monotonically improve point accuracy.
- Just-in-time inference — FlashAttention plus KV-cache reduces inference time by 43.6% in the ablation, enabling millisecond-scale zero-shot prediction.
- Explicit case that MSE-trained TS-FMs suffer mode collapse because predictive distributions across domains are highly divergent; TimeFlow avoids this by not committing to a prior parametric family.
- Engineering hygiene: the paper is explicit that RoPE and Pre-LN, routinely ignored in earlier TS-FM work, are essential for training stability at trillion-token scale.

## Limitations and open critiques
- Flow-matching inference is multi-step: each predicted patch requires K push-forward iterations (plus sample count for probabilistic outputs), so the paper's "just-in-time" claim trades off K against accuracy.
- TimeBench is dominated by real-world public corpora plus "mostly real-world" synthetic data, but the exact mixture and domain balance are not audited as carefully as Chronos's data card; whether the scaling claim rests on a few high-volume domains is unclear.
- Reported gains over Time-MoE are modest (~4.7% average MSE) and specific to the six TSLib benchmarks; performance on [GIFT-Eval](../datasets-benchmarks/gift-eval.md) and the Chronos zero-shot suite is less developed.
- FM-Net is a small MLP; the paper does not deeply study whether scaling the flow head proportionally to the backbone matters.
- The method requires a custom loss and sampling pipeline, making it less drop-in replaceable than a regression head; downstream consumers who want point forecasts must choose between median and mean statistics over samples.

## Follow-up work and dialogue
Sundial directly positions itself against [time-moe](./time-moe.md) (parametric point regression with MSE/quantile loss) and against [chronos](./chronos.md) / [llmtime](./llmtime.md) (discrete tokenisation with cross-entropy). Its core claim is that all discrete-tokenisation approaches throw away continuous structure that flow-matching preserves, and that parametric density heads (Gaussian, Student-t mixtures as in [moirai](./moirai.md)) limit probabilistic capacity. [Chronos-2](./chronos-2.md) is the 2025-era response in the discrete camp: continuous flow-matching versus discrete quantile decoder is the central showdown over "should probabilistic heads be parametric at all." [Timer-XL](./timer-xl.md) is a neighbouring decoder-only TS-FM that keeps the standard regression head, providing a natural ablation point.

## Reproducibility
- **Open weights:** yes — the paper states weights and code are released.
- **Code:** public repo stated as `https://github.com/thuml/Sundial`.
- **Training data:** TimeBench, ~1 trillion time points, a curated mix of real-world public TS and synthetic data; release is referenced in the paper.
- **Compute to retrain:** not disclosed as GPU-hours in the main text; ablation experiments run up to 30k iterations with context length 2880 and patch length 16.
- **Deployment footprint:** multiple family members (Sundial-Base and variants) sized below Time-MoE's activated parameter counts; RoPE, FlashAttention and KV-cache enable millisecond-scale zero-shot inference; no explicit CPU benchmark.

## When to cite this paper
Cite Sundial as the canonical reference for flow-matching as a generative objective for TS foundation models, for the argument that continuous tokenisation and non-parametric probabilistic heads beat both MSE regression and discrete categorical decoding, and for the TimeBench trillion-point pretraining corpus. It is the obligatory citation whenever the discussion is about probabilistic TS foundation models beyond quantile regression.

## In the knowledge graph
- **Cluster:** [Continuous / flow-matching FMs](../foundation-models/taxonomy.md#cluster-7--continuous--flow-matching-fms)
- **Architecture family:** [Flow matching continuous](../architectures/flow-matching-continuous.md)
- **Related concepts:** [probabilistic forecasting](../concepts/probabilistic-forecasting.md), [patch tokenization](../concepts/patch-tokenization.md), [scaling laws](../concepts/scaling-laws.md)
- **Dataset / corpus:** [TimeBench](../datasets-benchmarks/timebench.md)
- **See also:** [time-moe](./time-moe.md), [timer-xl](./timer-xl.md), [chronos](./chronos.md), [chronos-2](./chronos-2.md), [llmtime](./llmtime.md), [moirai](./moirai.md)
