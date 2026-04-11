# Decision Guide — Which Model Should I Pick?

You landed on this wiki because you have a task and a budget, and
you want to know which TS foundation model to try first. This page
partitions the field along the axes that actually matter in
deployment — probabilistic vs point output, univariate vs multivariate,
covariate support, horizon length, compute budget — and names the
1–3 models that lead each regime as of early 2026.

It reads
[state-of-the-art.md](state-of-the-art.md) for the "who wins where"
and [efficiency-and-cost.md](efficiency-and-cost.md) for the "what
does it cost" and compresses them into actionable picks. Every pick
points back to the leaderboard cell that justifies it. If your
situation does not match any regime below, fall back to
[leaderboard.md](leaderboard.md) and browse.

## Step 1 — What are you predicting?

### Univariate point forecasts, zero-shot

- **Top pick:** [TimesFM](../papers/timesfm.md) (200M) or the
  successor TimesFM-2.5 lineage. Closest to an apples-to-apples
  decoder-only baseline with the cleanest Monash and LTSF numbers;
  TimesFM-2.5 posts 51.0 / 29.5 WQL / MASE skill on GIFT-Eval
  (Chronos-2, Table 4).
- **Budget pick:** [TTM](../papers/ttm.md) (1–5M). The only TS-FM
  that reports CPU inference time in milliseconds (TTM, Table 3)
  and still sits near the LTSF Pareto frontier (TTM-A ETTh1 MSE
  0.400, TTM Table 1 — essentially matching Sundial-Large at
  25–100x fewer parameters per
  [efficiency-and-cost.md](efficiency-and-cost.md)).
- **Scale pick:** [Timer-S1](../papers/timer-s1.md) if you need the
  current GIFT-Eval ceiling and can afford the 0.75B-activated /
  8.3B-total MoE inference cost (Timer-S1, results section).

### Univariate probabilistic forecasts (you care about CRPS / WQL)

- **Top pick:** [Chronos-2](../papers/chronos-2.md) (120M). 51.4 /
  30.2 GIFT-Eval WQL / MASE skill, 46.6 on Chronos Benchmark II
  (Chronos-2, Tables 4 and 5); quantile decoder, no sampling.
- **Direct alternative:** [Sundial](../papers/sundial.md) if you
  want generative flow-matching samples and test-time calibration
  knobs (sample count, flow steps). Sundial, Table 1 LTSF numbers
  ETTh1 0.395, ETTm2 0.254 — the cleanest LTSF row in the wiki.
- **Cheap parametric alternative:** [MOIRAI](../papers/moirai.md)
  Small (14M) if you want a mixture-of-Student-t head on CPU-
  feasible memory (MOIRAI, Table 5 ties or beats full-shot PatchTST
  on 5 of 6 probabilistic datasets).

### Multivariate / covariate-informed forecasting

- **Only real pick:** [Chronos-2](../papers/chronos-2.md). On the
  fev-bench covariates subset (42 tasks), Chronos-2 reaches 47.0 SQL
  skill score; every other pretrained model that is not
  TabPFN-TS degrades to 28.0–39.9 because it silently ignores the
  covariates (Chronos-2, Figure 4b). On real covariate case studies
  (European energy prices, Rossmann retail) the gap widens to 51.3
  vs 46.5 SQL and 48.6 vs 44.3 WQL (Chronos-2, Figure 5).
- **For cross-variate dependence without covariates:**
  [Timer-XL](../papers/timer-xl.md) (flattened multivariate) or
  [MOIRAI](../papers/moirai.md) (any-variate attention). Both handle
  variable variate counts; neither wins decisively on fev-bench
  multivariate, but they beat univariate baselines on datasets with
  genuine cross-series coupling.

### Anomaly detection, imputation, classification (not just forecast)

- **Top pick:** [MOMENT](../papers/moment.md) family (40M / 125M /
  385M). It is the only TS-FM that reports all four tasks — forecast,
  classify, impute, anomaly — from a single encoder. See
  [../concepts/multi-task-universal.md](../concepts/multi-task-universal.md).
- **Task-token alternative:** [UniTS](../papers/units.md) for
  generative-plus-predictive joint training across 38 datasets.

## Step 2 — How tight is your compute budget?

### CPU only, <10M parameters

- [TTM](../papers/ttm.md) is the only TS-FM that is designed for
  this regime; TTM-Base is ~240,000x faster on CPU than Chronos-Base
  per TTM Table 3 (4.7 ms/batch vs 2340 s/batch, reproduced in
  [efficiency-and-cost.md](efficiency-and-cost.md)).

