# TS-FM Timeline, 2023–2026

A chronological walk through the twenty papers in this wiki. Each
entry names the paper, the month it first appeared on arXiv, its
cluster in the [taxonomy](../foundation-models/taxonomy.md), and
the one contribution it is most frequently cited for. The purpose
is orientation — for detail, open the paper leaf linked at the end
of each line.

Dates below are arXiv-metadata dates; several papers were later
published at ICML / NeurIPS / ICLR (see
[papers.md](../papers/papers.md) for venues).

## 2023

- **Feb 2023 — [GPT4TS / OFA](../papers/gpt4ts.md) (Cluster 4).** "One
  fits all": freeze GPT-2 (or BERT / BEiT), fine-tune only
  LayerNorm and position embeddings, use for forecasting,
  classification, anomaly, imputation. First serious evidence that
  LLM priors are usable for TS without full pretraining.
- **Oct 2023 — [TimeGPT-1](../papers/timegpt.md) (Cluster 1).** First
  commercial closed TS-FM API (Nixtla); conformal prediction
  intervals. Architectural and data details remain undisclosed.
- **Oct 2023 — [Lag-Llama](../papers/lag-llama.md) (Cluster 1).**
  First open probabilistic TS-FM with empirical scaling curves.
  Decoder-only with lag-feature tokens and a Student-t output head.
- **Oct 2023 — [TimesFM](../papers/timesfm.md) (Cluster 1).** Google's
  decoder-only patched transformer, ~200M params on ~100B points.
  The prototypical "TS is just next-patch prediction" recipe; the
  asymmetric `P_in=32 / P_out=128` output-patch trick becomes a
  field-wide idiom.
- **Oct 2023 — [LLMTime](../papers/llmtime.md) (Cluster 4).** Feed
  the numbers as plain text to a vanilla GPT-3 or Llama-2 and decode
  the completion. Surprisingly competitive zero-shot; baseline for
  "is any TS-specific machinery needed at all?"
- **Oct 2023 — [Time-LLM](../papers/time-llm.md) (Cluster 4).**
  Reprograms a frozen Llama by cross-attending patch embeddings to
  text-prototype tokens; "Prompt-as-Prefix" conditioning.

## 2024

- **Jan 2024 — [TTM](../papers/ttm.md) (Cluster 5).** IBM Tiny Time
  Mixers: 1–5M-parameter MLP-Mixer on public data, CPU-deployable.
  First paper to argue explicitly that parameter count is not
  destiny for TS foundation models.
- **Feb 2024 — [MOIRAI](../papers/moirai.md) (Cluster 2).**
  Salesforce masked encoder on LOTSA (~27B obs): multi-patch-size
  projections, any-variate attention, mixture-of-Student-t head.
  The first fully open large-corpus masked TS-FM.
- **Feb 2024 — [Timer](../papers/timer.md) (Cluster 1).** Tsinghua's
  decoder-only TS-FM trained on a unified "Single-Series Sequence"
  (S3) format. Emergent few-shot behavior at scale.
- **Feb 2024 — [TOTEM](../papers/totem.md) (Cluster 6).** Cross-domain
  VQ-VAE codebook plus generalist transformer; a discrete universal
  TS representation reusable across tasks.
- **Feb 2024 — [UniTS](../papers/units.md) (Cluster 6).** Harvard/MIT
  unified transformer with task-token interface across 38 datasets
  (forecast / classify / anomaly / impute jointly).
- **Mar 2024 — [Chronos](../papers/chronos.md) (Cluster 2).** AWS T5
  encoder-decoder on quantized TS vocab (4096 bins), 20M–710M family.
  Establishes the "regression via classification on a fixed vocab"
  recipe and TSMixup / KernelSynth augmentation pipeline.
- **Sep 2024 — [Time-MoE](../papers/time-moe.md) (Cluster 3).** First
  billion-parameter TS-FM (2.4B total, 1.1B active). Sparse
  decoder-only MoE on the Time-300B corpus (~309B points). First
  clean TS scaling-law curves on both active and total params.
- **Oct 2024 — [Timer-XL](../papers/timer-xl.md) (Cluster 1 / 6).**
  Timer's long-context multivariate successor. Flattens panels into
  a 1D sequence with 2D position embeddings; "universal
  TimeAttention" treats variates and time uniformly.
