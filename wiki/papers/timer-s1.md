# Timer-S1: A Billion-Scale Time Series Foundation Model with Serial Scaling

> **Short name:** `timer-s1` · **arXiv:** [2603.04791](https://arxiv.org/abs/2603.04791) · **PDF:** [local](../../papers/timer-s1_2603.04791.pdf) · **Date:** 2026-03 · **Venue:** preprint

**Authors:** Yong Liu, Xingjian Su, Shiyu Wang et al. (Tsinghua University and ByteDance; corresponding author Mingsheng Long)

## Abstract
Timer-S1 is a Mixture-of-Experts time series foundation model with 8.3B total parameters, 0.75B activated per token, and an 11.5K context length. The paper frames its contribution as "Serial Scaling" along three axes — model architecture, dataset, and training pipeline. Architecturally it introduces Serial-Token Prediction (STP), a generic objective that replaces rolling next-token inference with a stack of TimeSTP blocks that directly emit multi-horizon predictions. The corpus side is TimeBench, a one-trillion-point curated and augmented corpus used previously by the Timer team for Sundial (Timer-3) and re-curated here. On the GIFT-Eval leaderboard Timer-S1 posts the best pre-trained-model [MASE](../evaluation/metrics.md#17-mase--mean-absolute-scaled-error) (0.693) and [CRPS](../evaluation/metrics.md#21-crps--continuous-ranked-probability-score) (0.485), a 7.6% / 13.2% reduction relative to Sundial.

## Key contributions
- **Serial-Token Prediction (STP).** A training objective and inference scheme where a stack of *H* "TimeSTP" blocks each attends back to the original lookback plus the previous block's hidden state and emits a shift-by-one prediction, producing multi-horizon outputs in one forward pass without autoregressive rollout. Ablations show STP beats next-token prediction (NTP) and multi-token prediction (MTP).
- **Sparse TimeMoE + TimeSTP architecture.** 24 dense TimeMoE blocks (E=32 experts, K=2 active per token, QK-Norm, RoPE, Pre-RMSNorm, load-balancing auxiliary loss) followed by 16 TimeSTP blocks reused at inference for progressive serial refinement.
- **TimeBench curation at one trillion points.** Detailed data pipeline: causal mean imputation, k-sigma/IQR outlier removal on shifting windows, ADF-based predictability filtering, GIFT-Eval leakage removal, and two augmentation primitives — resampling (down-sampling and Fourier interpolation to expose multiple resolutions) and value-flipping (multiplying by -1 to invert trends while preserving dependencies).
- **Post-training stage for TS foundation models.** Stage 1 is continued pre-training on GIFT-Eval Pretrain with a weighted STP loss (1/sqrt(j) decay on deeper blocks) to focus short-term accuracy; stage 2 is RoPE-based context extension from 2,880 to 11,520 tokens.
- **Quantile-CRPS forecasting head.** Nine quantiles (0.1-0.9) trained with weighted quantile loss approximating CRPS, which is what GIFT-Eval actually scores.
- **State-of-the-art GIFT-Eval results.** Best MASE and CRPS among pre-trained models, with the largest gains on medium- and long-term horizons where naive next-token models accumulate error.

## Architecture at a glance
Timer-S1 is patch-tokenized with P=16 and hidden D=1024. Input values are first instance-renormalized (Eq. 1-2), then split into `N = ⌈T/P⌉` patches with left-padding and a per-patch binary mask `m_i ∈ {0,1}^P` marking padded positions. Each patch is concatenated with its mask and embedded by a shared residual network `R^{2P} → R^D` (Eq. 3) — i.e. the token encoder is a 32→1024 residual MLP, not a linear projection, and is weight-tied to the patch-projection output head. No additive positional embedding is applied at the embedding stage; relative position is injected inside attention via RoPE (Eq. 5-6). Pretraining context is T=2880 (N=180); stage-2 post-training extends to T=11520 (N=720) by RoPE continuation. The backbone is a 24-block decoder-only sparse-MoE transformer with QK-Norm, Pre-RMSNorm and 32 experts with top-2 routing — yielding 8.3B total / 0.75B active parameters. On top of the main stack sit 16 TimeSTP blocks: each one re-reads the original lookback plus the outputs of the preceding TimeSTP block through a projection, runs its own internal TimeMoE module, and emits a one-step-shifted prediction, so the final stack covers `(H+1)*P = 272` future points per forward pass. Inference simply retains these blocks, avoiding the autoregressive rollout that causes error accumulation in Timer and Time-MoE.

## Why it matters
Timer-S1 is the first paper to systematically separate "architecture scaling," "data scaling," and "training-pipeline scaling" for a TS foundation model, and the first to push GIFT-Eval MASE below the [Chronos](./chronos.md)/[Moirai](./moirai.md)/Sundial frontier using an explicit serial-prediction head instead of rolling next-token generation. It is also the authoritative writeup of TimeBench's curation pipeline: Sundial introduced the name and broad composition, but Timer-S1 is where the filtering, augmentation and leakage-handling are spelled out in enough detail to be reproduced.

## Strengths
- Head-to-head ablations isolate the STP objective, the data augmentation, and the TimeBench pre-training, so the claimed gains are attributed rather than bundled.
- Sparse (top-2 of 32) MoE configuration demonstrates that aggressive expert sparsity works at the TS-FM scale, extending the [Moirai-MoE](./moirai-moe.md)/Time-MoE trajectory to nearly 10B total parameters.
- Serial-Token Prediction directly addresses the long-horizon error-accumulation problem that has been the standard critique of decoder-only TS-FMs since Timer and [TimesFM](./timesfm.md).
- Training infrastructure is described in practical detail (BF16 on VeOmni, 44 TB of Parquet shards with hybrid memory-disk loading), making the engineering reproducible in principle.
- The quantile/CRPS head aligns the training loss with the benchmark's probabilistic metric instead of optimising MSE and reporting CRPS after the fact.

## Limitations and open critiques
- At 8.3B total parameters Timer-S1 is the largest TS-FM in the Timer family and retraining it from scratch is out of reach for most academic labs; the reported numbers are effectively un-ablated at non-ByteDance scale.
- TimeBench's exact domain mix and per-source weighting are still not published in auditable form despite the more detailed curation description; whether gains derive from a few high-volume domains remains opaque.
- The TimeSTP stack adds 16 blocks of inference compute per multi-horizon forecast; the paper frames this as cheaper than autoregressive rollout but does not report wall-clock comparisons at matched horizon length.
- Weights and code are announced but not released at submission time, so the GIFT-Eval numbers cannot yet be independently verified.
- The value-flipping augmentation implicitly assumes sign-symmetry of the underlying dynamics, which is violated for strictly non-negative series (counts, prices on log scale, utility demand); the paper does not discuss when to disable it.

## Follow-up work and dialogue
Timer-S1 is the direct successor to [timer](./timer.md), [timer-xl](./timer-xl.md) and [sundial](./sundial.md) (which the paper itself calls "Timer-3") within the THUML Timer lineage. Its strongest comparison is against Sundial: same corpus, same team, but STP-with-quantile-CRPS replaces TimeFlow flow-matching, and the paper argues STP wins on GIFT-Eval where Sundial's flow-matching head was benchmarked only on TSLib. Against [time-moe](./time-moe.md) it keeps the MoE decoder-only recipe but pushes expert sparsity further (top-2 of 32 vs top-2 of 8) and replaces next-token with STP. Against [chronos-2](./chronos-2.md) it is the decoder-only autoregressive camp's 2026 answer to the encoder-decoder/quantile-decoder camp.

## Reproducibility
- **Open weights:** announced ("will be released"), not yet available at the time of the preprint.
- **Code:** not yet released; training uses ByteDance's VeOmni framework.
- **Training data:** TimeBench, ~1T points, with the curation and augmentation pipeline documented in the paper; public dataset snapshots from Chronos and [LOTSA](../datasets-benchmarks/lotsa.md) are reused, but the full TimeBench mixture has not been released.
- **Compute to retrain:** not disclosed as GPU-hours; BF16 training on 44 TB of sharded Parquet suggests an industrial-scale cluster.
- **Benchmark scope:** GIFT-Eval (primary), with MASE 0.693 and CRPS 0.485 as pre-trained-model state of the art.

## When to cite this paper
Cite Timer-S1 (a) as the canonical reference for Serial-Token Prediction as an alternative to rolling next-token decoding in TS foundation models, (b) as the authoritative description of the TimeBench curation and augmentation pipeline (Sundial introduced the name; Timer-S1 documents how it is actually built), (c) for state-of-the-art pre-trained-model results on GIFT-Eval as of 2026, and (d) whenever the discussion concerns scaling TS-FMs to ~10B parameters with sparse MoE.

## In the knowledge graph
- **Cluster:** [Decoder-only autoregressive TS-FMs](../foundation-models/taxonomy.md#cluster-1--decoder-only-autoregressive-ts-fms) with strong secondary membership in [Mixture-of-experts TS-FMs](../foundation-models/taxonomy.md#cluster-3--mixture-of-experts-ts-fms)
- **Architecture family:** [Decoder-only autoregressive](../architectures/decoder-only-autoregressive.md), [Mixture-of-experts](../architectures/mixture-of-experts.md)
- **Related concepts:** [patch tokenization](../concepts/patch-tokenization.md), [scaling laws](../concepts/scaling-laws.md), [synthetic data augmentation](../concepts/synthetic-data-augmentation.md), [probabilistic forecasting](../concepts/probabilistic-forecasting.md)
- **Dataset / corpus:** [TimeBench](../datasets-benchmarks/timebench.md), [GIFT-Eval](../datasets-benchmarks/gift-eval.md)
- **See also:** [timer](./timer.md), [timer-xl](./timer-xl.md), [sundial](./sundial.md), [time-moe](./time-moe.md), [chronos-2](./chronos-2.md)