### Small GPU, 50M parameters or less

- [Chronos-2](../papers/chronos-2.md) "small" variant (28M, Chronos-2
  Sec 5.4) — gives up ~1 skill-score point vs the 120M flagship at
  nearly 2x faster inference.
- [MOIRAI](../papers/moirai.md)-Small (14M) — 205 ms/batch on GPU
  per TTM Table 3.
- [Moirai-MoE](../papers/moirai-moe.md)-Small (11M activated / 117M
  total) — matches dense-Moirai inference time with MoE capacity
  (Moirai-MoE, Table 4).

### Mid-size GPU, 100–500M parameters

- [Chronos-2](../papers/chronos-2.md) (120M) — 300 series/s on a
  single A10G (Chronos-2, Sec 5.4).
- [TimesFM](../papers/timesfm.md) (200M) or TimesFM-2.5.
- [MOMENT](../papers/moment.md)-Base (125M) or Large (385M) for
  multi-task coverage.

### Large GPU or multi-GPU, billion-scale

- [Time-MoE](../papers/time-moe.md)-Ultra (2.4B total / 1.1B
  active, runs within 8GB VRAM per Time-MoE Sec 3.2.3).
- [Timer-S1](../papers/timer-s1.md) (8.3B / 0.75B active) for
  GIFT-Eval ceiling performance.

## Step 3 — How long is your horizon?

- **Short (≤96 steps).** Any of the top picks above; Chronos
  Benchmark II numbers (most tasks have <300 observations) are the
  cleanest guide, and Chronos-2 leads at 46.6 (Chronos-2 Table 5).
- **Medium (96–720 steps).** LTSF-suite numbers are your guide.
  Sundial-Large and Time-MoE-Large are near parity at ETTh1 MSE
  0.395 / 0.394; TTM-A matches them at 5M parameters
  (leaderboard.md Section 5).
- **Long (>720 steps).** Single-pass decoders win over
  autoregressive rollout: Sundial (flow-matching head),
  Chronos-2 (quantile decoder), Timer-S1 (Serial-Token Prediction).
  All three bypass the compounding error that hits TimesFM, Timer
  and Time-MoE at long horizons — see
  [../research/failure-modes.md](../research/failure-modes.md) and
  [../research/open-problems.md](../research/open-problems.md#6-long-horizon-forecasting-and-autoregressive-rollout-error).

## Step 4 — What is your deployment constraint?

- **Open weights, no API dependency.** Chronos, Chronos-2,
  TimesFM, MOIRAI, Moirai-MoE, MOMENT, Timer, Timer-XL, TTM,
  Lag-Llama, Time-MoE, Sundial and Mamba4Cast all ship HuggingFace
  or GitHub checkpoints. See
  [../research/reproducibility.md](../research/reproducibility.md).
- **Closed API only.** [TimeGPT](../papers/timegpt.md) (Nixtla)
  remains the only commercial-API TS-FM in the wiki; no self-hosted
  option.
- **Fine-tuning on your own corpus.** Every open-weights family
  above supports it; TTM and MOIRAI have the cleanest fine-tuning
  recipes in their repos. See
  [training-a-small-model.md](training-a-small-model.md).

## If you are training your own model

See [training-a-small-model.md](training-a-small-model.md) and
[../research/training-recipes.md](../research/training-recipes.md)
for the disclosed pretraining hyperparameters of the 20 papers.
Short answer: LOTSA is the default open corpus, AdamW lr=1e-3 with
10K-step linear warmup plus cosine annealing is the majority recipe,
and Time-MoE's 128x A100 80GB cluster is the upper reference point
for what "full scale" currently costs.

## Related wiki pages

- [state-of-the-art.md](state-of-the-art.md) — narrative "who wins
  where" behind the picks on this page.
- [leaderboard.md](leaderboard.md) — the raw per-benchmark tables.
- [efficiency-and-cost.md](efficiency-and-cost.md) — params,
  latency, memory and CPU deployability.
- [methodology-caveats.md](methodology-caveats.md) — the
  asterisks and footnotes these picks hide.
- [../research/failure-modes.md](../research/failure-modes.md) —
  the documented weakness of each model in one table.
- [../foundation-models/taxonomy.md](../foundation-models/taxonomy.md)
  — the seven-cluster map this guide draws from.
