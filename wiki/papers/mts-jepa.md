# MTS-JEPA: Multi-Resolution Joint-Embedding Predictive Architecture for Time-Series Anomaly Prediction

> **Short name:** `mts-jepa` · **arXiv:** [2602.04643](https://arxiv.org/abs/2602.04643) · **PDF:** [local](../../papers/mts-jepa_2602.04643.pdf) · **Date:** 2026-02 · **Venue:** preprint

**Authors:** Yanan He (Purdue), Yunshi Wen (RPI), Xin Wang (Stony Brook), Tengfei Ma (Stony Brook)

## Abstract
MTS-JEPA targets multivariate time-series *anomaly prediction* — early warning ahead of a failure window, distinct from reactive detection or value forecasting. The paper extends the standard single-resolution JEPA recipe with two components: a multi-resolution predictive objective (parallel fine and coarse latent pathways) and a soft codebook bottleneck that enforces a discrete prototype geometry on the latent space. The bottleneck doubles as an intrinsic regularizer that prevents JEPA's signature representation-collapse failure mode without explicit negative samples. On four anomaly benchmarks (MSL, SMAP, SWaT, PSM) the model reaches top F1 and AUC across all four, with the largest gains on MSL.

## Key contributions
- Reformulates time-series anomaly *prediction* (predict whether window `W_{t+1}` will be anomalous given `W_t`) as latent-space JEPA prediction, in contrast to reconstruction-error or score-based detection methods.
- Multi-resolution predictive objective: an asymmetric teacher-student where the online branch sees only the fine-grained patched view but is supervised against both fine and coarse-grained latent targets, forcing global structure to be inferred from local observation.
- Soft codebook bottleneck `Q` mapping continuous encoder features to a simplex distribution over `K` learnable prototypes via temperature-scaled cosine similarity. The bounded convex-hull geometry gives an analytical non-collapse certificate (paper Appendix A.3) and turns latent prediction into a KL alignment between distributions on the simplex rather than unconstrained MSE in `R^D`.
- A combined loss `L = L_pred + L_code + λ_r L_rec` with `L_pred = λ_f (L_KL_fine + γ L_MSE_fine) + λ_c L_KL_coarse`. The auxiliary reconstruction term plus the codebook keep self-distillation stable.
- Strong cross-domain generality study (paper Table 2): trained on three datasets and evaluated on a held-out fourth, MTS-JEPA's AUC drops less than PatchTST or TS2Vec, and on SMAP the cross-domain checkpoint actually beats the in-domain.

## Architecture at a glance
A multivariate window `x ∈ R^{T × V}` is RevIN-normalized, then split into **fine view** (`P` patches of length `L`, with `T_w = P · L`) and **coarse view** (a single down-averaged token of length `L`). A shared **online encoder** `E_θ` (residual CNN tokenizer + transformer backbone) processes only the fine view at time `t`. The **EMA encoder** `E_ξ` processes both views at time `t+1` and produces multi-scale targets. The **soft codebook** `Q` projects continuous features `h_t,i` to simplex distributions `p_t,i,k = softmax(<h̄_t,i, c̄_k>/τ)` over `K` prototypes; the codebook is itself EMA-updated. The **dual predictor** has a fine branch (transformer over the patch sequence, output shape `R^{P × K}`) and a coarse branch (cross-attention with a learnable query token, output `R^{1 × K}`). An auxiliary **decoder** reconstructs the input patches from the soft-quantized embeddings to anchor the latent space. Training uses non-overlapping context-target pairs with stride 100, context length 100, target length 100, `P=5`, `L=20`.

### Sizes (He et al. Appendix B.2)

| Component | Type | Layers | d_model | d_ff | Heads | d_kv | Notes |
|---|---|---|---|---|---|---|---|
| Online encoder E_θ | residual CNN tokenizer + 2-layer projection MLP (64→32) + Transformer | 6 | 256 | not disclosed | 8 | not disclosed | EMA decay ρ=0.996 for E_ξ twin |
| EMA encoder E_ξ | identical to online encoder | 6 | 256 | not disclosed | 8 | not disclosed | weights via `ξ ← ρξ + (1−ρ)θ` |
| Fine predictor | Transformer | 2 | 128 | not disclosed | 4 | not disclosed | output `R^{P × K}` |
| Coarse predictor | Transformer (cross-attn with learnable query) | 2 | 128 | not disclosed | 4 | not disclosed | output `R^{1 × K}` |
| Codebook Q | K=128 prototypes, dim D=256, τ=0.1 | n/a | 256 | n/a | n/a | n/a | EMA-updated |
| Decoder (auxiliary recon) | not specified | not disclosed | not disclosed | not disclosed | not disclosed | not disclosed | input-space reconstruction |

Total params not disclosed. **Asymmetric** depth and width: predictor (L=2, d=128) is shallower and narrower than encoder (L=6, d=256). Patch length L=20, P=5 patches per window (T_c = T_t = 100). Channel-independent input formulation; RevIN normalization with cached affine stats for reconstruction loss inversion.

## Why it matters
This is the first JEPA paper to (a) tackle anomaly prediction rather than forecasting or classification, (b) introduce a multi-resolution latent-prediction objective, and (c) demonstrate that a discrete codebook can serve double duty as a regime detector and a self-distillation stabilizer. The "codebook prevents collapse" argument is the strongest theoretical-empirical bridge in the JEPA-for-TS literature so far: paper Appendix A.3 derives both an upper bound (bounded latent excursions from convex-hull membership) and a strictly positive lower bound on batch variance, and the ablation in Table 3 shows F1 collapses to near-random when the codebook module is removed (`w/o Codebook Module`) — much worse than removing only the auxiliary codebook losses.

## Strengths
- Top F1 *and* AUC on all four benchmarks — MSL, SMAP, SWaT, PSM — beating nine baselines including [TS-JEPA](./ts-jepa.md), TS2Vec, PatchTST, iTransformer, TimesNet, PAD, LSTM-VAE, K-Means, DeepSVDD (paper Table 1). MSL F1 33.58% vs TS-JEPA 25.49%; PSM AUC 77.85% vs TS2Vec 72.13%.
- Cross-domain transfer holds up: on SMAP, training on MSL+SWaT+PSM and evaluating on SMAP gives F1 33.81% / AUC 68.88% vs in-domain 33.64% / 65.41% — the cross-domain AUC is *higher*, suggesting the soft codebook learns prototypes general enough to transfer.
- The ablation table is unusually comprehensive: removing each of KL, reconstruction, predictive objective, codebook loss, codebook module, and downsample is reported separately. The codebook-module removal causing near-collapse is the cleanest evidence in the JEPA literature that geometric constraints (not just EMA targets and stop-gradients) are sometimes necessary.
- The dual-predictor design (transformer fine + cross-attention coarse) is a small but principled architectural change that maps cleanly onto the multi-resolution motivation.
- Visualization (paper Figure 4) shows a small subset of codebook entries activate disproportionately on anomalous windows, providing a concrete handle for interpretability.

## Limitations and open critiques
- Anomaly prediction only; no forecasting or classification benchmarks. Cannot be placed directly on the [GIFT-Eval](../datasets-benchmarks/gift-eval.md) or Monash leaderboards alongside the rest of the TS-FM cohort.
- Four anomaly datasets is the standard suite, but they are small (MSL/SMAP are spacecraft telemetry, SWaT is one industrial testbed, PSM is one server fleet). Generalization to genuinely novel domains is not tested beyond the cross-three-train-on-one transfer.
- Hyperparameter complexity is high: `K` codebook prototypes, temperature `τ`, `λ_f`, `λ_c`, `γ`, `λ_r`, `m_codebook`, `m_encoder`, EMA decay schedules, learning-rate schedule. The paper does not characterize sensitivity to most of these.
- Soft codebook is presented as a regularizer, but it adds substantial machinery (codebook EMA updates, simplex projection, KL alignment) compared to the I-JEPA recipe of stop-gradient + EMA alone. Whether something simpler (e.g., variance-covariance regularization à la VICReg) would suffice is not tested.
- No scaling study; the paper does not vary patch count `P`, codebook size `K`, or backbone width.
- Anomaly *prediction* is sensitive to the precursor-window definition (`stride 100`, target `T_t = 100`); robustness to that choice is not characterized.

## Follow-up work and dialogue
MTS-JEPA stands on the shoulders of [TS-JEPA](./ts-jepa.md) (which it cites as the single-resolution baseline) and [LaT-PFN](./lat-pfn.md) (the JEPA + PFN hybrid that pioneered JEPA on TS). Its codebook borrows from the VQ-VAE lineage that also underpins [TOTEM](./totem.md) and [Chronos](./chronos.md) — both of which use discrete codes for tokenization rather than for anomaly-prototype regimes. The multi-resolution argument echoes [TimeMixer](https://arxiv.org/abs/2405.14616) and [Pyraformer](https://openreview.net/forum?id=0EXmFzUn5I) but applies it inside a self-supervised JEPA loop rather than a supervised forecasting head. The collapse-prevention discussion connects MTS-JEPA to the broader I-JEPA / V-JEPA / DINO literature, where stop-gradient + EMA is the dominant recipe; MTS-JEPA's codebook constraint is an alternative geometric stabilization route worth tracking.

## Reproducibility
- **Open weights:** not advertised in the preprint.
- **Code:** the paper does not include a public repository URL.
- **Training data:** fully public — MSL, SMAP (NASA spacecraft telemetry), SWaT (industrial control testbed), PSM (eBay server metrics).
- **Compute to retrain:** the paper does not tabulate GPU-hours; experiments are five-seed averages but per-seed cost is not disclosed.
- **Deployment footprint:** parameter count is not tabulated; the encoder is a residual CNN tokenizer + transformer backbone, the predictor has fine and coarse branches, the codebook is `R^{K × D}`. Operates on context length `T_c = 100`, `P = 5` patches of length `L = 20`.

## When to cite this paper
Cite MTS-JEPA as the canonical reference for multi-resolution JEPA in time series, for the soft-codebook approach to representation-collapse prevention without explicit negatives, and for the framing of anomaly *prediction* (early warning) as a JEPA latent-prediction problem. It is also the right citation for cross-domain JEPA transfer under domain shift in industrial-anomaly settings.

## In the knowledge graph
- **Cluster:** [Cluster 8 — JEPA / latent-space prediction](../foundation-models/taxonomy.md#cluster-8--jepa--latent-space-prediction)
- **Architecture family:** [JEPA / latent-target prediction](../architectures/jepa-latent-prediction.md)
- **Related concepts:** [joint-embedding predictive architecture](../concepts/joint-embedding-predictive-architecture.md), [value quantization](../concepts/value-quantization.md) (codebook), [data normalization](../concepts/data-normalization.md) (RevIN), [multi-task universal](../concepts/multi-task-universal.md)
- **See also:** [TS-JEPA](./ts-jepa.md) (single-resolution predecessor, used as ablation baseline), [LaT-PFN](./lat-pfn.md) (JEPA+PFN hybrid), [TOTEM](./totem.md) (different VQ codebook role), [TSPulse](./tspulse.md) (multi-task masked-encoder, contrast point)

## Related wiki pages
- [JEPA / latent-space prediction architecture](../architectures/jepa-latent-prediction.md)
- [Joint-embedding predictive architecture concept](../concepts/joint-embedding-predictive-architecture.md)
- [Value quantization](../concepts/value-quantization.md)
