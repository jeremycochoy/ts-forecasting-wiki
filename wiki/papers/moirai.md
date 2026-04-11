# MOIRAI: Unified Training of Universal Time Series Forecasting Transformers

> **Short name:** `moirai` · **arXiv:** [2402.02592](https://arxiv.org/abs/2402.02592) · **PDF:** [local](../../papers/moirai_2402.02592.pdf) · **Date:** 2024-02 · **Venue:** ICML 2024

**Authors:** Gerald Woo, Chenghao Liu, Akshat Kumar, Caiming Xiong, Silvio Savarese, Doyen Sahoo (Salesforce AI Research)

## Abstract
MOIRAI introduces a unified masked-encoder transformer trained on LOTSA, a curated open corpus of roughly 27B observations across nine domains. The model tackles variable frequencies, variable number of variates, and probabilistic outputs in a single architecture.

## Key contributions
- Release of LOTSA, one of the largest open TS pretraining corpora with over 27B observations across nine domains.
- Multi-patch-size input and output projection layers (larger patches for high-frequency data, smaller for low-frequency) for explicit frequency specialization.
- Any-variate attention: time and variate axes are flattened into a single sequence with RoPE along time and learned binary biases along the variate axis, supporting arbitrary multivariate inputs without re-training.
- Mixture of parametric distributions (including Student-t) as the output head, producing flexible probabilistic forecasts with robust tails.
- Three open-weight sizes: MOIRAI-Small (14M), MOIRAI-Base (91M), MOIRAI-Large (311M).

## Architecture at a glance
MOIRAI is an encoder-only transformer that patches each input series with a frequency-dependent patch size chosen from a fixed set. Tokens from multiple variates are flattened into a single sequence, and any-variate attention lets the model condition each token on arbitrary others. A mixture-of-distributions head outputs the forecast distribution. Training optimizes negative log-likelihood on randomly sampled context and prediction lengths from LOTSA.

## Why it matters
MOIRAI was one of the first TS foundation models to fully commit to open, large-scale pretraining and to solve the variable-frequency, variable-variate problem inside one architecture, setting a standard for universal TS forecasters.

## Strengths
- LOTSA at roughly 27B observations across nine domains (energy, transport, climate, CloudOps, web, sales, healthcare, nature, economics) is, at time of publication, the largest fully open TS pretraining corpus, and is released together with the `uni2ts` training library.
- The multi-patch-size projection directly addresses cross-frequency negative interference, which prior work like N-BEATS handled by training one model per frequency; MOIRAI's ablation shows the frequency-specialized projections materially help on mixed-frequency pretraining.
- Any-variate attention is a clean, architecture-preserving answer to variable dimensionality: the same trained weights serve univariate and multivariate inference without fine-tuning, using a single shared transformer stack rather than a tower-per-variate.
- Flexible mixture-of-parametric output head avoids the positivity failure mode of Gaussian / Student-t heads on strictly-positive series and is competitive with direct quantile regression.
- Open under Apache-like terms (code, weights, LOTSA) makes MOIRAI the default open baseline for later work.

## Limitations and open critiques
- Flattening all variate-time tokens into one sequence makes attention cost O((V*T)^2) in the number of variates V times context length T. [chronos-2](./chronos-2.md) explicitly critiques this as O(V^2) memory scaling and introduces group attention to get back to O(V).
- Frequency specialization via discrete patch sizes is brittle: unseen or rare frequencies fall back to the nearest bucket, and [moirai-moe](./moirai-moe.md) argues the whole "frequency as a first-class input specialization" axis is wrong and replaces it with token-level sparse routing.
- Multivariate evaluation in the main paper leans on the [Monash](../datasets-benchmarks/monash-archive.md) multivariate splits and Long-Sequence Forecasting datasets (ETT, Electricity, Weather); the covariate-informed regime that [chronos-2](./chronos-2.md) and Toto emphasize is only partially covered.
- Scaling evidence is limited to three sizes at 14M / 91M / 311M, and the paper does not produce a Chinchilla-style log-log curve. Follow-ups (MOIRAI-2.0, TiRex, TimesFM-2.5) keep reporting gains that suggest MOIRAI is not yet at a scaling plateau.
- LOTSA includes several datasets that also appear in downstream evaluation (e.g. LOTSA CloudOps overlaps with CloudOps benchmarks), which weakens pure zero-shot framings; see [../research/reproducibility.md](../research/reproducibility.md) and [../benchmarks/state-of-the-art.md](../benchmarks/state-of-the-art.md).

## Follow-up work and dialogue
[moirai-moe](./moirai-moe.md) is the direct successor from the same group and argues that frequency-aware input specialization is a band-aid: they keep the masked-encoder any-variate framing but replace the frequency-specialized projections with sparse MoE routing over patch tokens, claiming the MoE approach generalizes better across unseen frequencies. [chronos-2](./chronos-2.md) critiques MOIRAI's O(V^2) memory and replaces any-variate attention with group attention. [moment](./moment.md) and [chronos](./chronos.md) contrast with MOIRAI on corpus choice ([Time Series Pile](../datasets-benchmarks/time-series-pile.md), Chronos corpus) and on architecture (MOMENT is single-patch-size, Chronos is quantized-vocabulary). On the decoder-only side, [timesfm](./timesfm.md), [timer](./timer.md), and [timer-xl](./timer-xl.md) take the opposite bet: no frequency specialization at all, and a patched autoregressive decoder; [timer-xl](./timer-xl.md) explicitly targets the any-variate-any-horizon setting MOIRAI solves with bidirectional attention.

## Reproducibility
- **Open weights:** yes — `Salesforce/moirai-1.0-R-small`, `-base`, `-large` on HuggingFace.
- **Code:** public at `SalesforceAIResearch/uni2ts`.
- **Training data:** fully public — LOTSA is released on HuggingFace with per-dataset licenses documented.
- **Compute to retrain:** AdamW with peak learning rate 1e-3, weight decay 0.1, `β1=0.9`, `β2=0.98`, linear warmup of 10,000 steps followed by cosine annealing. MOIRAI-Small is trained for 100K steps; Base and Large for 1M steps; nominal batch size 256 with effective batch size increased via sequence packing, on NVIDIA A100 40GB GPUs (MOIRAI, Appendix). Exact GPU count and wall-clock are not reported. See [../research/training-recipes.md](../research/training-recipes.md).
- **Deployment footprint:** 14M / 91M / 311M parameters; context and prediction lengths sampled during training so downstream length is flexible; inference demonstrated on standard GPUs.

## When to cite this paper
Cite MOIRAI as the canonical reference for LOTSA and for the "any-variate attention + multi-patch-size + flexible mixture output" recipe of a universal masked-encoder TS foundation model. It is also the right citation for the first clear demonstration that a single open encoder can handle arbitrary frequency, arbitrary number of variates, and flexible probabilistic output without task-specific fine-tuning.

## In the knowledge graph
- **Cluster:** [Masked-encoder / encoder-decoder TS-FMs](../foundation-models/taxonomy.md#cluster-2--masked-encoder--encoder-decoder-ts-fms)
- **Architecture family:** [Masked encoder](../architectures/masked-encoder.md)
- **Related concepts:** [patch tokenization](../concepts/patch-tokenization.md), [probabilistic forecasting](../concepts/probabilistic-forecasting.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md), [multi-task universal](../concepts/multi-task-universal.md)
- **Dataset / corpus:** [LOTSA](../datasets-benchmarks/lotsa.md)
- **See also:** [moirai-moe](./moirai-moe.md), [chronos](./chronos.md), [moment](./moment.md)
