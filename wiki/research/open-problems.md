# Open Problems in Time-Series Foundation Models

This is the flagship page for anyone looking for a research topic. Each
section states the problem, sketches what has been tried, says what is
still unresolved, and lists two or three concrete directions a
contributor could pick up. Entries are deliberately biased toward
questions where an individual researcher or a small team can still move
the needle — none of them require a thousand-GPU run.

The state of play is as of 2026-04. The picture shifts quickly, so when
you start a project, first check whether a recent paper has already
closed the gap; the [reading-roadmap.md](reading-roadmap.md) and
[comparison-matrix.md](comparison-matrix.md) are the fastest way to do
that.

## 1. Scaling laws for time-series

**Background.** In language modeling, the Chinchilla work gave a clean
statement of joint data-and-parameter optimality and a rule of thumb
that training-compute-optimal LLMs are undertrained. Nothing equivalent
is settled for TS-FMs. [Lag-Llama](../papers/lag-llama.md) published the first empirical scaling
curves for a probabilistic decoder-only TS-FM. Time-MoE demonstrated
that a 2.4B sparse MoE on [Time-300B](../datasets-benchmarks/time-300b.md) keeps improving, and [Chronos](../papers/chronos.md)
reported monotonic gains from 20M to 710M parameters. But the exponents
differ between reports, the datasets differ, and no paper yet argues
that it has found the compute-optimal frontier.

**What is still open.** Is there a Chinchilla-equivalent law for TS-FMs?
Does the optimal data-to-parameter ratio depend on whether pretraining
data is real ([LOTSA](../datasets-benchmarks/lotsa.md), Time-300B) or synthetic (Mamba4Cast-style PFN
priors, KernelSynth)? Does quantized-vocab training (Chronos) scale
differently from continuous-patch training ([TimesFM](../papers/timesfm.md), MOIRAI)? And what
is the right "tokens" unit — individual timesteps, patches, or
quantized codes?

**Directions a contributor could take.**
- Run a controlled three-axis sweep (params, data, context length) on a
  single corpus — LOTSA is the most defensible — and fit a joint law.
- Compare the fitted exponents for a decoder-only AR model, a masked
  encoder and a quantized-vocab T5, holding data fixed. This directly
  attacks the "does tokenization change the slope" question.
- Study whether synthetic-only pretraining has a fundamentally
  different scaling curve (e.g. saturation at low params) by reproducing
  Mamba4Cast at several sizes.

See [../concepts/scaling-laws.md](../concepts/scaling-laws.md).

## 2. True multivariate / covariate-informed forecasting

**Background.** Most TS-FMs were born univariate. MOIRAI's any-variate
attention and Timer-XL's flattened-token TimeAttention were the first
serious attempts to keep the channel dimension as a first-class citizen.
Chronos-2 pushed this further with group attention that lets the model
do in-context learning over related series at inference time. This is
arguably the most active frontier in 2025-2026: the best zero-shot
numbers on GIFT-Eval are now held by models that actively use related
series or exogenous variables, not by bigger univariate models.

**What has been tried.** Flattening channels into a single token stream
(Timer-XL), any-variate attention with learned channel identities
(MOIRAI), in-context grouping over related series (Chronos-2), and a
dedicated exogenous head on top of a lightweight backbone (TTM). Each
recipe handles some but not all of: irregular channel counts across
datasets, static covariates, known-future exogenous variables, and
cross-channel dependence.

**What is still open.** There is no agreed-upon interface for feeding a
model "here is the target, here are N related numerical series, here
are K static categorical covariates, here is a known-future calendar".
Every paper invents a slightly different scheme and then only evaluates
on datasets where that scheme is natural. The result is that we cannot
compare multivariate models cleanly.

**Directions.**
- Define and publish a benchmark protocol for covariate-informed
  forecasting with a fixed interface, and port three to five existing
  models to it.
- Probe whether any-variate attention actually learns cross-channel
  dependence or silently falls back to per-channel prediction
  (ablation: shuffle channels at inference).
- Extend Chronos-2's in-context group attention to irregular
  sampling and long contexts, the two cases its current training
  protocol does not stress.

See [../papers/chronos-2.md](../papers/chronos-2.md),
[../papers/timer-xl.md](../papers/timer-xl.md),
[../papers/moirai.md](../papers/moirai.md),
[../concepts/in-context-learning.md](../concepts/in-context-learning.md).

## 3. Probabilistic calibration under distribution shift

