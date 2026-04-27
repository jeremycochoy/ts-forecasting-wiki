# Lag-Llama: Towards Foundation Models for Probabilistic Time Series Forecasting

> **Short name:** `lag-llama` · **arXiv:** [2310.08278](https://arxiv.org/abs/2310.08278) · **PDF:** [local](../../papers/lag-llama_2310.08278.pdf) · **Date:** 2023-10 · **Venue:** NeurIPS 2023 TSALM workshop

**Authors:** Kashif Rasul, Arjun Ashok, Andrew Robert Williams et al. (Morgan Stanley, ServiceNow, Mila, Universite de Montreal)

## Abstract
Lag-Llama is an early open decoder-only foundation model for univariate probabilistic time-series forecasting. It uses lagged values as covariates, concatenated with date-time features, outputs a Student-t predictive distribution, and includes an empirical study of neural scaling laws on the pretraining data axis.

## Key contributions
- Frequency-agnostic tokenization via a fixed "ladder" of lag indices chosen to cover common seasonalities (hourly, daily, weekly, monthly, quarterly, yearly), concatenated with date-time covariates at each step.
- First open decoder-only probabilistic TS foundation model with public weights on 27 datasets across six domains, roughly 352M data windows of training tokens.
- LLaMA-style decoder-only transformer backbone with RMSNorm and rotary embeddings.
- Student-t distribution head parameterized via softplus-transformed variance/degrees-of-freedom for robust probabilistic forecasts.
- Empirical scaling-law analysis with respect to pretraining corpus size, one of the first published TS-specific scaling signals.
- Freq-Mix and Freq-Mask data augmentation as part of the training pipeline.

## Architecture at a glance
Lag-Llama is a LLaMA-style decoder-only transformer. At each time step the input token is a vector of lagged values at a fixed set of lag offsets, concatenated with time-of-day / day-of-week / month covariates, then projected to the residual stream. A Student-t output head parameterizes the next-step predictive distribution; autoregressive rollout produces multi-step probabilistic forecasts. Series are standardized using robust median / IQR scaling, and the model is frequency-agnostic because the lag ladder is fixed rather than tailored per dataset. Lag-Llama uses lag features as covariate input instead of patch tokenization; forecasting is autoregressive 1-step-ahead with a Student-t distribution head (Rasul et al. Section 3).

## Why it matters
Lag-Llama established an open, reproducible baseline for probabilistic TS foundation models and provided one of the first published scaling-law studies for time-series pretraining. It set the "decoder-only, parametric distribution head" design pattern that [timer](./timer.md), [time-moe](./time-moe.md), and others later iterated on.

## Strengths
- Fully open pipeline with reproducible hyperparameter search: 100-config random search, 27-dataset corpus, and a single Nvidia Tesla P100 12GB GPU budget make the whole training run reproducible at near-laptop compute.
- Frequency-agnostic lag ladder is a clean alternative to [moirai](./moirai.md)'s multi-patch-size projections and [timesfm](./timesfm.md)'s frequency dictionary: the same model serves hourly, daily, and yearly data without per-frequency modules, and unseen frequency combinations degrade gracefully.
- The finetuned Lag-Llama reaches an average rank of 2.786 across seven unseen datasets vs 5.000 for TFT and 5.071 for N-BEATS (Table 1), beating all supervised baselines as a general-purpose model after light fine-tuning.
- Probabilistic from the ground up: Student-t output head makes Lag-Llama one of the few TS foundation models with natively calibrated forecast distributions, evaluated via [CRPS](../evaluation/metrics.md#21-crps--continuous-ranked-probability-score) rather than MSE.
- Scaling-law analysis on the data axis: the paper plots pretraining-corpus size vs. downstream performance and discusses diversity of the 27-dataset corpus, a rare empirical contribution for TS foundation models at the time.

## Limitations and open critiques
- Univariate only; the "lags as covariates" trick does not extend to exogenous variables or multivariate panels. [chronos-2](./chronos-2.md) and [timer-xl](./timer-xl.md) are the natural successors on that axis.
- The lag ladder is hand-designed and assumes a recognizable seasonal structure; highly irregular or sparse series (intermittent demand, clickstreams) are not the target regime.
- The pretraining corpus is 27 datasets with ~352M data windows, small compared to [timesfm](./timesfm.md)'s ~100B points or [moirai](./moirai.md)'s 27B [LOTSA](../datasets-benchmarks/lotsa.md) observations. Zero-shot performance is correspondingly modest and the paper leans on fine-tuning for its SoTA claims.
- Student-t head is parametric and unimodal; it cannot represent multi-modal predictive distributions that the quantized head of [chronos](./chronos.md) or the flow-matching head of [sundial](./sundial.md) can.
- Reported scaling is on a narrow model family and short horizons; no log-linear curve across orders of magnitude in parameters, and later models have dwarfed it in raw scale.

## Follow-up work and dialogue
Lag-Llama sits in the decoder-only probabilistic line together with [timesfm](./timesfm.md) (which drops the probabilistic head in favour of point forecasts and larger data), [timer](./timer.md) (which reuses the decoder-only generative objective on a larger UTSD corpus but also drops the distribution head), and [sundial](./sundial.md) (which replaces the Student-t head with a flow-matching continuous distribution). On the corpus side, [moirai](./moirai.md) and [chronos](./chronos.md) argue Lag-Llama's 27-dataset corpus is too small; [moment](./moment.md) offers a comparable open alternative with broader multi-task support. [chronos-2](./chronos-2.md) critiques the lag-ladder approach as insufficient for multivariate and covariate tasks. See [../concepts/probabilistic-forecasting.md](../concepts/probabilistic-forecasting.md) and [../concepts/scaling-laws.md](../concepts/scaling-laws.md).

## Reproducibility
- **Open weights:** yes — `time-series-foundation-models/Lag-Llama` on HuggingFace (the paper and repo release checkpoints).
- **Code:** public, referenced in the paper.
- **Training data:** partially public — the 27 datasets are assembled from public sources ([Monash](../datasets-benchmarks/monash-archive.md) and related archives) with documented splits.
- **Compute to retrain:** single Nvidia Tesla P100 12GB GPU with 4 CPU cores and 24GB RAM; batch size 256, learning rate 1e-4, 100 random-sampled windows per epoch with 50-epoch early stopping. This is perhaps the smallest reproducible compute footprint in the TS-FM literature.
- **Deployment footprint:** small decoder-only model (parameter count not tabulated in the extracted sections but well under 100M in the default configuration); autoregressive multi-step rollout; inference runs comfortably on CPU for short horizons.

## When to cite this paper
Cite Lag-Llama as the canonical reference for frequency-agnostic lag-feature tokenization plus a Student-t output head in a decoder-only probabilistic TS foundation model. It is also the right citation for the first open, reproducible pretrained probabilistic forecaster on a small multi-domain corpus, and for early TS-specific scaling-law evidence on the data axis. For larger-scale or multivariate claims, prefer [chronos](./chronos.md), [moirai](./moirai.md), or [chronos-2](./chronos-2.md).

## In the knowledge graph
- **Cluster:** [Decoder-only autoregressive TS-FMs](../foundation-models/taxonomy.md#cluster-1--decoder-only-autoregressive-ts-fms)
- **Architecture family:** [Decoder-only autoregressive](../architectures/decoder-only-autoregressive.md)
- **Related concepts:** [probabilistic forecasting](../concepts/probabilistic-forecasting.md), [scaling laws](../concepts/scaling-laws.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md)
- **See also:** [timesfm](./timesfm.md), [timer](./timer.md), [time-moe](./time-moe.md)
