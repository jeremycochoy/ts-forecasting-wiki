# Glossary

Compact definitions for terms used across the wiki. One or two
sentences each, with cross-links to deeper pages where one exists.

- **Ablation.** A controlled experiment that removes a single component
  of a model to isolate its contribution. The gold standard of
  evidence in TS-FM papers.
- **Autoregressive rollout.** Generating a long horizon by feeding the
  model's own previous predictions back in as inputs. Compounds error
  with horizon length; see
  [open-problems.md](open-problems.md#6-long-horizon-forecasting-and-autoregressive-rollout-error).
- **Chinchilla.** The 2022 DeepMind scaling-law paper that argued
  compute-optimal LLMs are undertrained. No TS-FM equivalent is
  settled.
- **Context window.** The number of past timesteps (or patches) the
  model sees before forecasting.
- **Covariate.** An auxiliary variable associated with the target
  series (temperature alongside energy load, say). Static covariates
  are time-invariant; dynamic covariates change with time.
- **CRPS (Continuous Ranked Probability Score).** A proper scoring rule
  for probabilistic forecasts; lower is better. The de facto standard
  metric for TS-FMs with distributional output. See
  [../concepts/probabilistic-forecasting.md](../concepts/probabilistic-forecasting.md)
  and the full definition in
  [../evaluation/metrics.md#21-crps--continuous-ranked-probability-score](../evaluation/metrics.md#21-crps--continuous-ranked-probability-score).
- **Decoder-only.** A transformer trained to predict the next token
  given its left context, without a separate encoder. The GPT
  recipe — and the pattern used by TimesFM, Timer, Time-MoE and
  Lag-Llama.
- **Encoder-only.** A transformer whose training signal is over the
  whole input, typically via masked reconstruction. MOMENT, MOIRAI
  and Chronos-2 are encoder-only.
- **Exogenous variable.** A covariate that is known into the forecast
  horizon (a calendar, a pricing schedule). TTM explicitly supports
  exogenous heads.
- **Few-shot.** Fine-tuning or adapting a pretrained model on a small
  number of labeled examples from a new task.
- **Fine-tune.** Continuing training of a pretrained model on a target
  dataset, usually with a lower learning rate.
- **Flow matching.** A generative training objective that learns a
  continuous-time velocity field between noise and data. Used as the
  output head of Sundial. See [../architectures/flow-matching-continuous.md](../architectures/flow-matching-continuous.md).
- **Gaussian process (GP).** A non-parametric probabilistic model over
  functions. Used in Chronos's KernelSynth to generate synthetic
  training series.
- **GIFT-Eval.** A recent evaluation suite for TS-FMs designed to be
  cleaner than the legacy long-horizon benchmarks. See
  [../datasets-benchmarks/gift-eval.md](../datasets-benchmarks/gift-eval.md).
- **Horizon.** The number of future timesteps to predict.
- **In-context learning.** The ability of a pretrained model to adapt
  at inference time from examples provided in its context, without
  gradient updates. Chronos-2 pushes this for TS. See
  [../concepts/in-context-learning.md](../concepts/in-context-learning.md).
- **LoRA.** Low-rank adaptation: a parameter-efficient fine-tuning
  scheme that trains a small low-rank delta on top of frozen weights.
- **LOTSA.** Large-scale Open Time Series Archive, ~27B observations,
  assembled by the MOIRAI team. See
  [../datasets-benchmarks/lotsa.md](../datasets-benchmarks/lotsa.md).
- **Mamba.** A selective state-space model architecture with linear
  scaling in sequence length. The backbone of Mamba4Cast.
- **MASE (Mean Absolute Scaled Error).** A scale-free point forecast
  metric; standard on the Monash archive. Full definition and pitfalls
  in [../evaluation/metrics.md#17-mase--mean-absolute-scaled-error](../evaluation/metrics.md#17-mase--mean-absolute-scaled-error).
- **Masked reconstruction.** A pretraining objective in which parts of
  the input are hidden and the model is trained to reconstruct them.
  The MOMENT, MOIRAI and Moirai-MoE objective.
- **MoE (Mixture of Experts).** A sparse architecture that routes each
  token through a small subset of expert subnetworks. Time-MoE and
  Moirai-MoE are the two TS-FM examples.
- **Monash archive.** A long-standing collection of univariate
  forecasting benchmarks. See
  [../datasets-benchmarks/monash-archive.md](../datasets-benchmarks/monash-archive.md).
- **Multivariate.** A series with multiple channels that may interact.
  Distinct from "many univariate series".
- **Panel data.** A collection of related series (one per entity)
  observed over the same time window. Natural home for in-context
  learning over related series.
- **Patch.** A contiguous window of raw timesteps projected into a
  single token. See
  [../concepts/patch-tokenization.md](../concepts/patch-tokenization.md).
- **PFN (Prior-Fitted Network).** A model trained only on samples
  from a Bayesian prior, so that its forward pass approximates
  posterior inference. Mamba4Cast is the TS PFN.
- **Probabilistic forecast.** A forecast that outputs a predictive
  distribution (samples, quantiles, or a parametric family) rather
  than a single point. See
  [../concepts/probabilistic-forecasting.md](../concepts/probabilistic-forecasting.md).
- **Quantile loss.** The pinball loss used to train a quantile
  regression head. Minimized per quantile level. Full definition in
  [../evaluation/metrics.md#22-pinball--quantile-loss](../evaluation/metrics.md#22-pinball--quantile-loss).
- **RevIN.** Reversible instance normalization: subtract and rescale
  per-instance statistics before the model and reverse after. Used by
  MOMENT. See [../concepts/revin-normalization.md](../concepts/revin-normalization.md).
- **Scaling law.** An empirical power-law relationship between model
  size, data size, compute and loss. See
  [../concepts/scaling-laws.md](../concepts/scaling-laws.md).
- **State-space model (SSM).** A recurrent family with linear
  dynamics in a latent state; Mamba is a modern selective SSM.
- **Student-t head.** A probabilistic output head parameterizing a
  Student-t distribution (or a mixture thereof). Used by MOIRAI and
  Lag-Llama.
- **Synthetic data augmentation.** Generating new training series from
  a prior (KernelSynth, PFN priors, TSMix). See
  [../concepts/synthetic-data-augmentation.md](../concepts/synthetic-data-augmentation.md).
- **T5.** An encoder-decoder text-to-text transformer; the backbone
  Chronos reuses unchanged and MOMENT initializes from.
- **Teacher forcing.** Training an autoregressive model with the
  ground-truth previous token as input, rather than its own prediction.
- **Time Series Pile.** The corpus bundled with MOMENT. See
  [../datasets-benchmarks/time-series-pile.md](../datasets-benchmarks/time-series-pile.md).
- **Time-300B.** The ~300B-point corpus assembled for Time-MoE.
  Currently the largest published TS pretraining corpus. See
  [../datasets-benchmarks/time-300b.md](../datasets-benchmarks/time-300b.md).
- **TimeBench.** The ~1T-point corpus used by Sundial. See
  [../datasets-benchmarks/timebench.md](../datasets-benchmarks/timebench.md).
- **Value quantization.** Binning continuous values into a fixed
  vocabulary so that an unmodified text transformer can consume them.
  The Chronos recipe. See
  [../concepts/value-quantization.md](../concepts/value-quantization.md).
- **VQ-VAE.** Vector-Quantized Variational Autoencoder: learns a
  discrete codebook over a continuous input. Used as the tokenizer by
  TOTEM.
- **WQL (Weighted Quantile Loss).** A weighted version of the
  pinball loss used in probabilistic TS-FM leaderboards, notably by
  Chronos and Chronos-2. Full definition in
  [../evaluation/metrics.md#23-wql--weighted-quantile-loss](../evaluation/metrics.md#23-wql--weighted-quantile-loss).
- **Zero-shot.** Using a pretrained model on a new dataset without any
  fine-tuning at all. See
  [../concepts/zero-shot-forecasting.md](../concepts/zero-shot-forecasting.md).

## Related wiki pages

- [../concepts/README.md](../concepts/README.md)
- [../datasets-benchmarks/README.md](../datasets-benchmarks/README.md)
- [comparison-matrix.md](comparison-matrix.md)
