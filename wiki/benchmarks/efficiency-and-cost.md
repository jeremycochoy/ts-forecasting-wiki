# Efficiency and Cost

Accuracy is only half the story. Most TS-FM papers quote at least
one efficiency number — parameters, pretraining compute, inference
latency, peak memory, CPU runtime — and these numbers span several
orders of magnitude across the current generation. This page
aggregates the efficiency claims as reported, so a practitioner can
tell a 1M-parameter model from a 2.4B-parameter one at a glance.

## 1. Parameter counts (as reported)

| Model | Parameters | Activated | Source |
|---|---|---|---|
| TTM-Base | 1M | 1M | [TTM](../papers/ttm.md), Section 4.3 (arXiv:2401.03955) |
| TTM-Enhanced | 4M | 4M | TTM, Section 4.3 |
| TTM-Advanced | 5M | 5M | TTM, Section 4.3 |
| Moirai-Small | 14M | 14M | [Moirai-MoE](../papers/moirai-moe.md), Table 4 (arXiv:2410.10469) |
| Chronos-Tiny (TTM label) [^tt] | 8M | 8M | TTM, Table 3 |
| Moirai-MoE-Small | 117M total / 11M act. | 11M | Moirai-MoE, Table 4 |
| [Chronos-2](../papers/chronos-2.md) Small | 28M | 28M | Chronos-2, Section 5.4 (arXiv:2510.15821) |
| MOMENT-Small | 40M | 40M | [MOMENT](../papers/moment.md), Table 8 (arXiv:2402.03885) |
| Chronos-Small | 46M | 46M | Moirai-MoE, Table 4 |
| Time-MoE-Base | 113M total / 50M act. | 50M | [Time-MoE](../papers/time-moe.md), Table 2 (arXiv:2409.16040) |
| Moirai-Base | 91M | 91M | Moirai-MoE, Table 4 |
| Moirai-MoE-Base | 935M total / 86M act. | 86M | Moirai-MoE, Table 4 |
| Chronos-2 | 120M | 120M | Chronos-2, Section 5 |
| MOMENT-Base | 125M | 125M | MOMENT, Table 8 |
| Chronos-Base | 200M | 200M | TTM, Table 3 |
| [TimesFM](../papers/timesfm.md) (v1) | 200M | 200M | TimesFM, Section A.1 (arXiv:2310.10688) |
| Time-MoE-Large | 453M total / 200M act. | 200M | Time-MoE, Table 2 |
| Moirai-Large | 311M | 311M | Moirai-MoE, Table 4 |
| MOMENT-Large | 385M | 385M | MOMENT, Table 8 |
| Chronos-Large | 709M | 709M | TTM, Table 3 |
| Time-MoE-Ultra | 2.4B total / 1.1B act. | 1.1B | Time-MoE, Section 3.2.3 |
| Sundial-Large | ~130M class | — | [Sundial](../papers/sundial.md), Table 5 (inferred) |

[^tt]: The Chronos paper releases only Mini (20M), Small (46M), Base (200M), and Large (710M) variants — there is no "Chronos-Tiny" model in the paper itself. TTM Table 3 uses "Chronos-Tiny" as its label for the smallest Chronos config they benchmarked (~8M); we preserve TTM's label here but the reader should not expect a matching checkpoint in the Chronos release. See [model-sizing-cheatsheet.md](model-sizing-cheatsheet.md) for the canonical Chronos size table.

## 2. Inference latency and memory (on the same task)

TTM Table 3 gives one of the only apples-to-apples efficiency
comparisons in the literature: all models running on the same
benchmark batch, same hardware setup.

| Model | GPU time (ms/batch) | CPU time (s/batch) | Peak GPU mem (GB) | Source |
|---|---|---|---|---|
| TTM-Base | 4.7 | 0.01 | 0.06 | TTM, Table 3 |
| [GPT4TS](../papers/gpt4ts.md) | 13.9 | 0.3 | 1.34 | TTM, Table 3 |
| TimesFM (200M) | 24 | 0.4 | 2 | TTM, Table 3 |
| MOMENT-Large | 88 | 1.4 | 8 | TTM, Table 3 |
| Moirai-Small | 205 | 1.4 | 0.1 | TTM, Table 3 |
| Moirai-Base | 335 | 4.1 | 1 | TTM, Table 3 |
| Moirai-Large | 693 | 10.5 | 2 | TTM, Table 3 |
| Chronos-Tiny (TTM label) [^tt] | 1,389 | 2,504 | 2 | TTM, Table 3 |
| Chronos-Small | 1,386 | 2,349 | 6 | TTM, Table 3 |
| Chronos-Base | 1,395 | 2,340 | 16 | TTM, Table 3 |
| Chronos-Large | 1,393 | 2,352 | 41 | TTM, Table 3 |
| [Lag-Llama](../papers/lag-llama.md) | 1,619 | 37.5 | 0.2 | TTM, Table 3 |

Two things stand out. First, TTM-Base is **~240,000x faster on CPU**
than Chronos-Base and **~267x lighter** on peak GPU memory (TTM,
Table 3). The gap is dominated by [Chronos](../papers/chronos.md)'s patch-size-1 binned
tokenization, which forces one autoregressive step per time point.
Second, [Moirai](../papers/moirai.md) (masked-encoder, non-autoregressive) is already ~7x
faster than Chronos at the *same* parameter count because it
decodes the entire forecast in one pass.

Moirai-MoE Table 4 gives a second independent comparison focused
on Chronos vs Moirai vs Moirai-MoE (context length 512, 20 samples,
[Monash](../datasets-benchmarks/monash-archive.md) subset):