- **Oct 2024 — [Moirai-MoE](../papers/moirai-moe.md) (Cluster 3).**
  MOIRAI successor that drops the frequency-specific projections in
  favor of 32-expert top-2 MoE routing. Switches training objective
  from masked encoder to decoder-only causal; cluster-centroid
  gating.
- **Oct 2024 — [MOMENT](../papers/moment.md) (Cluster 2 / 6).** CMU
  Auton Lab open masked-encoder family at 40M / 125M / 385M,
  pretrained on the Time Series Pile. Evaluated jointly on forecast,
  classification, anomaly and imputation.
- **Oct 2024 — [Mamba4Cast](../papers/mamba4cast.md) (Cluster 5).**
  Synthetic-only Mamba-2 SSM trained PFN-style, single-pass horizon
  generation. The cleanest "synthetic-only, non-transformer" data
  point.

## 2025

- **Feb 2025 — [Sundial](../papers/sundial.md) (Cluster 7).** Tsinghua
  continuous flow-matching TS-FM on the ~1T-point TimeBench corpus.
  TimeFlow objective avoids discrete tokenization and parametric
  density heads; modern transformer stack (RoPE, Pre-LN,
  FlashAttention, KV-cache). First flow-matching TS-FM to post
  competitive GIFT-Eval numbers.
- **Oct 2025 — [Chronos-2](../papers/chronos-2.md) (Cluster 2 / 6).**
  AWS 120M encoder-only with *group attention* for in-context
  learning across related series and covariates. Leads fev-bench
  covariate subset (47.0 SQL) where every other pretrained model
  sits at 28–40. Signals that multivariate / covariate support,
  not scale, is the 2025 frontier.

## 2026

- **Mar 2026 — [Timer-S1](../papers/timer-s1.md) (Cluster 1 / 3).**
  Tsinghua / ByteDance 8.3B sparse MoE (top-2 of 32 experts) with
  Serial-Token Prediction: stacked shift-by-one heads replace
  autoregressive rollout. State of the art on GIFT-Eval among
  pre-trained models (MASE 0.693 / CRPS 0.485, a 7.6% / 13.2%
  reduction vs Sundial).

## How to read the trajectory

Four patterns are visible once the papers are lined up by date:

- **2023 was the LLM-adaptation year.** LLMTime, Time-LLM, GPT4TS,
  and the first TimeGPT / Lag-Llama preprints all asked "can we
  reuse NLP machinery?" The answer converged on "yes for the
  architecture, no for the tokenization" — patches beat text, and
  quantized or continuous numeric tokens replaced subword tokens.
- **Early 2024 was the open-recipe explosion.** Chronos, MOIRAI,
  Timer, MOMENT, TTM, UniTS and TOTEM all appeared within three
  months, each staking out a recipe (T5+quant, masked encoder,
  decoder-only, MLP-Mixer, task-token, VQ-VAE). This is the cluster
  of papers the [reading-roadmap](reading-roadmap.md) beginner
  track draws from.
- **Late 2024 was the scale answer.** Time-MoE pushed to 2.4B,
  Moirai-MoE showed sparse routing beats frequency heuristics,
  Timer-XL addressed multivariate, and Mamba4Cast proved synthetic
  data could replace real data at small scale. MoE became the only
  path to dense capacity above ~500M params.
- **2025–2026 is the multivariate and serial-prediction pivot.**
  Sundial attacks the rollout-compounding problem with flow
  matching; Chronos-2 attacks the univariate limitation with group
  attention; Timer-S1 attacks rollout compounding *and* billion-scale
  MoE with Serial-Token Prediction. The 2023 recipe is essentially
  frozen; the open questions are all at the interfaces
  (multivariate, covariates, long horizon, probabilistic
  calibration).

See [open-problems.md](open-problems.md) for what is still open in
each of those axes and [comparison-matrix.md](comparison-matrix.md)
for the dense side-by-side.

## Related wiki pages

- [reading-roadmap.md](reading-roadmap.md)
- [comparison-matrix.md](comparison-matrix.md)
- [open-problems.md](open-problems.md)
- [../foundation-models/taxonomy.md](../foundation-models/taxonomy.md)
- [../papers/papers.md](../papers/papers.md)