**Background.** TS-FMs now produce probabilistic forecasts in roughly
three styles: categorical sampling over a quantized vocab (Chronos),
mixture-of-Student-t output heads (MOIRAI, Lag-Llama), quantile
decoders (Chronos-2, TTM). Flow matching ([Sundial](../papers/sundial.md)) is a fourth,
generative path. All of them report competitive [CRPS](../evaluation/metrics.md#21-crps--continuous-ranked-probability-score) / [WQL](../evaluation/metrics.md#23-wql--weighted-quantile-loss) on
in-distribution splits. Very little is known about calibration when the
test series comes from a domain not represented in pretraining.

**What has been tried.** [TimeGPT-1](../papers/timegpt.md) wraps conformal prediction around a
closed model to provide post-hoc calibrated intervals. MOIRAI and
Lag-Llama report interval coverage but only on their own splits.
Evaluation suites like GIFT-Eval track probabilistic metrics
per-dataset, not per-shift-type.

**What is still open.** Which head family is most robust to scale
shifts, regime changes, and seasonality the model has never seen? Does
RevIN-style per-instance normalization actually help calibration, or
does it just move the miscalibration into a less visible place? Is
post-hoc conformal wrapping strictly preferable to training-time
calibration for TS-FMs, as some recent LLM work suggests?

**Directions.**
- Build a shift-aware evaluation harness (scale shift, trend reversal,
  frequency change, new seasonality) on top of GIFT-Eval and re-score
  the four head families across the same backbone.
- Study whether conformal wrapping of a flow-matching TS-FM (Sundial)
  gives useful intervals, since the current Sundial paper does not.
- Reproduce the Student-t mixture output on a quantized-vocab backbone
  and ask whether the gain reported in MOIRAI transfers.

See [../concepts/probabilistic-forecasting.md](../concepts/probabilistic-forecasting.md).

## 4. Data — pretraining corpora are small, synthetic is unproven

**Background.** Every TS-FM paper opens with a variant of the same
sentence: "unlike NLP, the time-series community does not have an
internet-scale corpus." Time-300B is the largest published corpus at
the moment; LOTSA and the [Time Series Pile](../datasets-benchmarks/time-series-pile.md) are smaller. Even the
largest is orders of magnitude below modern LLM training token counts,
and the diversity is concentrated in a few high-frequency domains
(energy, web traffic, finance).

One reaction is synthetic-only pretraining: Mamba4Cast uses a
PFN-style prior to sample synthetic series and never touches real data,
and Chronos's KernelSynth augments a real corpus with Gaussian-process
synthetic draws. So far this is promising on small and medium
benchmarks but has not been pushed to the scale at which LLMs found
their Chinchilla curve.

**What is still open.** How far can synthetic-only training go? Which
real-data types matter most (i.e. if you had to drop one LOTSA cluster,
which would hurt least)? Can we get a "web-scale" TS corpus from
something like Common Crawl's embedded tables and sensor logs, and
would quality filtering look anything like LLM quality filtering?

**Directions.**
- Push Mamba4Cast or a similar PFN model to billion-parameter scale on
  pure synthetic data and compare to Time-MoE head-to-head on
  GIFT-Eval.
- Run a LOTSA-ablation study: drop one domain at a time and measure
  zero-shot regression.
- Build and release a new corpus (the "LOTSA-2") from a fresh domain
  source; this is low-glamour but high-value work.

See [../datasets-benchmarks/datasets-benchmarks.md](../datasets-benchmarks/datasets-benchmarks.md),
[../concepts/synthetic-data-augmentation.md](../concepts/synthetic-data-augmentation.md).

## 5. Efficient inference — when is small actually right?

**Background.** TTM showed that a 1-5M-parameter MLP-Mixer with
adaptive patching is competitive on many forecasting benchmarks and
runs on CPU. Mamba4Cast uses a state-space backbone for linear-time
inference. On the other end, Time-MoE ships 2.4B parameters and
Moirai-MoE routes tokens through a sparse expert pool to amortize
compute. There is no principled story about which regime fits which
deployment.

**What is still open.** Is the gap between TTM and Time-MoE a capacity
gap, a data gap, or a task-shape gap? At what horizon, at what channel
count, at what required accuracy, does the billion-parameter model
start paying for itself? Does sparse MoE actually provide a Pareto
improvement at inference, or is the reported speedup a training-side
number?

**Directions.**
- Measure a Pareto front: latency / memory / CRPS across TTM,
  Mamba4Cast, Chronos-small, MOIRAI-base, Time-MoE, Moirai-MoE, on one
  benchmark.
- Distill a large MoE into a TTM-sized student and see how much of the
  gap closes.
- Characterize when the MoE router actually fires diverse experts
  versus collapsing to one — an open question inherited from the LLM
  MoE literature.

See [../papers/ttm.md](../papers/ttm.md),
[../papers/mamba4cast.md](../papers/mamba4cast.md),
[../papers/time-moe.md](../papers/time-moe.md),
[../papers/moirai-moe.md](../papers/moirai-moe.md).

## 6. Long-horizon forecasting and autoregressive rollout error

**Background.** Decoder-only AR TS-FMs (TimesFM, [Timer](../papers/timer.md), Time-MoE,
Lag-Llama) generate long horizons by rolling out their next-patch
prediction repeatedly. Error compounds, and most papers paper over this
by training with an output-patch size larger than the input-patch size
(TimesFM's trick) or by masking large contiguous blocks during training
(MOIRAI, [MOMENT](../papers/moment.md)). Sundial's flow-matching head produces the whole
horizon in one pass and sidesteps the rollout entirely; Mamba4Cast
similarly does single-pass horizon generation.

**What is still open.** On horizons of 1000+ steps, do single-pass
models strictly beat AR rollouts? Can we get the horizon-degradation
curve for every major TS-FM on the same benchmark? Is scheduled
sampling or teacher-forcing-free training the right fix for the AR
side?

**Directions.**
- Build a horizon-stress benchmark (same context, increasing horizon)
  and measure error growth across AR and single-pass models.
- Retrain a decoder-only model with scheduled sampling or exposure-bias
  correction and measure rollout improvement in isolation.
- Try hybrid decoders: autoregressive for the first N patches, flow
  matching for the tail.

## 7. Tokenization — no consensus

**Background.** The field has converged on "tokenize the series
somehow" but not on how. Patch embeddings (TimesFM, MOIRAI, MOMENT,
Chronos-2), value quantization to a fixed vocab (Chronos), numbers as
text ([LLMTime](../papers/llmtime.md)), VQ-VAE codebooks ([TOTEM](../papers/totem.md)), continuous flows (Sundial)
and flattened multivariate tokens (Timer-XL) all coexist. Each is
motivated by a different downstream need (probabilistic output,
multi-resolution handling, reuse of an LLM tokenizer, cross-domain
sharing).

**What is still open.** Nobody has a fair head-to-head between these
schemes with backbone and data held constant. The Chronos paper argues
quantization is fine; the TimesFM paper argues it is lossy; LLMTime
argues a text tokenizer is sufficient for zero-shot. None of these
claims have been cross-checked by a neutral team.

**Directions.**
- Fix the T5 backbone (to match Chronos) and swap tokenizers
  (quantized vocab, patch, VQ-VAE, text) on the same corpus; report
  CRPS, WQL, compute.
- Measure whether VQ codebooks transfer across domains (as TOTEM
  claims) by holding one domain out.
- Investigate hybrid tokenization: coarse-grained VQ for long-range
  context, fine continuous patches for short-range.

See [../concepts/patch-tokenization.md](../concepts/patch-tokenization.md),
[../concepts/value-quantization.md](../concepts/value-quantization.md).

## 8. Evaluation and benchmark contamination

**Background.** Zero-shot claims are only as strong as the guarantee
that the test series never appeared in pretraining. LOTSA, the Time
Series Pile and Time-300B all overlap with [Monash](../datasets-benchmarks/monash-archive.md) and with the long-horizon
benchmarks used by the Autoformer / FEDformer era. Several papers
report "zero-shot" numbers on datasets that are demonstrably in their
training corpus. GIFT-Eval was introduced partly as a cleaner protocol,
but not every paper reports on it, and different papers quote
different splits.

**What is still open.** We do not have a community-agreed procedure
for proving a dataset is held out. Paper-reported zero-shot numbers are
therefore not directly comparable. Worse, the most popular benchmarks
(ETT, Weather, Traffic) have been used so aggressively that the
informative signal has eroded.

**Directions.**
- Write a leakage audit: for each of the 19 papers, check which
  benchmarks are trivially in-pretraining-corpus, and publish the
  table.
- Propose a cryptographic-style train-test hash protocol that a paper
  can adopt to credibly claim zero-shot on a dataset.
- Replace the legacy long-horizon benchmarks with clean holdouts from
  LOTSA-2 or a fresh corpus.

See [../datasets-benchmarks/gift-eval.md](../datasets-benchmarks/gift-eval.md).

## 9. Interpretability and anomaly attribution

**Background.** TS-FMs inherit the usual transformer-interpretability
toolkit (attention maps, probing classifiers), but time-series
forecasts have natural semantic handles — seasonality, trend, level —
that text does not. MOMENT is one of the few models to be evaluated on
anomaly detection as a first-class task. Very little work tries to
attribute "the model predicted this peak because of that historical
pattern".

**What is still open.** Can we decompose a TS-FM forecast into
attributions over (context position, channel, frequency band)? Do
patch-tokenized and quantized-vocab models yield fundamentally
different interpretability tools? Is the in-context attention in
Chronos-2 a natural attribution surface for "which related series drove
this prediction"?

**Directions.**
- Port LLM-style circuit analysis to a small TS-FM (TTM or
  Mamba4Cast) and look for heads that isolate trend vs seasonality.
- Build an anomaly-attribution benchmark and score the handful of
  TS-FMs that claim anomaly support.
- Study whether flow-matching models (Sundial) produce naturally
  calibrated uncertainty that can drive anomaly scores.

## 10. Latent-space pretraining vs forecast-loss pretraining

**Background.** [Cluster 8](../foundation-models/taxonomy.md#cluster-8--jepa--latent-space-prediction)
— [LaT-PFN](../papers/lat-pfn.md), [TS-JEPA](../papers/ts-jepa.md),
[MTS-JEPA](../papers/mts-jepa.md) — replaces input-space
reconstruction with latent-space prediction under an EMA target
encoder, and reports gains on classification and anomaly prediction
benchmarks. The forecasting-first TS-FMs ([TimesFM](../papers/timesfm.md),
[Chronos](../papers/chronos.md), [MOIRAI](../papers/moirai.md),
[Sundial](../papers/sundial.md)) optimize a forecasting loss
directly. Whether the JEPA recipe wins, loses, or ties on Monash /
GIFT-Eval / Chronos Benchmark II at matched compute is the single
biggest open question for Cluster 8 and arguably for the whole
self-supervised TS literature.

**What has been tried.** TS-JEPA gives a small-scale matched-capacity
ablation against MAE, TS2Vec, and autoregressive (paper Table 1 / 2,
single V100), but only on 5 classification and 3 forecasting datasets
and only with a transformer of 2 heads / 128 dim. MTS-JEPA shows that
on continuous TS the EMA + stop-gradient bookkeeping is sometimes
insufficient to prevent representation collapse, and adds a soft
codebook bottleneck as backup (paper Appendix A.3 derives an
analytical non-collapse bound).

**What is still open.** Does latent-space prediction scale to
billion parameters the way [Time-MoE](../papers/time-moe.md) and
[Timer-S1](../papers/timer-s1.md) do? Can a JEPA-pretrained backbone
beat a forecast-loss-pretrained backbone on GIFT-Eval after a
forecasting-head finetune? What is the right asymmetry between
encoder and predictor capacity for time series — I-JEPA uses small
predictors, the TS papers use roughly symmetric architectures? And
is the codebook bottleneck specific to anomaly prediction's
discrete-regime structure, or is it a general stability fix for
JEPA on continuous TS?

**Directions.**
- Pretrain a 1B-parameter JEPA backbone on [LOTSA](../datasets-benchmarks/lotsa.md) or
  [TimeBench](../datasets-benchmarks/timebench.md), attach a Chronos-2-style quantile decoder, and
  compare against Moirai-2 / Timer-S1 head-to-head on GIFT-Eval.
- Replicate MTS-JEPA's codebook ablation on TS-JEPA's classification
  and forecasting benchmarks: is the codebook strictly necessary or
  is the EMA decay schedule enough?
- Sweep encoder/predictor size ratio at fixed total params, on a
  TS-JEPA-style classification benchmark, to fix the right asymmetry.
- Run a head-to-head latent-loss vs MAE-loss vs InfoNCE-loss
  experiment on a single backbone and corpus, to settle which of the
  three non-reconstructive self-supervision families is best for TS.

## Related wiki pages

- [reading-roadmap.md](reading-roadmap.md)
- [comparison-matrix.md](comparison-matrix.md)
- [../foundation-models/taxonomy.md](../foundation-models/taxonomy.md)
