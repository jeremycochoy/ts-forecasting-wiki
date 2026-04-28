# Chronos-2: From Univariate to Universal Forecasting

> **Short name:** `chronos-2` · **arXiv:** [2510.15821](https://arxiv.org/abs/2510.15821) · **PDF:** [local](../../papers/chronos-2_2510.15821.pdf) · **Date:** 2025-10 · **Venue:** preprint

**Authors:** Abdul Fatir Ansari, Oleksandr Shchur, Jaris Küken et al. (AWS)

## Abstract
Chronos-2 generalizes the Chronos recipe from univariate next-token prediction to a universal forecaster that natively handles univariate, multivariate, and covariate-informed tasks. A 120M-parameter encoder-only patch-based backbone uses group attention to perform in-context learning over collections of related series.

## Key contributions
- Group attention mechanism enabling in-context learning across related series and panels within a single forward pass.
- Unified interface covering univariate, multivariate, past-only, known-future, and categorical covariate-informed forecasting tasks.
- Robust scaling with a sinh-inverse transform, patch embeddings, and a quantile multi-step decoder for direct probabilistic forecasts.
- Heavy synthetic-data pretraining via "multivariatizers" that impose cotemporaneous or sequential dependencies on univariate base generators (AR, ETS, TSI, KernelSynth).
- State-of-the-art results on fev-bench, GIFT-Eval, and Chronos Benchmark II.

## Architecture at a glance
Chronos-2 is a 120M-parameter encoder-only transformer following the T5 encoder design. Inputs are robustly normalized (standardization plus sinh-inverse), augmented with a time-index and mask meta feature, patched, and embedded via a residual network. Each block alternates between time attention (within a series across patches) and group attention (within a group across series at the same patch index); a group can be a multivariate variate-set, targets plus covariates, or a cross-learning batch of related series. A quantile head predicts multiple horizons simultaneously without autoregressive rollout. Memory scales O(V) in the number of variates rather than O(V^2). Input patch size is 16, output patch size is 16, with up to 64 output patches (1024 steps) predicted in a single forward pass via the quantile decoder — direct multi-step, no autoregressive rollout (HuggingFace `amazon/chronos-2` config).

Two design rationales sit at the centre of the architecture. **Sinh-inverse normalization** (paper §3.1, Eq. 1-2) provides log-like stabilization of outliers — inspired by econometrics and energy forecasting (Uniejewski & Weron 2018) — where standardization alone fails on strictly-positive or heavy-tailed series. **Group attention** (paper §3.2) preserves the inductive bias that related series share dynamics: by aggregating information *within* a group at each patch index rather than flattening all variates into one long sequence, it gives O(V) memory in the variate count (vs. O(V^2) for MOIRAI's any-variate flattening) while still letting the model disambiguate per-group via learned group assignments — a direct critique of [moirai](./moirai.md)'s scaling behaviour.

## Why it matters
Chronos-2 closes the gap between univariate TS foundation models and the multivariate, covariate-rich settings common in practice. By making related-series in-context learning a first-class mechanism, it unifies what used to require separate architectures or task-specific fine-tuning.

## Strengths
- On fev-bench the 120M Chronos-2 reaches 90.7% average win rate and 47.3% skill score vs. the TiRex/TimesFM-2.5/Toto cluster at 42-43% skill score, with the largest gap on covariate-informed tasks where existing pretrained models degrade to near-univariate baselines.
- The group attention design is O(V) in variates rather than O(V^2) as in Moirai-1.0 or COSMIC, and the paper demonstrates this empirically via inference scaling on a single A10G GPU at ~300 series/sec.
- Training uses synthetic "multivariatizers" (cotemporaneous and sequential) that impose diverse dependency structures on univariate base generators. The ablation Section 5.4 shows the synthetic-only variant already carries most of the multivariate gain, supporting the authors' claim that multivariate pretraining data, not architecture alone, is the bottleneck.
- Ablation compares a smaller model variant, synthetic-only training, and the pre-long-context checkpoint, isolating the contribution of each ingredient rather than reporting only the final number.
- Supports categorical covariates through target-encoding for univariate and ordinal encoding for multivariate targets, a practical feature that most prior pretrained TS models skipped.

## Limitations and open critiques
- Technical report, not peer reviewed; many claims rest on fev-bench, a benchmark co-authored by the same AWS team (Shchur et al. 2025). Independent replication is still pending.
- Multivariate and covariate pretraining data is almost entirely synthetic. There is no ablation demonstrating how well the TCM / multivariatizer priors cover real causal structures, so the gap between synthetic cross-variate dynamics and, e.g., real retail promotions remains an open question.
- The paper reports fev-bench "leakage %" for baselines (up to 28% for Moirai-2.0) as a data-hygiene concern but does not audit its own corpus with comparable rigor against Chronos Benchmark II; the authors acknowledge partial overlap with GIFT-Eval training splits.
- Abandons Chronos v1's categorical tokenization without a head-to-head ablation on identical data, so the value of quantization vs. continuous patches is not cleanly isolated inside this paper — see [../concepts/value-quantization.md](../concepts/value-quantization.md).
- Group size, long-context post-training, and model scaling beyond 120M are not reported; the paper is explicit that a size sweep is left to future work.

## Follow-up work and dialogue
Chronos-2 is the direct response to univariate limitations of [chronos](./chronos.md), [timesfm](./timesfm.md), [lag-llama](./lag-llama.md), [moment](./moment.md), and [timer](./timer.md). It positions itself against [moirai](./moirai.md) (which flattens multivariate inputs and pays O(V^2) memory), Toto (cross-variate attention without known covariates), COSMIC (univariate targets only), and [timer-xl](./timer-xl.md) (decoder-only any-to-any), arguing that the group attention + synthetic multivariatizer recipe dominates on fev-bench. On the covariate side, it supersedes [lag-llama](./lag-llama.md) and TabPFN-TS. See [../benchmarks/leaderboard.md](../benchmarks/leaderboard.md) for where Chronos-2 currently sits and [../research/open-problems.md](../research/open-problems.md) for the open questions on synthetic multivariate pretraining.

## Reproducibility
- **Open weights:** referenced via the `amazon-science/chronos-forecasting` repository; the technical report does not pin a specific HuggingFace revision.
- **Code:** public, same `amazon-science/chronos-forecasting` repository as Chronos v1.
- **Training data:** partially public — univariate subset reuses Chronos v1 and GIFT-Eval pretraining datasets; multivariate data is 100% synthetic via TSI, TCM, AR, ETS, KernelSynth, and multivariatizers, which is in principle regenerable from the released code.
- **Compute to retrain:** not disclosed in the extracted sections; GPU count, optimizer steps, and token count are not reported in the main text.
- **Deployment footprint:** 120M parameters; runs on a single NVIDIA A10G with ~300 series/sec throughput at context 2048, horizon 64; no CPU benchmark reported.

## When to cite this paper
Cite Chronos-2 as the canonical reference for group attention as a mechanism for in-context multivariate and covariate-informed forecasting in a pretrained TS model, and for the "multivariatizer" class of synthetic data augmentation. It is also the right citation for the first clear demonstration that a single pretrained model can match or beat univariate foundation models on covariate-informed fev-bench tasks without fine-tuning.

## In the knowledge graph
- **Cluster:** [Masked-encoder / encoder-decoder TS-FMs](../foundation-models/taxonomy.md#cluster-2--masked-encoder--encoder-decoder-ts-fms)
- **Architecture family:** [Masked encoder](../architectures/masked-encoder.md)
- **Related concepts:** [patch tokenization](../concepts/patch-tokenization.md), [in-context learning](../concepts/in-context-learning.md), [probabilistic forecasting](../concepts/probabilistic-forecasting.md), [multi-task universal](../concepts/multi-task-universal.md)
- **Dataset / corpus:** [GIFT-Eval](../datasets-benchmarks/gift-eval.md)
- **See also:** [chronos](./chronos.md), [moirai](./moirai.md), [timer-xl](./timer-xl.md)
