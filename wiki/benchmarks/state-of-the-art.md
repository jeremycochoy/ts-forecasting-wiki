# State of the Art — Which Model Wins Where

This page interprets the numbers in
[leaderboard.md](leaderboard.md). It partitions the TS-FM space by
forecasting *regime* — the shape of the task a user actually faces —
because no single model is SOTA everywhere. Every claim here points
back to a specific cell on the leaderboard page.

## Zero-shot probabilistic forecasting (univariate, no covariates)

As of late 2025, the narrow "zero-shot univariate probabilistic"
leaderboard is tightly packed at the top. On [GIFT-Eval](../datasets-benchmarks/gift-eval.md), four models —
[Chronos-2](../papers/chronos-2.md) (51.4), TimesFM-2.5 (51.0), TiRex (50.2) and Toto-1.0
(48.6) — are within 3 skill-score points of each other under [WQL](../evaluation/metrics.md#23-wql--weighted-quantile-loss)
(Chronos-2, Table 4). On the same suite, Chronos-2 claims the top
spot but the pairwise confidence intervals it publishes for fev-bench
(Chronos-2, Figure 2b) show its win over TiRex and TimesFM-2.5 is
statistically significant (2–3 skill-score points, non-overlapping
95% CIs), whereas the gap between TiRex and TimesFM-2.5 is not.

The practical takeaway for *just* zero-shot univariate [CRPS](../evaluation/metrics.md#21-crps--continuous-ranked-probability-score)/WQL is
that Chronos-2, TimesFM-2.5 and TiRex are effectively a three-way
tie, with Chronos-2 slightly ahead on aggregate skill score and
clearly ahead on short-context tasks ([Chronos](../papers/chronos.md) Benchmark II WQL
skill score 46.6 vs 42.4/41.7, Chronos-2 Table 5).

## Zero-shot forecasting with covariates

This is the single regime with a clear separation. On the
fev-bench covariates subset (42 tasks), Chronos-2 reaches 47.0 SQL
skill score; TabPFN-TS, the only other model with native
known-covariate support, reaches 40.0; every other pretrained
model — including TiRex, TimesFM-2.5, Moirai-2.0, Toto-1.0, [Sundial](../papers/sundial.md) —
ignores the covariates and sits in a tight band between 28.0 and
39.9 (Chronos-2, Figure 4b). On domain case studies with real
dynamic covariates (European energy prices, Rossmann retail sales),
the gap widens further: 51.3 vs 46.5 SQL on the energy covariates
subset and 48.6 vs 44.3 WQL on retail (Chronos-2, Figure 5). If the
production task has covariates, Chronos-2 is the only current public
TS-FM where that information is actually used at inference time.

## Fine-tuned / long-horizon point forecasting (LTSF)

The classic "long-sequence forecasting" benchmarks (ETT, Weather,
ECL, Traffic, averaged over horizons {96, 192, 336, 720}) reward
different models than the zero-shot probabilistic suites. On ETTh1
and ETTh2 zero-shot MSE, Sundial-Large posts 0.395 / 0.334 and
Time-MoE-Large 0.394 / 0.405 (Sundial Table 1; [Time-MoE](../papers/time-moe.md) Table 3),
which are essentially at parity with the best full-shot baselines
of the 2023 era (PatchTST ~0.41, iTransformer ~0.45 per Time-MoE
Table 4). TTM-A (5M parameters) matches them at 0.400 / 0.333 ([TTM](../papers/ttm.md)
Table 1) despite being 80-500x smaller. MOIRAI-Large, by contrast,
is noticeably worse on ETTh1 (0.510 MSE, [MOIRAI](../papers/moirai.md) Table 6) — a
consistent weak point for that model family.

On this regime, the differentiator is not zero-shot prowess but
whether the model's training mix included ETT-like data. See
[methodology-caveats.md](methodology-caveats.md) — LTSF results for
[TimesFM](../papers/timesfm.md) and Chronos need an asterisk because their pretraining
corpora contain datasets overlapping with the ETT / Weather / ECL
families.

## Multivariate forecasting

On the fev-bench multivariate subset (26 tasks), Chronos-2 again
leads (57.9 SQL skill score) but the margin is small: Toto-1.0
57.6, TimesFM-2.5 57.1, TiRex 56.1 (Chronos-2 Figure 4a). The
Chronos-2 paper explicitly notes that "univariate channel-independent
models often perform on par with multivariate ones" on this subset
(Chronos-2, Section 5.2), echoing Nie et al.'s channel-independence
finding for PatchTST. Unless a dataset has short history and strong
instantaneous cross-series dependence, native multivariate support
(Chronos-2, Toto-1.0) is a small effect — much smaller than native
covariate support.

## Short-context / short-history tasks

Chronos Benchmark II is the best proxy because most of its 27 tasks
have fewer than 300 observations. Here the top-4 ordering is
Chronos-2 (46.6), TimesFM-2.5 (42.4), Toto-1.0 (41.9), TiRex (41.7)
under WQL skill score (Chronos-2, Table 5). Chronos-2 opens a ~4
skill-score-point lead here — considerably larger than on GIFT-Eval —
which the authors attribute to its [in-context learning](../concepts/in-context-learning.md) (ICL)
mechanism: with little history per series, the model leans on cross-
series information from the batch (Chronos-2, Section 5.2).

## Sub-10M lightweight tier

A handful of 2025-2026 models deliberately stay below 10M parameters
and argue they close most of the accuracy gap to billion-parameter
transformers. [TTM](../papers/ttm.md) (1-5M, TSMixer) is the original
case; [SEMPO](../papers/sempo.md) (NeurIPS 2025, 6.5M encoder-decoder
transformer) reports a 12% / 22% average MSE reduction over SOTA
TS-FMs on 16-dataset TSLib/GIFT-Eval averages (SEMPO, Table 1 + §5),
with ETTh1 inference at 22 s vs ~205 s Moirai-S and ~14,185 s
Chronos-L (SEMPO, Figure 6). The SEMPO numbers are on TSLib's MSE/MAE
convention and *are not directly comparable* to the GIFT-Eval WQL/MASE
skill-score tables on [leaderboard.md](leaderboard.md); no public head-
to-head of SEMPO against Chronos-2 / TimesFM-2.5 / TiRex / Moirai-2 on
either GIFT-Eval full or fev-bench exists yet.
[Mamba4Cast](../papers/mamba4cast.md) rounds out the cluster on the
synthetic-only SSM side. If the use case is a point-forecast univariate
problem under a strict CPU / 10M-param budget, any of these three are
reasonable first picks; the choice depends more on the backbone you
want (TSMixer, transformer, SSM) than on accuracy ordering.

## JEPA / latent-space prediction (Cluster 8)

[LaT-PFN](../papers/lat-pfn.md), [TS-JEPA](../papers/ts-jepa.md) and
[MTS-JEPA](../papers/mts-jepa.md) are the three current members of
[Cluster 8](../foundation-models/taxonomy.md#cluster-8--jepa--latent-space-prediction).
None reports on Monash, GIFT-Eval, Chronos Benchmark II or fev-bench,
so they cannot be ranked alongside the forecasting-first cohort
above. Their evaluation lives on different axes:

- **Classification (UCR / UEA).** [LaT-PFN](../papers/lat-pfn.md)
  posts 47.9% mean accuracy on UCR-128 with a frozen-encoder + SVM
  probe, beating TS2Vec (44.8%) zero-shot. [TS-JEPA](../papers/ts-jepa.md)
  reaches 91.5% on FordA, 73.8% on FordB, 89.5% on ECG5000 — within
  2 points of fully supervised transformers (TS-JEPA, Table 1).
- **Forecasting.** [LaT-PFN](../papers/lat-pfn.md) beats
  ARIMA / FBProphet / ForecastPFN on 5 LTSF datasets zero-shot, but
  is not benchmarked against modern TS-FMs. [TS-JEPA](../papers/ts-jepa.md)
  loses short-term to autoregressive on all 3 forecasting datasets
  (next-patch MSE) but wins long-term on 2 of 3 (cumulative MSE).
- **Anomaly *prediction*.** [MTS-JEPA](../papers/mts-jepa.md)
  posts top F1 + AUC across MSL / SMAP / SWaT / PSM, beating nine
  baselines including PatchTST, TS2Vec, iTransformer, TimesNet and
  TS-JEPA itself.

If your use case is *not* zero-shot point or quantile forecasting on
Monash / GIFT-Eval — particularly if you want a representation
encoder for downstream classification or anomaly prediction — the
JEPA cluster is competitive. For headline forecasting accuracy on
the dominant benchmarks, the forecasting-first cohort still wins
because Cluster 8 has not been benchmarked there.

## Non-forecasters in the multi-task cluster

[TSPulse](../papers/tspulse.md) (ICLR 2026, 1.06M-parameter TSMixer)
is the newest member of [Cluster 6](../foundation-models/taxonomy.md#cluster-6--multi-task--universal-unified-ts-models)
and explicitly does *not* report forecasting benchmarks — the paper
flags forecasting as future work and evaluates only classification
(UEA-29), imputation (LTSF under hybrid masking), anomaly detection
(TSB-AD VUS-PR) and similarity search (TSPulse, §6 and §7). It does
not appear on any forecasting leaderboard table above by design, so
it cannot be ranked against Chronos-2, TimesFM-2.5, or Moirai-2 on
the axes this page partitions. Treat TSPulse as the analysis-side
companion to [TTM](../papers/ttm.md), not as a forecasting baseline.

## Negative scaling on GIFT-Eval

[Moirai 2.0](../papers/moirai-2.md) is the newest cautionary data point.
On its own GIFT-Eval table (Table 1, arXiv:2511.11698), small (11.4M)
posts MASE/CRPS 0.728/0.516, base (87.1M) worsens to 0.732/0.525, and
large (305M) worsens further to 0.743/0.530, with training data held
fixed. The paper recommends the 11.4M variant and documents the
regression explicitly — one of the cleanest published TS-FM data points
that naive parameter scaling can hurt when the pretraining corpus is
constant. Note the capability regression versus Moirai-1: multivariate
and covariate support are *dropped entirely*, so Moirai-2 is strictly
univariate.

## Efficiency-constrained / CPU / on-device

TTM dominates this axis. At 1M (TTM-Base) to 5M (TTM-Advanced)
parameters it reports ZS MSE comparable or better than Moirai,
TimesFM, and Chronos on the ETT/Weather/ECL suite while running in
0.01s/batch on CPU versus ~2,350s/batch for Chronos-Base
(TTM, Tables 1 and 3 — a 240,000x CPU speedup). Time-MoE-Base is
the second option: 50M activated parameters, explicitly designed
for CPU inference (Time-MoE, Section 3.2.3). The Chronos-2 small
variant (28M) trails the base 120M model by only ~1 skill-score point
on GIFT-Eval with ~2x faster inference (Chronos-2, Section 5.4), and
is probably the most accurate sub-50M choice.

## Current SOTA recommendations by use case

These are opinions informed by the leaderboard, not paper-independent
ground truth. Always validate on your own holdout.

- **Zero-shot univariate point or probabilistic forecasting, arbitrary
  series, accuracy first.** Chronos-2 (120M) for the best
  aggregate numbers across all three major suites (fev-bench 47.3,
  GIFT-Eval WQL 51.4, Chronos-II WQL 46.6; Chronos-2 Tables 3–5).
  TimesFM-2.5 is an almost-identical runner-up within 1-5 skill-score
  points on every suite.
- **Zero-shot with covariates (known exogenous features).** Chronos-2
  is effectively unrivalled. TabPFN-TS is the only other pretrained
  model with real covariate support; all other TS-FMs silently drop
  them.
- **Long-horizon LTSF suite, fine-tuning allowed.** Sundial-Large or
  Time-MoE-Large for best aggregate ETT/ECL/Weather numbers
  (Sundial Table 1). If compute is constrained, TTM-A at 5M params
  matches them (TTM Table 1).
- **Short-history / few observations per series.** Chronos-2 by the
  largest margin on Chronos Benchmark II, with TimesFM-2.5 second.
- **CPU deployment / edge / extreme latency budget.** TTM (1–5M) by
  240,000x on CPU vs Chronos-Base (TTM Table 3). Time-MoE-Base is the
  mid-point at 50M activated parameters.
- **Long-context (thousands of timesteps).** Time-MoE (trained at
  4096 tokens, Time-MoE Section 3.2.3), [Timer-XL](../papers/timer-xl.md), or Chronos-2-2K /
  Chronos-2 with the 8192-step post-training (Chronos-2 Section 5.4).

The "no single winner" part of the story is what makes
[methodology-caveats.md](methodology-caveats.md) necessary: whichever
paper you read last will, by construction, show its own model on top.

## Related wiki pages

- [leaderboard.md](leaderboard.md)
- [methodology-caveats.md](methodology-caveats.md)
- [efficiency-and-cost.md](efficiency-and-cost.md)
- [../foundation-models/taxonomy.md](../foundation-models/taxonomy.md)
