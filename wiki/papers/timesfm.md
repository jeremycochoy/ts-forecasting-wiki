# TimesFM: A Decoder-Only Foundation Model for Time-Series Forecasting

> **Short name:** `timesfm` · **arXiv:** [2310.10688](https://arxiv.org/abs/2310.10688) · **PDF:** [local](../../papers/timesfm_2310.10688.pdf) · **Date:** 2024-04 · **Venue:** ICML 2024

**Authors:** Abhimanyu Das, Weihao Kong, Rajat Sen et al. (Google Research)

## Abstract
TimesFM introduces a decoder-only transformer pretrained as a general-purpose zero-shot forecaster. The model is trained on a large heterogeneous corpus mixing Google Trends, Wikipedia Pageviews, and synthetic series, and reaches accuracy close to fully supervised baselines on standard benchmarks without any task-specific training.

## Key contributions
- Patched-decoder architecture where the output patch length is larger than the input patch length, enabling efficient multi-step autoregression.
- Handles arbitrary history length, forecast horizon, and sampling frequency from a single model.
- ~200M parameter backbone pretrained on roughly 100B time points.
- Demonstrates that zero-shot forecasting with a single TS foundation model can rival dataset-specific supervised models.

## Architecture at a glance
TimesFM is a GPT-style decoder-only transformer operating on patched real-valued tokens. Input patches are projected into the residual stream, and each output step predicts an entire output patch, shortening the autoregressive rollout. Pretraining uses a next-patch prediction loss on Google Trends, Wiki Pageviews, and synthetic seasonal/trend mixtures. Input patch size is 32, output patch size is 128; forecasting is autoregressive rollout where the larger output patch (4x input) reduces the number of rollout steps needed to cover a given horizon (Das et al. Table 6).

## Why it matters
TimesFM was among the first TS models to clearly demonstrate that a single pretrained transformer can generalize zero-shot across [Monash](../datasets-benchmarks/monash-archive.md), Darts, and ETT with quality comparable to supervised specialists, establishing the decoder-only patched paradigm for time-series foundation models.

## Strengths
- Clean empirical story: on Monash (18 datasets after filtering) the 200M model beats llmtime by >25% in scaled MAE and is within error of N-BEATS despite never training on those datasets.
- The asymmetric input/output patch trick (e.g. `input_patch=32`, `output_patch=128`) is ablated directly in Fig 3b on 512-step ETT horizons and shows a monotonic improvement as output patch grows, giving a principled middle ground between one-shot and fully autoregressive decoding.
- Preliminary scaling study across 17M / 70M / 200M checkpoints shows a log-linear FLOP-vs-error trend on Monash (Fig 3a), one of the earliest concrete scaling-law signals for TS foundation models.
- A synthetic-data ablation (Fig 3d) isolates its value: roughly no effect on ETTh (hourly, well-represented) but a clear gain on ETTm (15 min) and on under-represented Monash granularities such as quarterly and yearly.

## Limitations and open critiques
- Point-forecast only. The authors explicitly defer probabilistic forecasting to future work, so TimesFM v1 cannot be compared directly on [CRPS](../evaluation/metrics.md#21-crps--continuous-ranked-probability-score)/quantile benchmarks against [chronos](./chronos.md) or [lag-llama](./lag-llama.md).
- No covariate handling at pretraining time; the paper only sketches a zero-shot residual regression workaround and relies on finetuning for covariate-aware use. This is a real gap for retail/supply-chain deployments where exogenous inputs dominate.
- The Wiki Pageviews / Google Trends corpus is massive but narrow in character (search-interest and page-view counts). Frequencies like hourly Wiki and daily Trends dominate the mix in Table 1, which raises a fair question about how much of the "zero-shot" generalization is really cross-domain vs. cross-granularity interpolation inside a biased prior.
- Hyperparameter and architecture search was admittedly light; the paper says MLP/Mamba alternatives and covariate-aware variants were left to follow-ups.
- Benchmarks are the usual Monash/Darts/ETT triad. [Chronos-2](./chronos-2.md) and [moirai](./moirai.md) follow-ups argue these suites under-represent long-horizon, high-dimensional, and probabilistic regimes — see [../benchmarks/state-of-the-art.md](../benchmarks/state-of-the-art.md).

## Follow-up work and dialogue
TimesFM established the "decoder-only + patched + synthetic-augmented" recipe that [timer](./timer.md), [time-moe](./time-moe.md), and [sundial](./sundial.md) all build on. [timer-xl](./timer-xl.md) directly generalizes the single-series decoder to multivariate any-to-any prediction, citing TimesFM's univariate restriction as its motivation. On the probabilistic side, [chronos](./chronos.md) and [lag-llama](./lag-llama.md) pursue a different axis (tokenized/quantile output) that TimesFM v1 deliberately skipped. [moirai](./moirai.md) and [moirai-moe](./moirai-moe.md) critique TimesFM's narrow corpus and offer the [LOTSA](../datasets-benchmarks/lotsa.md) datasets plus frequency-aware or MoE routing as alternative ways to handle heterogeneity. See also [../research/open-problems.md](../research/open-problems.md) on corpus bias and covariate support.

## Reproducibility
- **Open weights:** partial — the paper states weights would be released; a checkpoint has since been published on HuggingFace (`google/timesfm-1.0-200m`), but the paper itself does not pin a hub URL.
- **Code:** referenced in paper but URL not extracted.
- **Training data:** partially public — M4 (all granularities, ~100k series), Electricity (15min + Hourly), Traffic (hourly + 15min), Weather (10-min), LibCity and Favorita Sales are public and listed in TimesFM Table 1. Google Trends is sourced from ~22k "head queries" across four granularities (~0.5B points total per the paper); Wiki Pageviews is "all Wikimedia pages" aggregated into four granularities with excessive-zero filtering (~300B points in the source corpus per TimesFM §2 "Wiki Pageviews"). Synthetic component is 3M series × 2048 points = 6.1B points. The exact Google Trends head-query list and the exact Wiki Pageviews page list used are not released.
- **Compute to retrain:** 2 days to complete 1.5M iterations at batch 4096 on a TPUv5e pod with 16 tensor cores for the 200M model; exact FLOPs not disclosed.
- **Deployment footprint:** 200M parameters; configurable context length up to 512 patches; inference demonstrated at standard-GPU scale; no explicit CPU benchmark.

## When to cite this paper
Cite TimesFM as the canonical reference for the decoder-only patched TS foundation model and for the specific trick of making the output patch longer than the input patch to reduce autoregressive steps. It is also the right citation for the first clear zero-shot parity claim against supervised specialists on Monash/Darts/ETT; later papers should be preferred for probabilistic or multivariate claims.

## In the knowledge graph
- **Cluster:** [Decoder-only autoregressive TS-FMs](../foundation-models/taxonomy.md#cluster-1--decoder-only-autoregressive-ts-fms)
- **Architecture family:** [Decoder-only autoregressive](../architectures/decoder-only-autoregressive.md)
- **Related concepts:** [patch tokenization](../concepts/patch-tokenization.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md), [scaling laws](../concepts/scaling-laws.md), [synthetic data augmentation](../concepts/synthetic-data-augmentation.md)
- **See also:** [timer](./timer.md), [lag-llama](./lag-llama.md), [time-moe](./time-moe.md)