| Model | Spent time (s) | Source |
|---|---|---|
| Chronos-Small (46M) | 551 | Moirai-MoE, Table 4 |
| Chronos-Base (200M) | 1,177 | Moirai-MoE, Table 4 |
| Chronos-Large (710M) | 2,780 | Moirai-MoE, Table 4 |
| Moirai-Small (14M) | 264 | Moirai-MoE, Table 4 |
| Moirai-Base (91M) | 358 | Moirai-MoE, Table 4 |
| Moirai-Large (311M) | 537 | Moirai-MoE, Table 4 |
| Moirai-MoE-Small (11M act./117M) | 273 | Moirai-MoE, Table 4 |
| Moirai-MoE-Base (86M act./935M) | 370 | Moirai-MoE, Table 4 |

Moirai-MoE adds ~10x total parameters over dense Moirai but stays
at *identical* inference time because only a fraction of the
experts fires per token.

## 3. Training compute and pretraining scale

| Model | Pretraining data | Training compute | Source |
|---|---|---|---|
| TTM-Advanced (5M) | 1B samples | 24–30 GPU-hours on 6x A100 | TTM, Section 4.3 |
| TimesFM (200M) | ~100B timepoints | ~2 TPUv5e-days per 1.5M iters | TimesFM, Section 6.1 |
| MOIRAI | [LOTSA](../datasets-benchmarks/lotsa.md): 27B observations / 231B flat-observations | not individually reported | MOIRAI, Appendix A (arXiv:2402.02592) |
| Chronos-2 (120M) | Undisclosed real+synthetic | throughput 300 series/s on one A10G at inference | Chronos-2, Section 1 |
| MOMENT-Large (385M) | [Time Series Pile](../datasets-benchmarks/time-series-pile.md) | 404 GPU-hours / 40.8 tCO2eq | MOMENT, Table 8 |
| MOMENT-Base (125M) | Time Series Pile | 308 GPU-hours / 31.1 tCO2eq | MOMENT, Table 8 |
| MOMENT-Small (40M) | Time Series Pile | 308 GPU-hours / 31.1 tCO2eq | MOMENT, Table 8 |
| Time-MoE-Ultra | [Time-300B](../datasets-benchmarks/time-300b.md): 309B timepoints, 9 domains | 100K steps × 1024 batch × 4M tokens/iter | Time-MoE, Table 1 & Section 3.2.3 |
| Sundial | [TimeBench](../datasets-benchmarks/timebench.md): ~1T pretraining points | not individually reported | Sundial, Table 5 |

Time-300B and TimeBench are the two largest TS pretraining corpora
currently documented, at roughly 3 and 10x the LOTSA scale.
Whether that translates linearly to downstream accuracy remains
contested — Moirai-MoE-Base outperforms Chronos-Large on zero-shot
[CRPS](../evaluation/metrics.md#21-crps--continuous-ranked-probability-score) with ~8x fewer active parameters (Moirai-MoE, Table 2), which
is evidence that pure data scale is not the only dimension that
matters.

## 4. Efficiency wins via sparsity

- **Time-MoE** reports training costs reduced by 78% and inference
  costs by 39% versus dense models with the same activated-
  parameter count (Time-MoE, Section 4.4 Scalability Analysis).
- **Moirai-MoE** matches dense-Moirai inference time at roughly 8x
  total parameters by activating only 11M/86M of its 117M/935M
  experts per forward pass (Moirai-MoE, Table 4).
- **Chronos-2** reports 300 time-series/s throughput on a single
  NVIDIA A10G GPU at 120M parameters, with a 28M "small" variant
  at nearly 2x faster inference and only ~1 skill-score point
  lower on [GIFT-Eval](../datasets-benchmarks/gift-eval.md) (Chronos-2, Section 5.4).

## 5. CPU deployability

TTM is the only TS-FM designed to run *well* on CPU: 0.01s/batch
CPU time (TTM, Table 3) with total model size <5M parameters. The
Chronos-2 small variant, Time-MoE-Base, and Moirai-Small are the
next cheapest options at 28M–50M activated parameters. Everything
above ~200M parameters is effectively GPU-only for real-time
forecasting — Chronos-Base's 2,340 s/batch CPU time (TTM, Table 3)
is an emphatic demonstration of why.

## 6. Reading efficiency alongside accuracy

The efficiency numbers above become interesting when cross-
referenced with [leaderboard.md](leaderboard.md):

- TTM-Advanced at 5M parameters posts ETTh1 MSE 0.400 (TTM, Table
  1), matching Sundial-Large (0.395, Sundial Table 1) at 25-100x
  fewer parameters.
- Moirai-MoE-Base at 86M activated parameters beats Chronos-Large
  (710M) on aggregated zero-shot CRPS (0.478 vs 0.500; Moirai-MoE,
  Table 2) — ~8x more efficient per query, winning anyway.
- Chronos-2 at 120M beats TimesFM-2.5 on fev-bench skill score (47.3
  vs 42.3; Chronos-2, Table 3) at roughly 1.7x fewer parameters.

The efficient frontier of TS-FMs in late 2025 runs roughly
TTM (5M) → Moirai-MoE-Small (11M) → Chronos-2-Small (28M) → Time-MoE-Base
(50M) → Chronos-2 (120M). Everything else either sits strictly below
that frontier or is trading millions of parameters for marginal
accuracy gains.

## Related wiki pages

- [leaderboard.md](leaderboard.md)
- [state-of-the-art.md](state-of-the-art.md)
- [methodology-caveats.md](methodology-caveats.md)
- [../foundation-models/taxonomy.md](../foundation-models/taxonomy.md)
