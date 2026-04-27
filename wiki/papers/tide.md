# Long-term Forecasting with TiDE: Time-series Dense Encoder

> **Short name:** `tide` · **arXiv:** [2304.08424](https://arxiv.org/abs/2304.08424) · **PDF:** [local](../../papers/tide_2304.08424.pdf) · **Date:** 2023-04 (v5: 2024-04) · **Venue:** TMLR 2023

**Authors:** Abhimanyu Das, Weihao Kong, Andrew Leach, Shaan Mathur, Rajat Sen, Rose Yu (Google Research / Google Cloud / UC San Diego)

## Abstract
TiDE is a Multi-Layer Perceptron (MLP) encoder-decoder for long-term time-series forecasting that handles covariates and non-linear dependencies while keeping the simplicity and speed of linear baselines. The authors prove that a linear analogue of TiDE attains a near-optimal error rate on linear dynamical systems under mild spectral assumptions, and empirically show that the full model matches or outperforms transformer baselines on the standard long-horizon benchmarks while being 5–10× faster than the best transformer.

## Key contributions
- A pure-MLP encoder-decoder forecaster with no self-attention, recurrence, or convolution; computational cost is linear in the look-back `L` and horizon `H`.
- A "feature projection" residual block that compresses high-dimensional dynamic covariates per time-step before they reach the encoder, side-stepping the `(L+H)·r` blow-up of naively flattened covariates.
- A "temporal decoder" residual block that re-injects future covariates into the per-step prediction — a highway from `\tilde x_{L+t}` to `\hat y_{L+t}` that lets sharp covariate-driven events (holidays, promotions) propagate without waiting for the encoder bottleneck.
- A theoretical result (Section A of the paper) that the linear simplification of TiDE is near-optimal on linear dynamical systems whose state matrix has maximum singular value bounded away from one.

## Architecture at a glance
TiDE is channel-independent: each series is encoded separately with shared weights. The encoder concatenates the look-back `y_{1:L}^{(i)}`, projected dynamic covariates `\tilde x_{1:L+H}^{(i)}`, and static attributes `a^{(i)}`, then passes them through `n_e` residual MLP blocks. The decoder produces an `H × p` matrix; a per-step temporal decoder turns each column into a scalar prediction by concatenating with the projected future covariate. A global linear residual connection from the look-back to the horizon is added on top, so DLinear is exactly recoverable as a degenerate sub-model. Training uses MSE with mini-batch SGD (Adam) on rolling-origin look-back/horizon pairs.

## Why it matters
TiDE crystallized the post-DLinear consensus that, for the canonical long-horizon ETT/Weather/Traffic/Electricity benchmarks, transformer self-attention adds little once a strong linear residual and a covariate-aware MLP are in place. It is the same Google research group (Das, Kong, Sen) that later released [TimesFM](./timesfm.md), and several TimesFM design choices — channel independence, covariate handling, a global residual — trace directly back to TiDE.

## Strengths
- Matches PatchTST on 6 of 7 long-horizon datasets in Table 2 of the paper, beats DLinear in nearly every (dataset, horizon) cell, and clearly leads on the largest dataset (Traffic, 862 series), where MSE is 10.6% lower than PatchTST at horizon 720.
- 5× faster inference and >10× faster training than PatchTST on the Electricity benchmark; PatchTST runs out of GPU memory for `L ≥ 1440`, while TiDE scales linearly in `L` and trains comfortably to `L = 2880` (Figure 2).
- Wins decisively on the M5 retail competition (Table 3): WRMSSE 0.611 against DeepAR 0.789 and PatchTST 0.976. The PatchTST gap is driven by PatchTST's inability to consume covariates, which is precisely the scenario the temporal decoder targets.
- Self-contained ablations cover the residual connections (Table 5), the temporal decoder (Figure 3, semi-synthetic event experiment), and the look-back/horizon trade-off (Figure 4, monotonically improving MSE as `L` grows on Traffic).

## Limitations and open critiques
- Single-domain trained per dataset; no pretraining, no zero-shot transfer. TiDE is not a foundation model — it sits in the same lineage as N-BEATS, DLinear, and PatchTST, and is consequently absent from cross-domain leaderboards like [GIFT-Eval](../datasets-benchmarks/gift-eval.md) and [Chronos Benchmark II](../benchmarks/leaderboard.md).
- Point-only output. There is no probabilistic head; reported metrics are MSE / MAE on standard-normalized data. CRPS, WQL, and quantile coverage are out of scope.
- The benchmark suite is exactly the seven LTSF datasets — ETT (×4), Weather, Traffic, Electricity. As discussed in [methodology-caveats](../benchmarks/methodology-caveats.md), this suite over-represents short context, single-domain hourly data, and undersamples the long-horizon, high-dimensional, probabilistic regimes that follow-up TS-FMs target.
- The MLP-Mixer / N-BEATS family is not benchmarked head-to-head; only N-HiTS appears in Table 2. A direct comparison to TSMixer or NLinear at matched parameter counts would have strengthened the "self-attention is unnecessary" claim.
- The theoretical result is for the *linear analogue*, not the full non-linear model; transferring its near-optimality guarantee to TiDE-with-MLPs is left informal.

## Follow-up work and dialogue
TiDE serves two roles in the post-2023 literature. First, as a strong supervised baseline on long-horizon benchmarks: [moirai](./moirai.md) reports TiDE alongside PatchTST/TFT/DeepAR as the dataset-specific full-shot reference, and [chronos](./chronos.md) and the GIFT-Eval suite reuse it the same way (see [evaluation/seasonality-and-baselines.md](../evaluation/seasonality-and-baselines.md) and [evaluation/what-was-evaluated.md](../evaluation/what-was-evaluated.md)). Second, as a design ancestor of [TimesFM](./timesfm.md): channel independence, an explicit covariate pathway, MSE training, and a linear residual all cross over. The TimesFM paper does not formally inherit TiDE's MLP backbone, but the team and the engineering choices are continuous. The lightweight non-transformer cluster — [TTM](./ttm.md), [SEMPO](./sempo.md), [TSPulse](./tspulse.md) — is the spiritual successor: take TiDE-style efficiency, add pretraining and channel mixing, and you get a TS-FM that runs on CPU.

## Reproducibility
- **Open weights:** not applicable — TiDE is trained per dataset, not pretrained; reference checkpoints are not the deliverable.
- **Code:** the paper supplements include a reference implementation; the public release lives under [google-research/google-research/tide](https://github.com/google-research/google-research/tree/master/tide) (paper §5.1).
- **Training data:** fully public — the seven LTSF benchmarks (ETT ×4, Weather, Traffic, Electricity) and the M5 competition data; train/val/test split is the standard 7:1:2 inherited from Wu et al. 2021.
- **Compute to retrain:** single NVIDIA T4 GPU, 64-core Xeon, batch size 8 on Electricity (`L` up to 2880); per-epoch training time scales linearly in `L` (Figure 2b). Exact GPU-hours are not tabulated.
- **Deployment footprint:** model size depends on `(L, H, hiddenSize, n_e, n_d)` and is in the low-millions parameter regime for the LTSF-tuned configurations; appendix B.3 of the paper gives the per-dataset hyperparameters.

## When to cite this paper
Cite TiDE as the canonical reference for the claim that a pure MLP encoder-decoder with a covariate highway and a linear residual matches transformer self-attention on long-horizon benchmarks, at a 5–10× speed advantage. It is also the right citation for the lineage from DLinear to TimesFM (same lead author, same engineering DNA), and for the M5-with-covariates result that singles out covariate handling as a real axis where PatchTST-style channel-independent transformers underperform.

## In the knowledge graph
- **Cluster:** Pre-FM long-horizon baseline (not part of the eight-cluster TS-FM taxonomy). See [foundations/deep-learning-era](../foundations/deep-learning-era.md).
- **Architecture family:** MLP encoder-decoder; closest TS-FM cousins are the [lightweight non-transformer](../architectures/lightweight-non-transformer.md) family ([TTM](./ttm.md), [TSPulse](./tspulse.md)).
- **Related concepts:** [patch tokenization](../concepts/patch-tokenization.md) (counter-example: TiDE deliberately avoids patches), [data normalization](../concepts/data-normalization.md), [probabilistic forecasting](../concepts/probabilistic-forecasting.md) (counter-example: TiDE is point-only).
- **Used as a baseline in:** [moirai](./moirai.md), [GIFT-Eval](../datasets-benchmarks/gift-eval.md) full-shot reference; see [evaluation/what-was-evaluated.md](../evaluation/what-was-evaluated.md).
- **See also:** [TimesFM](./timesfm.md) (same Google team, decoder-only foundation-model successor), [TTM](./ttm.md) (TSMixer-based lightweight TS-FM), [SEMPO](./sempo.md) (6.5M lightweight encoder-decoder).

## Related wiki pages
- [foundations/deep-learning-era](../foundations/deep-learning-era.md)
- [benchmarks/leaderboard](../benchmarks/leaderboard.md)
- [evaluation/seasonality-and-baselines](../evaluation/seasonality-and-baselines.md)
