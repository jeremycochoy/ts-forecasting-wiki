# Training Recipes

This page consolidates the disclosed pretraining hyperparameters for
the TS foundation models in this wiki: optimizer, learning rate,
schedule, batch size, step count, precision, hardware. The goal is to
let a reader who wants to train their own TS-FM see the recipe space
at a glance, rather than digging through each paper's appendix in
turn. Entries are sourced from the PDFs in `papers/`; cells marked
"—" mean the paper does not disclose that value in the extracted
main text.

For motivation, context on *why* these choices matter, and the
open questions around them, see
[../concepts/scaling-laws.md](../concepts/scaling-laws.md) and
[../benchmarks/efficiency-and-cost.md](../benchmarks/efficiency-and-cost.md).

## Pretraining hyperparameters, as reported

| Model | Optimizer | Peak LR | Schedule | Batch size | Steps | Precision | Hardware | Source |
|---|---|---|---|---|---|---|---|---|
| [Chronos](../papers/chronos.md) (T5 family) | AdamW, wd 0.01 | 1e-3 | Linear decay to 0 | 256 sequences (effective) | 200K | — | 8x A100 40GB | Chronos, Sec 5 (arXiv:2403.07815) |
| [TimesFM](../papers/timesfm.md) (200M) | — | 5e-4 (peak) | Cosine decay | 4096 | 1.5M iters | — | 16-core TPUv5e, ~2 days | TimesFM, Sec 3 and App (arXiv:2310.10688) |
| [MOIRAI](../papers/moirai.md) small | AdamW, wd 0.1, b1 0.9, b2 0.98 | 1e-3 | Linear warmup 10K + cosine | — | 100K | — | A100 40GB | MOIRAI, App (arXiv:2402.02592) |
| [MOIRAI](../papers/moirai.md) base / large | AdamW, wd 0.1, b1 0.9, b2 0.98 | 1e-3 | Linear warmup 10K + cosine | — | 1M | — | A100 40GB | MOIRAI, App |
| [Moirai-MoE](../papers/moirai-moe.md) small | AdamW, wd 0.1, b1 0.9, b2 0.98 | 1e-3 | Linear warmup 10K + cosine | 1024 | 50K | bfloat16 | 16x A100 40GB | Moirai-MoE, Sec B (arXiv:2410.10469) |
| [Moirai-MoE](../papers/moirai-moe.md) base | same | 1e-3 | same | 1024 | 250K | bfloat16 | 16x A100 40GB | Moirai-MoE, Sec B |
| [Time-MoE](../papers/time-moe.md) family | AdamW, wd 0.1, b1 0.9, b2 0.95, aux-loss 0.02 | 1e-3 | Linear warmup 10K + cosine | 1024 | 100K | BF16 | 128x A100 80GB | Time-MoE, App B (arXiv:2409.16040) |
| [Timer](../papers/timer.md) | AdamW | 5e-5 (base) to 2e-6 (final) | Cosine annealing | 8192 patches | 10 epochs (proportional) | — | A100 | Timer, App B (arXiv:2402.02368) |
| [Timer-S1](../papers/timer-s1.md) (8.3B / 0.75B active) | — (load-balancing aux loss) | — | Two-stage: pretrain + GIFT-Eval continued + RoPE context extension | — | — | BF16 | ByteDance VeOmni cluster | Timer-S1, method section (arXiv:2603.04791) |
| [Sundial](../papers/sundial.md) | AdamW | — | — | — | — | — | 32x A100 | Sundial, App B (arXiv:2502.00816) |
| [MOMENT](../papers/moment.md) family | Adam + weight decay | 1e-4 peak / 1e-7 final | Cosine | 2048 patches | up to 1e7 iters cap | — | single A6000 49GB per variant | MOMENT, App (arXiv:2402.03885) |
| [TTM](../papers/ttm.md) | — (TSMixer training) | — | — | 4500 | 20 epochs | — | 6x A100, 24–30 hours total | TTM, Sec 4.3 and App (arXiv:2401.03955) |
| [Lag-Llama](../papers/lag-llama.md) | — (see Appendix D search) | 1e-4 | — | 256 | 100 windows/epoch | — | 1x Tesla P100 12GB | Lag-Llama, App D (arXiv:2310.08278) |
| [Mamba4Cast](../papers/mamba4cast.md) | AdamW | 1e-5 peak / 1e-7 final | Cosine | 64 (synthetic samples) | 420K batches (3 days) | — | 1x RTX 2080 Ti | Mamba4Cast, App (arXiv:2410.09385) |
| [LaT-PFN](../papers/lat-pfn.md) | — (linear warmup on weight decay and target-encoder EMA decay; µ-parametrization for stability) | — | Linear warmup on schedule parameters | — | — (24h wall-clock per seed) | — | 1x NVIDIA A10G per seed | LaT-PFN, Sec 4 (arXiv:2405.10093) |
| [TS-JEPA](../papers/ts-jepa.md) | AdamW | Tuned ∈ {1e-3, 1e-4, 1e-5, 1e-6} | — | 32 | — | — | 1x NVIDIA V100 | TS-JEPA, App (arXiv:2509.25449) |
| [MTS-JEPA](../papers/mts-jepa.md) | — (5-seed averaging for main results) | — | — | — | — | — | — | MTS-JEPA, Sec 4 (arXiv:2602.04643) |

