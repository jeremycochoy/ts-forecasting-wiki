# Mamba4Cast: Efficient Zero-Shot Time Series Forecasting with State Space Models

> **Short name:** `mamba4cast` · **arXiv:** [2410.09385](https://arxiv.org/abs/2410.09385) · **PDF:** [local](../../papers/mamba4cast_2410.09385.pdf) · **Date:** 2024-10 · **Venue:** preprint (NeurIPS 2024 TSALM workshop)

**Authors:** Sathya Kamesh Bhethanabhotla, Omar Swelam, Julien Siems, David Salinas, Frank Hutter (University of Freiburg / ELLIS)

## Abstract
Mamba4Cast replaces transformer backbones with a Mamba-2 selective state-space model and trains it entirely on synthetic data in the Prior-data Fitted Network (PFN) style. The resulting zero-shot forecaster produces a full horizon in a single forward pass and scales gracefully with prediction length, matching Chronos-Base on 17 real-world datasets while being orders of magnitude faster at long horizons.

## Key contributions
- Mamba-2 selective state-space model as a non-transformer backbone for zero-shot time-series forecasting.
- Purely synthetic pretraining via Prior-data Fitted Networks (PFN): 70% Gaussian Process priors and 30% ForecastPFN priors, with no real-data exposure.
- Single-pass direct forecast: the model predicts the entire horizon in one forward pass rather than autoregressive rollout, decoupling inference latency from horizon length.
- Competitive [MASE](../evaluation/metrics.md#17-mase--mean-absolute-scaled-error) versus Chronos-Base/Small, DeepAR, AutoARIMA, and seasonal-naive across 17 datasets, with the efficiency gap widening as prediction length grows.
- Ablations on convolution layers, prior composition, and inference strategy.

## Architecture at a glance
Mamba4Cast consists of four components: (1) min-max scaled inputs with time-feature positional encoding, (2) an embedding stack of stacked causal Conv1d + GELU layers for input values and positional features, (3) an encoder of Mamba-2 blocks interleaved with LayerNorm and another dilated convolution block, (4) a linear decoder that maps the final hidden states to a full horizon of point forecasts. Training is on synthetic sequences of length uniformly sampled in [30, 512] with prediction length uniformly sampled in [10, 60].

## Why it matters
Mamba4Cast demonstrates that competitive zero-shot TS forecasting is achievable without transformers and without real-data pretraining. It combines two independent design choices — linear-time SSM backbone and PFN-style synthetic training — and shows they compose into a favourable accuracy-latency trade-off, especially at long horizons where transformer-based autoregressive decoding is penalised.

## Strengths
- Linear-time state-space recurrence means inference cost scales linearly in prediction length, while autoregressive transformers scale super-linearly. Figure 2 of the paper shows the latency gap over Chronos-Small widens rapidly as horizon grows from 8 to 50 steps.
- No real-data exposure during pretraining: the model is trained only on GP and ForecastPFN priors, which sidesteps licensing, data-contamination, and leakage concerns that affect many TS foundation models.
- Single-pass horizon generation removes autoregressive drift and eliminates rollout error accumulation.
- Training is cheap enough to fit on a single RTX 2080 Ti in ~3 days (420k batches × 64 samples), making the approach accessible to research labs rather than requiring a compute cluster.
- Mamba4Cast is close to or at the top of the critical-difference diagram on 17 benchmark datasets, beating Chronos-Small and DeepAR in mean rank.

## Limitations and open critiques
- Strictly univariate: the paper explicitly acknowledges that "Mamba4Cast is limited to the univariate domain, which only forms a small portion of real time series problems."
- Heavy reliance on prior diversity: the model's quality is bounded by how well the GP and ForecastPFN priors cover real-world TS distributions, and the paper notes that the results depend on the kernel composition.
- Point forecasts only: Mamba4Cast outputs a single horizon trajectory per window and has no built-in probabilistic head, while Chronos (its main baseline) provides sampled predictive quantiles natively.
- Trained on short contexts (up to 512 steps); long-context generalisation of Mamba-2 for forecasting is not stress-tested as deeply as in transformer TS-FMs.
- The model and training protocol are small compared to frontier TS-FMs; whether the recipe scales to Time-MoE or [Sundial](./sundial.md) capacity is an open question.

## Follow-up work and dialogue
Mamba4Cast is the natural counterpart to [time-moe](./time-moe.md) in the efficiency versus capacity debate: SSM + synthetic-only training vs. MoE + large-scale real data. The latency argument goes in Mamba4Cast's favour for long horizons, while the absolute accuracy on real-world benchmarks still trails the largest real-data TS-FMs. Within the lightweight/non-transformer cluster, [TTM](./ttm.md) is the other reference point — TTM also avoids transformers and targets CPU inference, but trains on real public data rather than synthetic priors. [Lag-Llama](./lag-llama.md) sits between the two philosophies as a transformer trained on real data with lag features. Mamba4Cast's synthetic-only approach inherits from ForecastPFN and extends the PFN-for-forecasting line to SSM backbones.

## Reproducibility
- **Open weights:** yes — the paper states pretrained weights and code are released.
- **Code:** public repo stated as `https://github.com/automl/Mamba4Cast`.
- **Training data:** fully synthetic, generated on-the-fly from Gaussian Process priors (70%) and ForecastPFN priors (30%); no real-data corpus is used for training.
- **Compute to retrain:** 420,000 batches of size 64, cosine-annealed LR 1e-5 to 1e-7, training for ~3 days on a single Nvidia RTX 2080 Ti, plus 60k additional rounds with a different kernel composition.
- **Deployment footprint:** Mamba-2 encoder plus convolutional embedding and a linear head; parameter count is small enough for single-GPU inference. The paper's Figure 2 demonstrates multi-second-scale inference for batch sizes up to 128 on reasonable hardware.

## When to cite this paper
Cite Mamba4Cast as the canonical reference for a Mamba/SSM backbone in zero-shot time-series forecasting, for the combination of PFN-style synthetic pretraining with state-space models, and as the primary efficiency data point for long-horizon inference cost relative to autoregressive transformer TS-FMs.

## In the knowledge graph
- **Cluster:** [Lightweight / non-transformer FMs](../foundation-models/taxonomy.md#cluster-5--lightweight--non-transformer-fms)
- **Architecture family:** [Lightweight non-transformer](../architectures/lightweight-non-transformer.md)
- **Related concepts:** [synthetic data augmentation](../concepts/synthetic-data-augmentation.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md), [in-context learning](../concepts/in-context-learning.md)
- **See also:** [ttm](./ttm.md), [timesfm](./timesfm.md), [time-moe](./time-moe.md), [lag-llama](./lag-llama.md), [chronos](./chronos.md)
