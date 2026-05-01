# Timer: Generative Pre-trained Transformers Are Large Time Series Models

> **Short name:** `timer` · **arXiv:** [2402.02368](https://arxiv.org/abs/2402.02368) · **PDF:** [local](../../papers/timer_2402.02368.pdf) · **Date:** 2024-02 · **Venue:** ICML 2024

**Authors:** Yong Liu, Haoran Zhang, Chenyu Li, Xiangdong Huang, Jianmin Wang, Mingsheng Long (Tsinghua)

## Abstract
Timer recasts time series as a generative pretraining problem, introducing a unified Single-Series Sequence (S3) format and training a GPT-style decoder-only transformer on the Unified Time Series Dataset (UTSD), which scales up to roughly 1B time points. The resulting model unifies forecasting, imputation, and anomaly detection under a single next-token objective.

## Key contributions
- Single-Series Sequence (S3) format converting heterogeneous multivariate TS into a unified univariate token stream by splitting channels into independent sequences.
- UTSD, a hierarchical pretraining corpus released at 1G / 2G / 4G / 12G scales, covering energy, health, environment, nature, transport, IoT, and web domains.
- Decoder-only autoregressive GPT-style pretraining with next-token prediction on patched real-valued tokens.
- Unified generative treatment of forecasting, imputation (as inpainting via masked generation), and anomaly detection (as reconstruction error).
- Scaling experiments along both parameter count (1M to 50M) and data scale (UTSD-1G to UTSD-12G) showing consistent downstream improvements.
- Few-shot transfer reducing fine-tuning data by up to 99% on downstream tasks compared to from-scratch training.

## Architecture at a glance
Timer is a GPT-style decoder-only transformer with causal self-attention trained with next-token prediction on patched real-valued tokens, using segment length S=96. The S3 format splits multivariate data into channel-independent univariate sequences so that heterogeneous datasets with different dimensionalities and frequencies all feed the same model without architectural changes. Downstream tasks are reformulated as generation: forecasting via continued generation, imputation via masked-segment generation, anomaly detection via reconstruction loss. Input and output patch size is 96 (segment length S=96); forecasting is autoregressive rollout, one patch per step via next-token prediction over the S3 format (Liu et al. Table 14).

### Sizes (Liu et al. Table 6, heads fixed at 8 across all sizes)

| Variant | Layers | d_model | d_ff | Heads | d_kv | Params | Patch | Context |
|---|---|---|---|---|---|---|---|---|
| Timer dim-sweep (d=256 / 512 / 768 / 1024) | 6 | 256 / 512 / 768 / 1024 | 512 / 1024 / 1536 / 2048 | 8 | — | 3.21M / 12.72M / 28.51M / 50.59M | 96 | ≤1440 |
| Timer layer-sweep (L=2 / 4 / 6 / 8 at d=256) | 2 / 4 / 6 / 8 | 256 | 512 | 8 | — | 1.10M / 2.16M / 3.21M / 4.27M | 96 | ≤1440 |
| Timer-29M (released main) | 6 | 768 | 1536 | 8 | — | 28.51M | 96 | ≤1440 |
| Timer-50M (released) | 6 | 1024 | 2048 | 8 | — | 50.59M | 96 | ≤1440 |
| Timer-67M (released largest) | 8 | 1024 | 2048 | 8 | — | 67.40M | 96 | ≤1440 |

Released checkpoints `Timer-1B / 16B / 28B` differ only in pretraining corpus scale (UTSD-1G / UTSD+Buildings900K / UTSD+LOTSA), not in `(L, d, h)`. `d_kv` not tabulated; the paper sets per-head dim implicitly via `d_model / heads`.

## Why it matters
Timer is one of the clearest demonstrations that the LLM recipe — a large decoder-only transformer trained generatively on a unified sequence format — transfers to time series and yields strong few-shot behavior on downstream TS tasks. It is one of the first TS-FM papers to report parameter and data scaling curves side-by-side.

## Strengths
- UTSD is released in four hierarchical sizes (1G / 2G / 4G / 12G), enabling controlled data-scaling studies the way language-model papers run token-budget sweeps. This is rare in the TS-FM literature.
- Parameter scaling is studied at fixed data size (1M to 50M parameters), giving direct evidence that scaling both axes matters and producing reported improvements of 14.7-25.1% on PEMS few-shot forecasting.
- Unified task treatment is not hand-waved: the paper reports forecasting, imputation, and anomaly detection with the same backbone and same next-token loss, matching or exceeding task-specific small models on each.
- Channel-independent S3 format sidesteps the variable-dimensionality problem cleanly without needing frequency-specialized projections like [moirai](./moirai.md) or a flattened any-variate sequence.
- Emphasizes the data-scarce regime, which is where foundation models are actually useful in practice; evidence for up to 99% fine-tuning-data reduction is concrete and benchmark-backed.

## Limitations and open critiques
- Channel independence means Timer cannot model cross-variate dependencies at inference time. This is a deliberate choice but leaves multivariate / covariate-informed tasks on the table. [timer-xl](./timer-xl.md) is the same group's own answer to this critique.
- Point-forecast only: Timer outputs deterministic next-patch predictions without a calibrated predictive distribution, so it is not directly comparable to [chronos](./chronos.md), [lag-llama](./lag-llama.md), or [moirai](./moirai.md) on [CRPS](../evaluation/metrics.md#21-crps--continuous-ranked-probability-score) / [WQL](../evaluation/metrics.md#23-wql--weighted-quantile-loss).
- Parameter sweep tops out at 50M, much smaller than contemporaneous [timesfm](./timesfm.md) (200M), [chronos](./chronos.md) (710M), or [moirai](./moirai.md) (311M); Timer's scaling story is convincing qualitatively but does not establish where the log-linear trend breaks.
- UTSD-12G is comfortably inside the [Monash](../datasets-benchmarks/monash-archive.md)/PEMS/ETT axis of public data, so benchmark overlap with downstream evaluation is likely non-negligible; the paper does not audit this explicitly — see [../research/reproducibility.md](../research/reproducibility.md).
- No explicit probabilistic head, no covariate handling, no in-context learning across related series; these are limitations the paper acknowledges and are all addressed in [timer-xl](./timer-xl.md) or competing lines.

## Follow-up work and dialogue
[timer-xl](./timer-xl.md) is the direct successor from the same Tsinghua group and generalizes Timer's S3 univariate sequence formulation to multivariate any-to-any prediction with a 2D attention layout, explicitly citing Timer's channel-independence as the motivation. [time-moe](./time-moe.md) takes the same decoder-only generative recipe and introduces mixture-of-experts scaling, following the same philosophy as [moirai-moe](./moirai-moe.md). [timesfm](./timesfm.md) shares the GPT-style patched decoder design but uses an asymmetric input/output patch trick and a different corpus (Google Trends / Wiki Pageviews); [sundial](./sundial.md) reuses Timer's backbone but replaces the point-prediction head with a flow-matching continuous distribution. On the probabilistic axis, [chronos](./chronos.md) and [lag-llama](./lag-llama.md) stake out the quantized-vocabulary and Student-t-head alternatives.

## Reproducibility
- **Open weights:** yes — released at `thuml/Large-Time-Series-Model`.
- **Code:** public at `thuml/Large-Time-Series-Model`, referenced in the paper.
- **Training data:** fully public — UTSD is released at `huggingface.co/datasets/thuml/UTSD` in hierarchical capacities.
- **Compute to retrain:** not fully disclosed; scaling runs use UTSD-4G and vary layer count and model dimension, but total GPU-hours and token counts are not tabulated in the main text.
- **Deployment footprint:** 1M-50M parameters across the scaling sweep (reported sizes far smaller than TimesFM/Chronos); segment length S=96 with flexible context under autoregressive rollout; inference demonstrated on standard GPUs.

## When to cite this paper
Cite Timer as the canonical reference for the Single-Series Sequence (S3) format and UTSD, and for the first clear demonstration that GPT-style next-token pretraining on a channel-independent univariate formulation yields a unified backbone usable across forecasting, imputation, and anomaly detection with strong few-shot behavior. For multivariate or probabilistic claims, prefer a successor such as [timer-xl](./timer-xl.md), [chronos](./chronos.md), or [sundial](./sundial.md).

## In the knowledge graph
- **Cluster:** [Decoder-only autoregressive TS-FMs](../foundation-models/taxonomy.md#cluster-1--decoder-only-autoregressive-ts-fms)
- **Architecture family:** [Decoder-only autoregressive](../architectures/decoder-only-autoregressive.md)
- **Related concepts:** [patch tokenization](../concepts/patch-tokenization.md), [multi-task universal](../concepts/multi-task-universal.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md), [in-context learning](../concepts/in-context-learning.md)
- **See also:** [timer-xl](./timer-xl.md), [timesfm](./timesfm.md), [time-moe](./time-moe.md)
