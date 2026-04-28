# TS-JEPA: Joint Embeddings Go Temporal

> **Short name:** `ts-jepa` · **arXiv:** [2509.25449](https://arxiv.org/abs/2509.25449) · **PDF:** [local](../../papers/ts-jepa_2509.25449.pdf) · **Date:** 2025-09 · **Venue:** NeurIPS 2024 Workshop on Time Series in the Age of Large Models

**Authors:** Sofiane Ennadir (KTH Stockholm), Siavash Golkar (NYU), Leopoldo Sarra (Flatiron Institute)

## Abstract
TS-JEPA is the first systematic adaptation of LeCun's Joint-Embedding Predictive Architecture to time-series representation learning. The model splits a series into non-overlapping patches, masks a high fraction (70%), encodes the visible patches with a transformer, and trains a predictor to forecast the *latent embeddings* of the masked patches as produced by an EMA target encoder. Frozen-encoder evaluation shows TS-JEPA matches or beats Masked-Auto-Encoder (input-space MAE), TS2Vec (contrastive), and autoregressive baselines on classification, while staying competitive with autoregressive on long-horizon forecasting.

## Key contributions
- Establishes the four-component JEPA recipe — tokenizer, online encoder, predictor, EMA target encoder — as a clean baseline for time-series self-supervision, parallel to I-JEPA for images and V-JEPA for video.
- Direct head-to-head against MAE (same architecture, same masking ratio modulo small offset 70 vs 75), TS2Vec, and an autoregressive baseline; all four share the encoder backbone so the only variable is the pretraining objective.
- Demonstrates that JEPA's "latent reconstruction" beats input-space reconstruction (MAE) on noisy classification benchmarks like FaultDetectionA/B and ECG5000, supporting the paper's claim that input-space objectives over-fit to confounding noise.
- Shows TS-JEPA's pretrained encoder retains 75–90% of fully supervised classification accuracy with only 5–20% of labels (paper Figure 2), evidencing strong sample efficiency.

## Architecture at a glance
TS-JEPA divides each univariate window of length `T` into 10 non-overlapping patches; a 1-D CNN tokenizer plus sin-cos absolute positional encoding produces patch tokens. A uniform 70% mask drops `P_M` patches; the online encoder `E_θ` (transformer with 2 attention heads, embedding dim 128) processes only the visible `P_N` patches, then the predictor `P_β` (same architecture) emits predicted latents `z'_M` for the masked positions. The EMA target encoder `E_ξ` encodes the masked patches `P_M` directly, producing targets `t_M`. The training loss is the L1 distance `||z'_M − t_M||_1` averaged over masked positions. AdamW optimizer, learning rate tuned per dataset in `{1e-3 … 1e-6}`. All evaluation is frozen-encoder + small classification or regression head. Long-horizon forecasting uses autoregressive rollout of single-patch predictions.

### How TS-JEPA prevents representation collapse
Predicting your own encoder's outputs has a trivial degenerate solution: drive every embedding to the same constant vector and the loss is identically zero. The paper devotes paper §2(iv) and Appendix §1 ("On the EMA-Encoder") to this concern. Two design choices, inherited verbatim from the BYOL / I-JEPA lineage, block the collapse:

- **The target encoder `E_ξ` is not trained by backprop.** Its weights are an exponential moving average of the online encoder's weights, `E_ξ ← m · E_ξ + (1 − m) · E_θ`. The target is therefore a slowly-moving fixed point that the online branch chases, rather than a co-adapting partner that could race to the constant solution with it.
- **EMA decay `m = 0.998`** (paper Appendix §1). This is the same value used by I-JEPA and V-JEPA. The paper notes the choice is *empirical* — they observed it "give[s] good results while avoid[ing] the collapse" — and explicitly defers the ablation of `m` to future work, alongside the ablations of the 70% masking ratio, the L1 (vs L2) loss, and the predictor depth.

The paper's own collapse-prevention citations are LeCun's JEPA position paper [12] and BYOL [Grill et al. 2020, paper ref 10]; the appendix additionally cites I-JEPA [1] and V-JEPA [2] as validation that the EMA-target recipe transfers across modalities. TS-JEPA does not add any auxiliary anti-collapse machinery on top — no codebook bottleneck (contrast [MTS-JEPA](./mts-jepa.md), which adds a soft-codebook simplex with an analytical non-collapse bound), no contrastive negatives (contrast [TS2Vec](../concepts/contrastive-representation-learning.md)), no variance/covariance regularizers (contrast VICReg). The "two encoders, one EMA-tracked" trick is the entire stabilizer.

## Why it matters
Until TS-JEPA, the time-series JEPA literature consisted of one application paper (Time-Series JEPA for predictive remote control, Girgis et al. 2024) and one combined work ([LaT-PFN](./lat-pfn.md), 2024), neither of which isolates the JEPA objective from other architectural changes. TS-JEPA is the first systematic study, with controlled baselines that share the encoder and only vary the pretraining objective — this is the cleanest evidence in the literature that "predict in latent space, not in input space" carries over from I-JEPA to TS at all. The paper closes by positioning TS-JEPA as a building block for future TS-FMs, which [MTS-JEPA](./mts-jepa.md) then takes up.

## Strengths
- Clean, controlled comparison: same encoder dimensions across MAE, TS2Vec, autoregressive, and TS-JEPA, so claimed gains are attributable to the pretraining objective rather than capacity.
- Best classification accuracy on FordA (91.5%), FordB (73.8%), and competitive on FaultDetectionA (85.8%) and ECG5000 (89.5%) — consistently within 2 points of fully supervised transformers (paper Table 1).
- Wins on 2 of 3 long-horizon forecasting datasets (ETT-Small, Electricity) over the autoregressive baseline despite using simple patch-rollout (paper Figure 3); demonstrates that JEPA's latent-space objective doesn't sacrifice long-horizon stability.
- Sample-efficiency curves (paper Figure 2) cover labeled-data fractions from 5% to 20%, showing TS-JEPA dominates the supervised transformer in the low-label regime.
- The paper is short and surgical — one figure, one objective, no extra moving parts — which makes it a good reference template for how to ablate a TS pretraining objective.

## Limitations and open critiques
- Tiny model and tiny experiment: 2 attention heads, embedding dim 128, 10 patches per series, 5 datasets total. No scaling study; no claim about how the recipe behaves at the [Time-MoE](./time-moe.md) / [Timer-S1](./timer-s1.md) parameter brackets.
- Loses to autoregressive on short-term forecasting (next-patch) on all three forecasting datasets, which is the regime most TS-FMs optimize for; the paper acknowledges this and frames TS-JEPA as a *balanced* representation rather than a forecasting specialist.
- No probabilistic output. The predictor emits a point estimate of the masked latent; no quantile head, no CRPS, no calibration analysis.
- No GIFT-Eval, no Monash, no Chronos Benchmark II — same evaluation gap as [LaT-PFN](./lat-pfn.md), so the model is hard to place on the cross-paper leaderboards in [benchmarks/leaderboard.md](../benchmarks/leaderboard.md).
- No ablation of the 70% masking ratio, the L1 loss vs L2, the EMA decay `m=0.998`, or the depth/width of the predictor. The paper labels these as future work.
- Single architecture for encoder and predictor; no investigation of asymmetric capacities (e.g., a small predictor with a large encoder, which is the I-JEPA pattern).

## Follow-up work and dialogue
TS-JEPA is the bridge between [LaT-PFN](./lat-pfn.md) (PFN+JEPA hybrid for forecasting) and [MTS-JEPA](./mts-jepa.md) (JEPA + multi-resolution + codebook for anomaly prediction). The MTS-JEPA paper directly cites TS-JEPA and uses it as the single-resolution baseline that its multi-resolution architecture is supposed to improve on; that comparison is included in MTS-JEPA's main table (TS-JEPA F1 25.5–71.95% vs MTS-JEPA 33.6–72.9% across MSL/SMAP/SWaT/PSM, paper Table 1). On the broader self-supervised side, TS-JEPA's encoder/predictor/EMA-target setup is the same template used by [MOMENT](./moment.md) (input-space target) and [TSPulse](./tspulse.md) (input-space + FFT target); the JEPA-vs-masked-input comparison is sharpened in this paper but not yet at billion-parameter scale.

## Reproducibility
- **Open weights:** not advertised in the paper; the workshop paper does not commit to a public checkpoint.
- **Code:** [github.com/Sennadir/TS_JEPA](https://github.com/Sennadir/TS_JEPA) (referenced in paper Appendix §2).
- **Training data:** fully public — FordA, FordB, FaultDetectionA, FaultDetectionB, ECG5000 (UCR-style classification); Weather, ETT-Small, Electricity (forecasting).
- **Compute to retrain:** single NVIDIA V100 GPU; specific GPU-hours not tabulated. AdamW; LR ∈ {1e-3, 1e-4, 1e-5, 1e-6} grid-searched.
- **Deployment footprint:** transformer with 2 heads, embedding dim 128, 10 patches per window — well under 1M parameters in the encoder. Trivially CPU-deployable.

## When to cite this paper
Cite TS-JEPA as the first systematic study of JEPA for time series, as the canonical reference for the four-component (tokenizer + encoder + predictor + EMA target) JEPA-for-TS template, and for the controlled MAE-vs-JEPA-vs-contrastive-vs-autoregressive comparison at matched encoder capacity. It is also the right citation when arguing that JEPA's latent-space objective is more robust to input noise than MAE on classification benchmarks.

## In the knowledge graph
- **Cluster:** [Cluster 8 — JEPA / latent-space prediction](../foundation-models/taxonomy.md#cluster-8--jepa--latent-space-prediction)
- **Architecture family:** [JEPA / latent-target prediction](../architectures/jepa-latent-prediction.md)
- **Related concepts:** [joint-embedding predictive architecture](../concepts/joint-embedding-predictive-architecture.md), [patch tokenization](../concepts/patch-tokenization.md), [contrastive representation learning](../concepts/contrastive-representation-learning.md) (head-to-head baseline)
- **Architecture comparisons:** [masked-encoder](../architectures/masked-encoder.md) (input-space MAE counter-method)
- **See also:** [LaT-PFN](./lat-pfn.md) (predecessor), [MTS-JEPA](./mts-jepa.md) (successor with multi-resolution + codebook), [MOMENT](./moment.md) (input-space masked-encoder analog), [TS2Vec](../concepts/contrastive-representation-learning.md) (contrastive baseline beaten on classification)

## Related wiki pages
- [JEPA / latent-space prediction architecture](../architectures/jepa-latent-prediction.md)
- [Joint-embedding predictive architecture concept](../concepts/joint-embedding-predictive-architecture.md)
- [Contrastive representation learning](../concepts/contrastive-representation-learning.md)
