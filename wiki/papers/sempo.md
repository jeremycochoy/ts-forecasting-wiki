# SEMPO: Lightweight Foundation Models for Time Series Forecasting

> **Short name:** `sempo` · **arXiv:** [2510.19710](https://arxiv.org/abs/2510.19710) · **PDF:** [local](../../papers/sempo_2510.19710.pdf) · **Date:** 2025-10 · **Venue:** NeurIPS 2025 (poster)

**Authors:** Hui He, Kun Yi, Yuanchi Ma et al. (Beijing Institute of Technology, Singapore Management University, State Information Center, Tongji University; mala-lab)

## Abstract
SEMPO is a lightweight time series foundation model with only 6.5M parameters, pre-trained on a curated 83M-time-point subset of UTSD. It targets the tension between versatility and affordability that plagues modern TS foundation models: rather than scale the model and corpus, SEMPO scales neither and still beats 100M-to-700M-parameter baselines on zero-shot and few-shot forecasting. Two components drive the result. The energy-aware spectral decomposition (EASD) module transforms each input to the frequency domain and partitions it along both an energy axis and a frequency axis with a learnable threshold, so low-energy but informative components are not drowned out by the high-energy ones that dominate attention. The mixture-of-prompts Transformer (MoPFormer) replaces the hand-designed or MoE-based specialization of existing TS-FMs with a pool of 128 small learnable prompt experts, adaptively routed per token and injected into the self-attention key/value matrices.

## Key contributions
- **Energy-aware Spectral Decomposition (EASD).** FFT the input, partition the spectrum into high- and low-energy branches at a learnable threshold `tau`, then apply independently parameterized multi-band frequency masks to each branch before an iFFT. This replaces random patch masking and explicitly preserves low-energy signals the paper argues existing TS-FMs ignore.
- **Mixture-of-Prompts (MoP).** A pool of `I = 128` learnable prompt vectors in `R^{D_p}`; a linear-softmax adaptive router produces per-token gating scores `s_{i,p}`, and a weighted sum of experts is reparameterized through a two-layer MLP plus reshape into structured key/value prompts distributed across `S` transformer layers. The mixed prompt is concatenated into the self-attention K and V along the patch dimension. This is distinct from token-level FFN MoE — the routing decides a prefix, not a feed-forward branch.
- **Two-stage training.** Stage 1: self-supervised reconstruction pre-training with EASD masking, no prompts. Stage 2: MoP tuning with the backbone frozen, updating only the prompts and prediction head under a composite reconstruction + multi-resolution forecasting loss.
- **6.5M params, 83M time points, 10 hours on 4 A6000.** Full two-stage training cost is explicitly disclosed, which is unusual for the field.
- **12% / 22% average error reduction.** Across 16 datasets on the TSLib and GIFT-Eval benchmarks, SEMPO reports average MSE reductions of 12% zero-shot and 22% few-shot (5% of training data) versus state-of-the-art TS-FMs at 10x-100x more parameters and 10x-3000x more pre-training data.

## Architecture at a glance
Encoder-decoder transformer with `S = 6` layers, 16 heads, `D_p = 256`, patch size `L_p = 64`, RMSNorm, pre-normalization, SwiGLU. Input is channel-independent. After EASD masking produces `N_m = 4` masked versions of the input, each is patchified and projected; the resulting `(N_m, P, D_p)` tokens flow through the MoPFormer encoder and a reconstruction/prediction decoder. Prompts from the `I = 128` expert pool are routed per token, reparameterized into `S x 2 x D_p` key-value pairs and concatenated into every self-attention layer. Total parameter count is 6.5M. Training corpus is a curated multi-domain subset of the UTSD collection totaling roughly 83M time points, a deliberately small fraction of typical TS-FM pre-training budgets. Input patch size is 64; forecasting uses a two-stage approach: (1) reconstruction pretraining with EASD masking, (2) MoP prediction head fine-tuning that produces the forecast (He et al. Section 4).

### Sizes (He et al. §4 + Appendix C.3 + Appendix D)

| Variant | Layers (S) | d_model (D_p) | d_ff | Heads | d_kv (implied) | Params | Patch (L_p) | Context (L) |
|---|---|---|---|---|---|---|---|---|
| SEMPO-Base (default) | 6 | 256 | 256 | 16 | 16 | 6.5M | 64 | 512 |
| SEMPO-Enhanced | 6 | 256 | 256 | 16 | 16 | 7.3M | 128 | 1024 |
| SEMPO-Advanced | 6 | 256 | 256 | 16 | 16 | 9.9M | 128 | 1536 |

Encoder-decoder transformer with RMSNorm + pre-norm + SwiGLU. `d_kv` is not stated explicitly; implied as `D_p / heads = 16`. `I = 128` learnable prompt vectors are routed per token by a dense linear-softmax router (no top-K) and reparameterized into `S × 2 × D_p` key-value pairs concatenated into every self-attention layer. `N_M = 4` EASD-masked spectral views per training sample. Sparse-MoE ablation (3 experts, 1 active) is 8.5M parameters and underperforms.

## Why it matters
SEMPO is the clearest paper-level statement so far that scaling is not destiny for TS foundation models along *either* axis: it argues you can simultaneously shrink the model by two orders of magnitude *and* the pre-training corpus by three-to-four orders of magnitude and still come out ahead. It supplies two concrete mechanisms — low-energy spectral preservation and prompt-based specialization — that do the work scale was previously doing, and it ablates them.

## Strengths
- Reports head-to-head gains vs Time-MoE, Moirai, Chronos, Timer, TimesFM, and Moment on TSLib zero-shot while being 1.8x to 100x smaller in parameters and 1.3x to 3700x smaller in pre-training points (Table 1), with 12 out of 14 column wins.
- EASD is directly ablated (A.1 drops the energy-axis partitioning, A.2 replaces frequency-wise masking with random patch masking) and both ablations degrade MSE consistently across all seven TSLib datasets, isolating the contribution of each half of the spectral module.
- MoP is ablated against a sparse-MoE variant (3 experts, 1 activated, 8.5M params) and against conventional prefix tuning; SEMPO beats both at smaller total size, which is non-trivial given MoE is the current default for TS specialization.
- The efficiency plot (Figure 6) reports ETTh1 wall-clock inference at 22s vs 205s for Moirai-S, 1879s for TimesFM-200M, 14185s for Chronos-L — roughly 10x faster than the smallest prior TS-FM and 600x faster than the largest, at better accuracy.
- Training cost is explicitly disclosed: 10 hours on 4 A6000-48G GPUs, BF32, batch 2048. Compare with the opaque "industrial cluster" disclosures typical of billion-parameter TS-FMs.
- Low-energy-component preservation is shown visually (Figures 1 and 4): on ETTh1 and Electricity, ChronosS and MoiraiL collapse toward dominant high-energy peaks while SEMPO's spectrum tracks the full energy range of the ground truth.

## Limitations and open critiques
- **Point-forecast only.** No probabilistic head; the loss is MSE plus per-horizon L2, so SEMPO does not speak to CRPS / WQL-style probabilistic benchmarks and is not comparable to [Chronos-2](./chronos-2.md) or [Sundial](./sundial.md) on quantile calibration. The conclusion explicitly names "flexible distribution forecasting" as future work.
- **Channel independence only.** Multivariate inputs are decomposed into univariate series; cross-channel interactions are not modelled at pre-training or inference time. The conclusion also flags "multivariate interactions" as future work, which is a notable gap against [Moirai](./moirai.md) and [TTM](./ttm.md) which support channel mixing.
- **Two-stage training contaminates the zero-shot story.** MoP tuning adapts per dataset using a small prediction head on the target domain; the headline "zero-shot" numbers come from pre-training + tuning on other datasets. The distinction between zero-shot and few-shot is therefore cleaner than in, e.g., Chronos, but the tuning stage is not truly prompt-free.
- **Pre-training corpus is a curated UTSD subset.** The ~83M-point mix is not published as a named dataset snapshot, and the 9:1 train-val split and domain weights are only described at a high level. Reproducing "83M points" requires picking the same subset.
- **Benchmark scope is TSLib-weighted.** Seven TSLib datasets dominate the main tables; the GIFT-Eval slice is nine non-overlapping datasets (the authors remove overlaps with pre-training and TSLib), and details are pushed to Appendix D. Head-to-head on the full GIFT-Eval suite against TimesFM, Moirai or Chronos is not reported.
- **Lightweight capacity ceiling.** At 6.5M params the model has the same inductive-bias-beats-scale caveat as [TTM](./ttm.md): on very long contexts or sharply non-stationary regimes the capacity may not match a billion-parameter decoder.

## Follow-up work and dialogue
SEMPO is best read as the explicit counter-narrative to the Time-MoE / Timer-S1 / Sundial trillion-point-scale trajectory. Where [Time-MoE](./time-moe.md) and [Timer-S1](./timer-s1.md) argue that MoE transformers at 8B+ parameters and 300B+ points unlock the scaling law for time series, SEMPO argues you can cover most of that ground with 6.5M params and 83M points if you fix how existing models spend their capacity — specifically, by preserving low-energy frequency signals and pushing specialization into prompts instead of feed-forward experts.

Against [TTM](./ttm.md), the comparison is symmetric but mechanism-different: both target the sub-10M regime, but TTM uses a non-transformer TSMixer backbone with adaptive patching and resolution-prefix tuning, while SEMPO stays with a transformer and injects prompt experts into the attention. SEMPO reports a 4.6% MSE reduction over TTM in 5% few-shot (Table 2), though on ETTh1 zero-shot TTM remains competitive. The two papers together constitute the strongest case that "transformer by default" is not justified in TS FMs — but they disagree on whether to abandon the transformer or fix it.

Against [Moirai-MoE](./moirai-moe.md) and [Time-MoE](./time-moe.md), the key distinction is where the routing lives. Moirai-MoE and Time-MoE route each token through a top-k subset of feed-forward experts, so each token picks a different FFN; the activated-parameter count stays below the total by sparsity. SEMPO routes each token to a soft combination of *prompt* experts — prefix vectors concatenated into the attention key/value — so routing controls which prefix conditions the token, not which FFN processes it. There is no top-k gating (the router is a dense softmax), and the total parameter count is already small so there is nothing to amortize. The ablation in Table 3 where MoP is replaced with a 3-expert sparse MoE (B.1) isolates exactly this choice and shows MoP wins at smaller size.

Against [Mamba4Cast](./mamba4cast.md), SEMPO is the other "small model, surprising results" entry in the lightweight cluster, but Mamba4Cast trains on synthetic priors via a PFN and uses a state-space backbone, while SEMPO trains on real curated data with a prompt transformer.

## Reproducibility
- **Open weights:** code and data released at `github.com/mala-lab/SEMPO` per the abstract; weight artifacts referenced in the paper but specific hub identifiers not extracted.
- **Code:** public — `github.com/mala-lab/SEMPO`.
- **Training data:** fully public in principle — a curated subset of the UTSD collection totaling ~83M time points with a 9:1 train-val split; the exact subset list and per-source weighting are documented in Appendix C but not released as a named snapshot.
- **Compute to retrain:** disclosed — 10 hours on 4 NVIDIA A6000-48G GPUs, BF32, batch size 2048, AdamW (lr 1e-3, wd 0.1, betas 0.9/0.95), 10,000 linear warmup steps. This is among the most reproducible pre-training budgets disclosed in a TS foundation model paper.
- **Deployment footprint:** 6.5M parameters; `S = 6` layers, 16 heads, `D_p = 256`, `L_p = 64`, lookback `L = 512`; inference on ETTh1 measured at 22s total (Figure 6). CPU-feasible in principle but not explicitly benchmarked on CPU.

## When to cite this paper
Cite SEMPO (a) as the canonical reference for the claim that pre-training corpus size and model size can be reduced simultaneously without sacrificing zero- and few-shot generalization on TSLib and GIFT-Eval, (b) as the introduction of energy-aware spectral decomposition and the argument that Fourier-based TS methods systematically ignore low-energy yet informative components, (c) as the first clear demonstration of mixture-of-prompts routing as an alternative to mixture-of-experts for parameter-efficient specialization in a TS foundation model, and (d) as the 6.5M-parameter entry in any efficiency-versus-accuracy comparison where TTM, Moirai-S, and Chronos-S are the relevant baselines.

## In the knowledge graph
- **Cluster:** [Cluster 5 — Lightweight / non-transformer FMs](../foundation-models/taxonomy.md#cluster-5--lightweight--non-transformer-fms) (primary — the 6.5M parameter tier places it alongside [TTM](./ttm.md) and [Mamba4Cast](./mamba4cast.md) even though the backbone is a transformer)
- **Architecture family:** [lightweight non-transformer](../architectures/lightweight-non-transformer.md) (as the "sub-10M lightweight regime" page), with secondary links to [masked encoder](../architectures/masked-encoder.md) (encoder-decoder with reconstruction pre-training) and [mixture of experts](../architectures/mixture-of-experts.md) (for contrast — SEMPO's MoP is explicitly *not* MoE)
- **Related concepts:** [patch tokenization](../concepts/patch-tokenization.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md), [scaling laws](../concepts/scaling-laws.md) (counter-narrative: less data, fewer params, competitive results), [revin normalization](../concepts/revin-normalization.md) (instance norm before FFT)
- **Dataset / corpus:** curated ~83M-point subset of UTSD; not disclosed as a named dataset snapshot. Evaluation on TSLib (ETTh1/h2, ETTm1/m2, Weather, Electricity, Traffic) and 9 non-overlapping datasets from [GIFT-Eval](../datasets-benchmarks/gift-eval.md).
- **See also:** [ttm](./ttm.md) (same sub-10M lightweight regime, non-transformer), [mamba4cast](./mamba4cast.md) (other small-model path, state-space backbone), [moirai-moe](./moirai-moe.md) (contrast: SEMPO's mixture-of-prompts routes prefix conditioning, Moirai-MoE's mixture-of-experts routes FFN selection), [time-moe](./time-moe.md) (scale counterpoint), [timer-s1](./timer-s1.md) (opposite end of the scaling axis)

## Related wiki pages
- [Cluster 5 — Lightweight / non-transformer FMs](../foundation-models/taxonomy.md#cluster-5--lightweight--non-transformer-fms)
- [lightweight non-transformer](../architectures/lightweight-non-transformer.md)
- [scaling laws](../concepts/scaling-laws.md)
- [zero-shot forecasting](../concepts/zero-shot-forecasting.md)
