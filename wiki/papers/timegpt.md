# TimeGPT-1

> **Short name:** `timegpt` · **arXiv:** [2310.03589](https://arxiv.org/abs/2310.03589) · **PDF:** [local](../../papers/timegpt_2310.03589.pdf) · **Date:** 2023-10 · **Venue:** preprint

**Authors:** Azul Garza, Cristian Challu, Max Mergenthaler-Canseco (Nixtla)

## Abstract
TimeGPT-1 is presented as the first end-to-end commercial time-series foundation model delivered as an API. It is trained on a large multi-domain corpus of over 100 billion data points and produces zero-shot forecasts along with conformal-prediction uncertainty intervals, outperforming classical, ML, and deep-learning baselines across a test set of more than 300,000 unseen time series.

## Key contributions
- First commercially deployed TS foundation model exposed through a Python SDK and REST API.
- Claimed training corpus of "the largest collection of publicly available time series", encompassing over 100 billion data points across finance, economics, demographics, healthcare, weather, IoT, energy, web traffic, sales, transport, and banking.
- Transformer encoder-decoder backbone with CNN-augmented input and output blocks and local positional encoding, explicitly not initialized from an LLM.
- Conformal prediction for distribution-free uncertainty intervals on arbitrary downstream series.
- Popularized the "time-series foundation model" framing among practitioners.

## Architecture at a glance
TimeGPT is described as a transformer encoder-decoder with self-attention blocks, CNN layers in the input/output pipelines, residual connections, and layer normalization, producing a fixed-length forecast via a linear output head. Conformal prediction on rolling historical errors supplies prediction intervals at inference time without needing a parametric distribution head. The system accepts arbitrary univariate or multivariate histories and exogenous regressors through the API. Exact architecture sizes and training hyperparameters are not disclosed.

## Why it matters
TimeGPT demonstrated demand and feasibility for a production-grade zero-shot forecasting service and introduced the vocabulary of "TS foundation models" to the broader practitioner community, setting the stage for the open research wave that followed.

## Strengths
- First paper to take the foundation-model-as-API framing seriously for time series and to ship it as a product, drastically lowering the bar for practitioners who had previously needed to train DeepAR or N-BEATS per dataset.
- Large-scale evaluation on over 300,000 unseen test series spanning monthly, weekly, daily, and hourly frequencies; across Table 1 TimeGPT ranks first or second against a broad baseline set (Theta, ETS, CES, LightGBM, LSTM, DeepAR, TFT, N-HiTS) on rMAE and rRMSE.
- Conformal prediction sidesteps the need to commit to a parametric distribution head like Student-t ([lag-llama](./lag-llama.md)) or a quantized categorical ([chronos](./chronos.md)); it gives calibrated intervals without retraining, at the cost of needing historical residuals at inference.
- Inference throughput is reported at 0.6 ms per series on GPU, comparable to Seasonal Naive and orders of magnitude faster than AutoARIMA or iteratively-trained global models, demonstrating that a single-forward-pass foundation model can replace heavy per-dataset training pipelines in production.
- Supports exogenous variables and calendar features at inference, which is a practical strength over contemporaneous [timesfm](./timesfm.md) and [chronos](./chronos.md) v1 which left covariates for follow-up.

## Limitations and open critiques
- Closed source: weights, training data, exact architecture, parameter count, and training compute are all undisclosed. Scientific reproducibility is essentially zero — see [../research/reproducibility.md](../research/reproducibility.md). Later papers can only cite TimeGPT as a commercial reference, not a baseline they can retrain or audit.
- The "100B data points" training corpus is described only in prose, with no public dataset release and no breakdown per domain or frequency, so claims of diversity cannot be independently verified.
- No head-to-head comparison with other foundation models in the paper itself: Section 6.1.1 ("Comparison with recent Foundation Models") is explicitly marked "Work in progress" in the preprint, so TimeGPT is never directly benchmarked against [chronos](./chronos.md), [moirai](./moirai.md), [timesfm](./timesfm.md), or [lag-llama](./lag-llama.md) inside the paper.
- Conformal prediction requires enough in-sample historical residuals at inference; for very short context or cold-start series the uncertainty quantification is weak.
- Multivariate support and exogenous inputs are exposed through the API but the paper does not describe how they are encoded inside the transformer, and there is no ablation showing they actually help.
- The paper leans heavily on "scaling laws" prose without providing a single scaling curve or ablation; most quantitative evidence is a single rMAE/rRMSE table per frequency.

## Follow-up work and dialogue
TimeGPT is the commercial anchor of the TS foundation model wave; it predates and frames the open-research response represented by [chronos](./chronos.md), [timesfm](./timesfm.md), [moirai](./moirai.md), [moment](./moment.md), and [lag-llama](./lag-llama.md). Each of these takes a different stance on the reproducibility gap TimeGPT opened: Chronos and Moirai release weights and corpora, TimesFM releases weights, Lag-Llama releases everything at small scale, and MOMENT releases the full [Time Series Pile](../datasets-benchmarks/time-series-pile.md). On the uncertainty side, [lag-llama](./lag-llama.md) argues for parametric Student-t heads, [chronos](./chronos.md) for quantized categorical heads, and [sundial](./sundial.md) for flow-matching continuous distributions — in contrast with TimeGPT's post-hoc conformal approach. On the benchmark hygiene side, the size and composition of TimeGPT's 100B-point training corpus cannot be audited, so any zero-shot claim is weakened by the possibility of undisclosed overlap; see [../benchmarks/state-of-the-art.md](../benchmarks/state-of-the-art.md).

## Reproducibility
- **Open weights:** no.
- **Code:** no; only a Python SDK (`nixtla`) and REST API.
- **Training data:** proprietary — described as "largest collection of publicly available time series (>100B data points)" but neither the exact source list nor the splits are disclosed.
- **Compute to retrain:** "multi-day training period on a cluster of NVIDIA A10G GPUs"; exact GPU-hours, parameter counts, and FLOPs are not disclosed.
- **Deployment footprint:** parameter count not disclosed; inference exposed as an API (`nixtla.forecast`), reported at 0.6 ms per series on GPU in internal tests; no CPU benchmark reported, and no public download of the model.

## When to cite this paper
Cite TimeGPT as the canonical reference for the first commercially-deployed time-series foundation model delivered as an API, for the "foundation model + conformal prediction" uncertainty recipe, and for popularizing the TS-FM framing to industry. For any claim about model architecture, training corpus, or reproducible scaling behavior, prefer an open paper such as [chronos](./chronos.md), [timesfm](./timesfm.md), [moirai](./moirai.md), or [lag-llama](./lag-llama.md).

## In the knowledge graph
- **Cluster:** [Masked-encoder / encoder-decoder TS-FMs](../foundation-models/taxonomy.md#cluster-2--masked-encoder--encoder-decoder-ts-fms)
- **Architecture family:** [Encoder-decoder T5](../architectures/encoder-decoder-t5.md)
- **Related concepts:** [zero-shot forecasting](../concepts/zero-shot-forecasting.md), [probabilistic forecasting](../concepts/probabilistic-forecasting.md)
- **See also:** [chronos](./chronos.md), [moirai](./moirai.md), [timesfm](./timesfm.md)
