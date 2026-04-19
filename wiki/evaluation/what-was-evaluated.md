# What Each Paper Actually Evaluated

This page is a per-paper cross-reference: for each of the 23 papers
in the wiki, what metrics did the authors report, on which datasets,
under what protocol, against which baselines, and did the paper
produce probabilistic forecasts? Rows are sourced from the paper
leaves at `../papers/<slug>.md`. Cells use short tags; "—" means the
paper does not report on that axis. The purpose of the table is to
make it obvious why headline numbers across papers often cannot be
compared — different rows report different columns.

This is the "what evaluation was made" table. For *how to interpret*
the metrics themselves, see [metrics.md](metrics.md); for how to read
a benchmark protocol, see [protocols.md](protocols.md).

## Table

| Paper | Metrics | Datasets / suites | Protocol | Baselines | Probabilistic? | Link |
|---|---|---|---|---|---|---|
| TimesFM | scaled MAE (GM vs sNaive), MAE | [Monash](../datasets-benchmarks/monash-archive.md) (18), Darts, ETT | zero-shot; fixed horizon per dataset | sNaive, ARIMA, ETS, N-BEATS, llmtime | No | [../papers/timesfm.md](../papers/timesfm.md) |
| Chronos | MASE, WQL, sMAPE | 42 datasets (15 in-domain + 27 Benchmark II) | zero-shot on Bench II; in-domain elsewhere | sNaive, ETS, ARIMA, Theta, DeepAR, TFT, PatchTST, N-BEATS | Yes (categorical samples → WQL/CRPS) | [../papers/chronos.md](../papers/chronos.md) |
| Chronos-2 | MASE skill, WQL skill, SQL skill, median runtime | [GIFT-Eval](../datasets-benchmarks/gift-eval.md) (97 tasks), Chronos Bench II (27), fev-bench (100) | zero-shot; 95% bootstrap CIs | TiRex, TimesFM-2.5, Toto, Moirai-2.0, TabPFN-TS, Sundial, COSMIC, sNaive | Yes (9-quantile decoder) | [../papers/chronos-2.md](../papers/chronos-2.md) |
| MOMENT | MSE, MAE (forecasting); F1 (anomaly); accuracy (classification) | TSLib suite (ETT, Weather, ECL, ILI); TSB-UAD; UCR/UEA | zero-shot + linear-probe | PatchTST, DLinear, TimesNet, GPT4TS | No (point only) | [../papers/moment.md](../papers/moment.md) |
| MOIRAI | MSE, MAE, CRPS, MASE | Monash multivariate, [LOTSA](../datasets-benchmarks/lotsa.md) hold-out, LTSF (ETT, ECL, Weather), 6-dataset CRPS table | zero-shot (paper's claim), leakage w.r.t. LOTSA | PatchTST, TiDE, TFT, DeepAR, iTransformer, sNaive | Yes (Student-t mixture head) | [../papers/moirai.md](../papers/moirai.md) |
| Moirai-MoE | aggregate MAE (GM vs sNaive), CRPS, MASE | Monash 29-dataset, 10-dataset held-out (Table 2), 39-dataset | zero-shot, explicitly LOTSA-disjoint | Chronos, TimesFM, Moirai, iTransformer, PatchTST, DeepAR, WaveNet, FFNN | Yes (Student-t mixture) | [../papers/moirai-moe.md](../papers/moirai-moe.md) |
| Timer | MSE, MAE | UTSD benchmarks, ETT, PEMS, small data-scarce subsets | in-domain + few-shot, emphasis on data-scarce | PatchTST, iTransformer, DLinear, TimesNet | No (point only) | [../papers/timer.md](../papers/timer.md) |
| Timer-XL | MSE, MAE | LTSF (ETT, ECL, Weather), ERA5-Large (4920 stations, ~2880 ctx) | zero-shot long-context; fixed `h` set | Moirai-Large, TimesFM, MOMENT, PatchTST | No (point only) | [../papers/timer-xl.md](../papers/timer-xl.md) |
| Timer-S1 | MASE, CRPS (quantile-weighted) | GIFT-Eval (primary) | zero-shot post-training; GIFT-Eval leakage scrubbed from corpus | GIFT-Eval leaderboard models | Yes (9-quantile decoder) | [../papers/timer-s1.md](../papers/timer-s1.md) |
| Lag-Llama | CRPS | 27-dataset probabilistic suite from Monash + related archives | zero-shot + fine-tune; scaling study | DeepAR, Transformer baselines, sNaive | Yes (Student-t head) | [../papers/lag-llama.md](../papers/lag-llama.md) |
| TimeGPT | rMAE, rRMSE, empirical coverage | 300k-series internal suite (monthly/weekly/daily/hourly) | zero-shot API; conformal intervals | Theta, ETS, CES, LightGBM, LSTM, DeepAR, TFT, N-HiTS | Yes (conformal intervals, not a proper distribution) | [../papers/timegpt.md](../papers/timegpt.md) |
| Time-MoE | MSE, MAE | LTSF 6-dataset (ETTh1/2, ETTm1/2, Weather, ECL), Monash held-out | zero-shot + in-distribution | TimesFM, Moirai, Chronos, PatchTST, DLinear, iTransformer | No (point only) | [../papers/time-moe.md](../papers/time-moe.md) |
| Time-LLM | MSE, MAE | TSLib LTSF suite (ETT, Weather, ECL, Traffic, ILI) | zero-shot + few-shot via adapter | GPT4TS, PatchTST, DLinear, TimesNet | No (point only) | [../papers/time-llm.md](../papers/time-llm.md) |
| GPT4TS | MSE, MAE; F1; accuracy | TSLib LTSF + classification/anomaly (6 task families) | in-domain (per-dataset fit of adapter) | TimesNet, PatchTST, DLinear, TS2Vec | No (point only for forecasting) | [../papers/gpt4ts.md](../papers/gpt4ts.md) |
| LLMTime | NLL, CRPS, sMAPE | Darts, Monash, Informer benchmark | zero-shot (no fine-tune, no adapter) | sNaive, ARIMA, N-BEATS, SM-GP | Yes (digit-distribution continuous density) | [../papers/llmtime.md](../papers/llmtime.md) |
| TTM | MSE, MAE | LTSF (ETT, Weather, ECL, Traffic) | zero-shot + few-shot + fine-tune | Chronos, Moirai, TimesFM, PatchTST, iTransformer, DLinear | No (point only) | [../papers/ttm.md](../papers/ttm.md) |
| UniTS | MSE, MAE; F1; accuracy | 38-dataset TSLib-style suite (forecasting/class/anomaly/imputation) | in-domain + few-shot transfer | 12 forecasting, 20 classification, 18 anomaly, 16 imputation baselines including text-LLMs | No (point forecasts only) | [../papers/units.md](../papers/units.md) |
| TOTEM | MSE, MAE; F1 | TSLib + zero-shot held-out domains (neuroscience, river flow, birth rate) | specialist + generalist; ~500 experiments | 14 forecast, 17 imputation, 19 anomaly baselines | No (point only) | [../papers/totem.md](../papers/totem.md) |
| Sundial | MASE, CRPS, MSE, MAE | GIFT-Eval (97 tasks, Table 2), Chronos Bench II, LTSF (Table 1) | zero-shot | TimesFM, Moirai, Chronos, Time-MoE, PatchTST, iTransformer, N-BEATS, TabPFN-TS | Yes (flow-matching generative) | [../papers/sundial.md](../papers/sundial.md) |
| Mamba4Cast | MASE, ranks (critical-difference diagram) | 17-dataset Monash-style suite | zero-shot + latency vs horizon | Chronos-Base/Small, DeepAR, AutoARIMA, sNaive | No (point only; compared to probabilistic Chronos on MASE only) | [../papers/mamba4cast.md](../papers/mamba4cast.md) |
| Moirai 2.0 | MASE, CRPS (9-quantile pinball) | [GIFT-Eval](../datasets-benchmarks/gift-eval.md) only (Table 1) | zero-shot; GIFT-Eval TrainTest train split used during pretraining, test split held out | 30 filtered FMs from the GIFT-Eval leaderboard (TimesFM-2.5, TiRex, Toto, Chronos-Bolt, Sundial, ...) | Yes (9-quantile pinball head) | [../papers/moirai-2.md](../papers/moirai-2.md) |
| SEMPO | MSE, MAE | TSLib zero-shot (7 datasets × 4 horizons = 14 splits, Table 1); 5% few-shot on TSLib; 16-dataset average including 9 GIFT-Eval non-overlapping datasets (Appendix D) | zero-shot after Stage 1 + MoP-tuning on target domain; full 5%-few-shot variant | Time-MoE-B/L, Timer, Moirai-S/B/L, Chronos-S/B/L, TimesFM, Moment, TTM | No (point only) | [../papers/sempo.md](../papers/sempo.md) |
| TSPulse | VUS-PR (anomaly), accuracy (classification), MSE (imputation), retrieval F1 | TSB-AD (uni + multi), UEA-29, LTSF imputation (ETTh1/2, ETTm1/2, Weather, ECL), UCR+synthetic similarity | zero-shot + short fine-tune; no forecasting benchmarks reported | MOMENT, UniTS, VQShape, Chronos, TimesFM | No (not a forecaster; `Headpred` injects only a few points into semantic embedding) | [../papers/tspulse.md](../papers/tspulse.md) |

## Derived observations

These observations only summarize what is in the table plus the
paper notes; none are fabricated from external sources.

1. **MASE is near-universal on Monash.** Every paper that reports on
   Monash or a Monash-derived held-out set (TimesFM, Chronos,
   Chronos-2, Moirai, Moirai-MoE, Sundial, Timer-S1, Lag-Llama,
   Mamba4Cast) reports MASE or a scaled-MAE equivalent. MASE is the
   lingua franca for cross-dataset point forecasting.
2. **LTSF-style MSE is the other dominant axis.** The LTSF 6-dataset
   suite (ETT / Weather / ECL / Traffic / ILI) is the default MSE
   benchmark for TimesFM, Moirai, Time-MoE, Timer-XL, Sundial, TTM,
   MOMENT, Time-LLM and GPT4TS — roughly half the wiki. These
   numbers are z-normalized MSE, not raw, and are only comparable
   across papers that used the same TSLib split and the same
   context length.
3. **CRPS is reported almost exclusively by the encoder-decoder and
   quantile-decoder cluster.** MOIRAI, Moirai-MoE, Chronos,
   Chronos-2, Sundial, Lag-Llama, Timer-S1 and LLMTime report CRPS
   or a direct WQL equivalent. Point-only decoders (TimesFM, Timer,
   Timer-XL, TTM, MOMENT, UniTS, TOTEM, Time-LLM, GPT4TS, Time-MoE,
   Mamba4Cast) generally do not — Mamba4Cast explicitly only reports
   MASE against Chronos despite Chronos being probabilistic.
4. **No paper in the wiki reports calibration diagrams alongside
   CRPS.** PIT histograms and reliability diagrams are absent from
   every paper leaf in this wiki, including the papers that depend
   on calibrated distributions (Chronos, MOIRAI, Lag-Llama, Sundial,
   Chronos-2). CRPS and WQL are used as stand-alone measures of
   "probabilistic quality", which is a gap noted in
   [probabilistic-evaluation.md](probabilistic-evaluation.md).
5. **Only Chronos-2 reports bootstrap CIs.** Chronos-2 is the only
   paper in the table that attaches 95% bootstrap confidence
   intervals to its aggregate skill-score numbers. All other papers
   report point estimates only; small wins (< 2% skill score, < 0.02
   raw MASE) are not statistically meaningful without CIs.
6. **Multivariate / covariate-aware evaluation is thin.** Only
   Chronos-2 and Timer-XL specifically stress multivariate or
   covariate-informed forecasting at scale (Chronos-2 on fev-bench's
   covariates subset, Timer-XL on ERA5-Large). No paper reports an
   energy score or variogram score. This is the clearest gap in the
   wiki's current benchmark coverage and is discussed in
   [../research/open-problems.md](../research/open-problems.md).
7. **TimeGPT is an outlier in baseline style.** It is the only paper
   in the table that reports empirical coverage of conformal
   intervals as its probabilistic claim (rather than CRPS or WQL)
   and the only one evaluated against a mixed industrial baseline
   stack including LightGBM. Its "scaling laws" prose in the paper
   has no accompanying curve or ablation, which the paper leaf
   flags explicitly.
8. **Zero-shot leakage is acknowledged by only a minority.** The
   papers that explicitly audit their own pretraining-vs-evaluation
   overlap are Chronos-2 (GIFT-Eval partial overlap, stated),
   Timer-S1 (removes GIFT-Eval from [TimeBench](../datasets-benchmarks/timebench.md)), and Moirai-MoE
   (LOTSA-disjoint hold-outs). TimesFM, Chronos (v1) and several
   others inherit the standard footnote. See
   [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md)
   section 1.

## Related wiki pages

- [metrics.md](metrics.md) — what each metric tag in the table
  actually means.
- [protocols.md](protocols.md) — what "zero-shot" and "in-domain"
  mean in each row.
- [comparability-checklist.md](comparability-checklist.md) — a
  practical checklist for deciding whether two rows are actually
  comparable.
- [../benchmarks/leaderboard.md](../benchmarks/leaderboard.md) — the
  actual numbers for the rows above.
- [../research/comparison-matrix.md](../research/comparison-matrix.md)
  — the complementary architecture/training comparison matrix.
