# Time-MoE: Billion-Scale Time Series Foundation Models with Mixture of Experts

> **Short name:** `time-moe` · **arXiv:** [2409.16040](https://arxiv.org/abs/2409.16040) · **PDF:** [local](../../papers/timemoe_2409.16040.pdf) · **Date:** 2024-09 · **Venue:** ICLR 2025 Spotlight

**Authors:** Xiaoming Shi, Shiyu Wang, Yuqi Nie et al.

## Abstract
Time-MoE introduces the first billion-parameter-class time-series foundation model by combining a sparse mixture-of-experts decoder-only transformer with the large Time-300B pretraining corpus. It shows empirical neural scaling behavior for time series under MoE scaling.

## Key contributions
- First billion-parameter TS foundation model, with the largest variant (Time-MoE-ultra) at 2.4B total and ~1.1B activated parameters.
- Release of Time-300B, roughly 309B points across nine domains (energy, finance, healthcare, nature, sales, synthetic, transport, web, other).
- Sparse MoE decoder-only transformer with a shared expert plus top-K routed experts per token.
- Point-wise SwiGLU tokenization, RoPE, RMSNorm, and multi-resolution forecasting heads with horizons {1, 8, 32, 64}.
- Empirical demonstration that scaling laws for model size and training tokens transfer to time series.

## Architecture at a glance
Time-MoE is a decoder-only transformer whose feed-forward blocks are replaced with sparse mixture-of-experts layers. Each token is routed to a small subset of experts plus a shared expert, keeping activated parameters bounded even as total capacity grows. Training is autoregressive next-patch prediction on Time-300B with a Huber loss for outlier robustness and an auxiliary load-balancing term. Multiple forecasting heads of different horizons are optimized jointly and dynamically scheduled at inference.

## Why it matters
Time-MoE is the landmark result for scaling time-series foundation models beyond the hundred-million-parameter regime. It provides the clearest published evidence so far that neural scaling laws translate from language to time series when capacity is added via sparse MoE, and establishes a billion-parameter reference point the rest of the field now has to argue for or against.

## Strengths
- Demonstrates a roughly 20% and 24% zero-shot and in-distribution forecasting error reduction over prior dense baselines (TimesFM, Moirai, [Chronos](./chronos.md)) with comparable activated FLOPs, at six standard benchmarks.
- First convincing scaling-law curves for TS foundation models: at matched activated FLOPs, sparse MoE variants dominate dense variants across three scales (50M / 200M / 1.1B activated).
- Flexible horizon handling via multi-resolution heads avoids the fixed-horizon rigidity of Moment and the truncation issue of Timer.
- Time-MoE-ultra is engineered to run inference on consumer hardware with less than 8 GB VRAM despite the 2.4B-parameter capacity.

## Limitations and open critiques
- Time-300B is heavily skewed: the "Nature" domain alone contributes ~90% of observations, so the scaling-law evidence is partially a study of one domain's breadth rather than balanced coverage.
- Probabilistic forecasting is handled only via point regression with Huber loss; there is no calibrated predictive density, which Sundial and [Chronos-2](./chronos-2.md) target directly.
- TTM pushes back on the premise: 1-5M-parameter Mixer models match or beat larger TS foundation models zero-shot, suggesting that capacity alone is not what the benchmarks reward.
- MoE routing stability, expert collapse, and the sensitivity of results to the auxiliary-loss coefficient are acknowledged but not deeply characterized.
- Evaluated benchmarks (ETT, Weather, ECL, Traffic) are the TSLib suite whose leakage and saturation are a growing concern; performance on [GIFT-Eval](../datasets-benchmarks/gift-eval.md) and the Chronos zero-shot benchmark is reported less extensively.

## Follow-up work and dialogue
TTM explicitly frames itself against Time-MoE: if a 1-5M-parameter TSMixer can match much larger TS foundation models on zero/few-shot benchmarks, then the billion-parameter MoE path looks like overkill for practical deployment, and the accuracy-per-parameter ratio that matters for edge and CPU inference goes the other way. [Sundial](./sundial.md) reports a roughly 4.7% average MSE reduction relative to Time-MoE with fewer parameters, arguing the bottleneck is the parametric point-regression head, not capacity. [Moirai-MoE](./moirai-moe.md) applies similar sparse routing but motivates it as replacing the frequency-projection heuristic of [Moirai](./moirai.md), giving a second sparse-MoE data point the field can compare to. See also [timesfm](./timesfm.md) and [timer](./timer.md) for the dense decoder-only counterparts this paper scales past.

## Reproducibility
- **Open weights:** yes — the paper states Time-MoE models and the Time-300B corpus are open-sourced; hub location referenced in paper but URL not extracted.
- **Code:** public repo stated as `https://github.com/Time-MoE/Time-MoE`.
- **Training data:** Time-300B is released alongside the paper; it is a cleaned aggregation of public TS corpora.
- **Compute to retrain:** each model trained for 100,000 steps at batch size 1024 and maximum sequence length 4096 (~4M time points per iteration). Optimizer is AdamW with peak learning rate 1e-3, weight decay 0.1, `β1=0.9`, `β2=0.95`, auxiliary load-balancing loss coefficient 0.02, linear warmup for 10,000 steps then cosine annealing. Training runs on 128x NVIDIA A100 80GB GPUs with BF16 precision and multi-resolution output heads for horizons `{1, 8, 32, 64}` (Time-MoE, Appendix B). Wall-clock time and total GPU-hours are not disclosed. See [../research/training-recipes.md](../research/training-recipes.md).
- **Deployment footprint:** three variants (base 50M activated / 113M total, large 200M / 453M, ultra 1.1B / 2.4B); context length 4096; ultra runs inference within 8 GB VRAM and base/large are positioned for CPU inference.

## When to cite this paper
Cite Time-MoE as the canonical reference for (i) the first billion-parameter TS foundation model, (ii) evidence that neural scaling laws transfer to time series when capacity is added sparsely, and (iii) the Time-300B pretraining corpus. It is the natural opposing data point to TTM in any efficiency-versus-capacity discussion.

## In the knowledge graph
- **Cluster:** [Mixture-of-experts TS-FMs](../foundation-models/taxonomy.md#cluster-3--mixture-of-experts-ts-fms)
- **Architecture family:** [Mixture of experts](../architectures/mixture-of-experts.md)
- **Related concepts:** [scaling laws](../concepts/scaling-laws.md), [patch tokenization](../concepts/patch-tokenization.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md)
- **Dataset / corpus:** [Time-300B](../datasets-benchmarks/time-300b.md)
- **See also:** [moirai-moe](./moirai-moe.md), [timesfm](./timesfm.md), [timer](./timer.md), [ttm](./ttm.md), [sundial](./sundial.md)