A few rows in this table are striking on their own:

- **Only two papers disclose their *cluster* size.** Time-MoE (128x
  A100 80GB) and Moirai-MoE (16x A100 40GB) are the only entries with
  a clear "how many GPUs" number. Most others report a GPU *type*
  without a count, which is exactly the gap that makes reproducibility
  hard to estimate.
- **AdamW lr=1e-3 with cosine-after-10K-warmup is the majority
  recipe.** Chronos uses a simpler linear-decay schedule, but TimesFM,
  MOIRAI, Moirai-MoE and Time-MoE all converge on the same Adam family
  with 10K warmup steps and cosine annealing. Peak 1e-3 works for all
  dense and MoE sizes in that cluster. TimesFM's 5e-4 peak is the
  outlier on the low side; MOMENT's 1e-4 peak on BERT-style masked
  reconstruction is a different regime.
- **Batch size and steps are coupled, not independent.** Chronos at
  200K steps with batch 256 processes roughly the same token volume
  as MOIRAI-base at 1M steps with an effective batch from packing.
  Chronos Appendix Section 5 and MOIRAI Section 4 both explicitly
  note that packed sequences inflate the effective batch size well
  above the nominal number, which is one of the hardest things to
  compare across papers.
- **Hardware footprints span four orders of magnitude.**
  Mamba4Cast's single 2080 Ti (synthetic data only) and Lag-Llama's
  single P100 sit at one extreme; Time-MoE's 128-GPU A100 80GB cluster
  at the other. TTM (6x A100, 24–30 hours total) is the cheapest *real-data*
  pretraining recipe in the set.

## Training-objective and precision summary

- **Next-patch point loss.** TimesFM, Timer, Timer-XL, TTM,
  Mamba4Cast. Usually MSE or Huber (Time-MoE uses Huber with an
  auxiliary load-balancing term for MoE routing).
- **Categorical cross-entropy on a quantized vocabulary.** Chronos
  (T5 family), over a 4096-bin vocab.
- **Masked reconstruction MSE.** MOMENT, MOIRAI (with an NLL twist
  from the mixture-of-Student-t head); Moirai-MoE drops the masked
  objective in favor of causal decoder-only training with a packing
  masking ratio r=0.3.
- **Flow matching on continuous patches.** Sundial's TimeFlow Loss
  regresses a conditional-optimal-transport velocity field between
  Gaussian noise and the target patch.
- **Serial-Token Prediction (STP).** Timer-S1 replaces next-token
  prediction with a stack of shift-by-one heads, each trained with a
  weighted loss `1/sqrt(j)` on the j-th block to focus on near-term
  accuracy.
- **Precision: BF16 for MoE, FP32 otherwise.** Time-MoE, Moirai-MoE
  (bfloat16) and Timer-S1 (BF16) all explicitly use reduced precision;
  every dense TS-FM in the table that discloses precision uses FP32 by
  default. This is the one place where the TS-FM literature lags
  modern LLM practice.

## What these recipes do *not* tell you

Three things are missing from every table cell, and the recipes are
not comparable until they appear:

- **Effective token count.** "100K steps at batch 1024" is not a
  direct token count without the packing ratio and the patch length.
  Time-MoE reports it explicitly (4M time points per iteration);
  MOIRAI and MOMENT require derivation. See
  [../research/reproducibility.md](reproducibility.md) for the
  closest approximation we have.
- **Total FLOPs.** Only TimesFM reports wall-clock time on a specific
  hardware footprint (`~2 days on 16-core TPUv5e`); no paper in the
  list reports actual pretraining FLOPs or energy. MOMENT reports
  GPU-hours and tCO2-equivalent (404 hours / 40.8 tCO2 for Large),
  which is the sole energy disclosure.
- **Stability tricks.** Gradient clipping, loss spike recovery,
  warmup length sensitivity, and learning-rate decay tails are mostly
  absent. MOMENT's gradient clip at 5.0 and Moirai-MoE's warmup=10K
  specification are the closest the field gets to a disclosed
  stability recipe.

## Related wiki pages

- [comparison-matrix.md](comparison-matrix.md) — the architectural
  side-by-side that complements this recipe table.
- [reproducibility.md](reproducibility.md) — open weights, code,
  data, compute, and deployment footprint per paper.
- [../benchmarks/efficiency-and-cost.md](../benchmarks/efficiency-and-cost.md)
  — inference-side efficiency and cost.
- [../concepts/scaling-laws.md](../concepts/scaling-laws.md) —
  parameter / data / compute scaling.
- [../benchmarks/training-a-small-model.md](../benchmarks/training-a-small-model.md)
  — practical guide for pretraining a small TS-FM.
