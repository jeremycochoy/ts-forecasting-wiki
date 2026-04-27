# Failure Modes Catalog

Every TS foundation model is good at some things and bad at others.
The good things are in the abstract of every paper; the bad things
are in appendices, qualitative-analysis subsections, and follow-up
papers' critiques. This page collects them in one place so a reader
who is about to deploy a model knows what to expect. Each row cites
a specific paper section or follow-up critique.

For broader architectural trade-offs, see
[../architectures/decoder-only-autoregressive.md](../architectures/decoder-only-autoregressive.md),
[../architectures/masked-encoder.md](../architectures/masked-encoder.md),
and the per-paper "Limitations and open critiques" blocks linked
below.

## Per-model documented weaknesses

| Model | Failure mode | Evidence |
|---|---|---|
| [Chronos](../papers/chronos.md) | Strong trend overflow. Binned values live in `[-15s, +15s]` where `s` is context mean absolute value; series that drift beyond that range clip. | Chronos, Sec 5.7 and Fig 13 (Air Passengers qualitative example). |
| [Chronos](../papers/chronos.md) | Categorical CE is not distance-aware; adjacent bins are as far apart as distant bins. Ordinal or label-smoothed variants left to future work. | Chronos, Sec 6 (discussion). |
| [Chronos](../papers/chronos.md) | Univariate only, no covariate handling; addressed in the follow-up. | Chronos, Sec 6; [Chronos-2](../papers/chronos-2.md) positioning. |
| [TimesFM](../papers/timesfm.md) | Point-forecast only, no probabilistic head in v1. Direct CRPS/WQL comparisons with Chronos / Lag-Llama need a head retrofit. | TimesFM, App A.1 (future work). |
| [TimesFM](../papers/timesfm.md) | Google Trends plus Wiki Pageviews corpus is narrow in character: frequency mix is dominated by hourly-Wiki and daily-Trends. Generalization on rare granularities (yearly / quarterly) is specifically helped by synthetic augmentation (Fig 3d), which hints the base prior does not cover them. | TimesFM, Fig 3d ablation. |
| [TimesFM](../papers/timesfm.md) | Output-patch length has a floor below which short-series (monthly, yearly) cannot be represented; the paper notes the trade-off explicitly. | TimesFM, Sec 3 (Longer output patches). |
| [MOIRAI](../papers/moirai.md) | O((V·T)^2) attention cost from flattening all variate-time tokens into one sequence; scales badly in the number of variates. | [Chronos-2](../papers/chronos-2.md) Sec 5 positioning; Group attention is the stated fix. |
| [MOIRAI](../papers/moirai.md) | Multi-patch-size frequency heuristic is brittle on unseen frequencies; the follow-up argues routing subsumes it. | [Moirai-MoE](../papers/moirai-moe.md), Sec 1 (motivation). |
| [MOIRAI](../papers/moirai.md) | Non-monotone scaling on LTSF: MOIRAI-Large ETTh1 MSE (0.510) is worse than MOIRAI-Base (0.434); the curve inverts. | MOIRAI, Table 6 (arXiv:2402.02592); reproduced in Sundial Table 1. |
| [Moirai-MoE](../papers/moirai-moe.md) | Cluster-centroid gate is seeded from a *pretrained* MOIRAI, so the recipe depends on a two-stage pipeline; a greenfield practitioner cannot reproduce in one run. | Moirai-MoE, Sec 3. |
| [Time-MoE](../papers/time-moe.md) | Time-300B is skewed: the "Nature" domain contributes ~90% of observations, so clean scaling is partially a study of one domain's breadth. | Time-MoE, Table 1. |
| [Time-MoE](../papers/time-moe.md) | Point regression only (Huber); no calibrated predictive density. | Time-MoE, Sec 3.2. |
| [Timer](../papers/timer.md) | Limited native support for arbitrary output lengths; long horizons may be truncated. | Time-MoE, Sec 1 (explicit critique of Timer). |
| [MOMENT](../papers/moment.md) | Fixed input context length; no variable-context support at inference. | Time-MoE, Sec 1 (explicit critique of MOMENT). |
| [MOMENT](../papers/moment.md) | Point-only forecasting; probabilistic head not first class. | [what-was-evaluated.md](../evaluation/what-was-evaluated.md) row. |
| [Sundial](../papers/sundial.md) | Flow-matching inference is multi-step: each patch costs `K` push-forward iterations, and probabilistic output needs multiple noise seeds. "Just-in-time" is an accuracy-vs-cost trade-off. | Sundial, Sec 4.4 ablation. |
| [Sundial](../papers/sundial.md) | TimeBench auditing is loose: synthetic / real-world mixture and per-domain weighting are not fully published, so whether scaling gains derive from a few high-volume domains is unclear. | Sundial, Sec 4 data card. |
| [TTM](../papers/ttm.md) | Point regression only; no native probabilistic head. | TTM, Sec 4. |
| [TTM](../papers/ttm.md) | True multivariate zero-shot is limited because cross-channel mixing is a fine-tune-only addition. | TTM, Sec 4 (decoder channel mixer). |
| [TTM](../papers/ttm.md) | Benchmarks are biased toward the TSLib long-horizon suite (ETT, Weather, ECL, Traffic). GIFT-Eval coverage is missing in the published version. | TTM, Sec 4.1 (evaluation choice). |
| [Lag-Llama](../papers/lag-llama.md) | Single-parametric Student-t head; the authors explicitly defer richer distribution heads (e.g. IQN) to follow-ups and note the training-side difficulties. | Lag-Llama, Sec 4.3. |
| [Lag-Llama](../papers/lag-llama.md) | No first-class support for real multivariate dynamics; features are encoded through lag tokens. | Lag-Llama, Sec 4 (covariates discussion). |
| [LLMTime](../papers/llmtime.md) | Digit-token tokenization is expensive at inference; one number becomes multiple tokens, so decoding is slow. | LLMTime, Sec 2 and Fig 4. |
| [Time-LLM](../papers/time-llm.md) | Frozen LLM backbone inherits the LLM's idiosyncratic priors; no new pretrained TS artifact. | [llm-reprogramming](../architectures/llm-reprogramming.md). |
| [GPT4TS](../papers/gpt4ts.md) | Per-dataset fine-tuning of adapter layers; not a released checkpoint that can be deployed as-is. | GPT4TS, Sec 3. |
| [Chronos-2](../papers/chronos-2.md) | Multivariate training data is almost entirely synthetic (TCM / multivariatizer priors); the gap to real causal structures is unaudited. | Chronos-2, Sec 4. |
| [Chronos-2](../papers/chronos-2.md) | Not peer-reviewed; many claims rest on fev-bench, a benchmark co-authored by the same AWS team. | Chronos-2, technical report status. |
| [Timer-S1](../papers/timer-s1.md) | Value-flipping augmentation assumes sign-symmetric dynamics; breaks silently on strictly non-negative series (counts, utility demand). | Timer-S1, Sec 3 (augmentation description). |
| [Timer-S1](../papers/timer-s1.md) | Weights and code announced but not released at submission, so GIFT-Eval numbers are not independently verifiable yet. | Timer-S1, status. |
| [Mamba4Cast](../papers/mamba4cast.md) | Synthetic-only pretraining: no real time series touched. Real-world distribution shift coverage depends entirely on the sampling prior. | Mamba4Cast, Sec 3. |
| [Mamba4Cast](../papers/mamba4cast.md) | Point output; compares to probabilistic Chronos only on MASE, not on CRPS. | Mamba4Cast, Sec 4 evaluation. |
| [TimeGPT](../papers/timegpt.md) | Closed API, undisclosed size, undisclosed training data; "scaling laws" prose in the paper has no accompanying curve. | TimeGPT leaf; [what-was-evaluated.md](../evaluation/what-was-evaluated.md). |
| [LaT-PFN](../papers/lat-pfn.md) | Univariate only, no covariate handling; emergent-patch finding is qualitative (Figures 9–10) without an ablation linking it to a specific architectural choice. | LaT-PFN, Sec 2 scope; Sec 5 (qualitative observation). |
| [LaT-PFN](../papers/lat-pfn.md) | 100-bin categorical decoder; not a calibrated quantile family. CRPS / coverage analysis not reported. | LaT-PFN, Sec 3.3 (decoder description). |
| [LaT-PFN](../papers/lat-pfn.md) | Heavily-tuned hyperparameters reported with extreme precision (`λ_latent = 3.77e-3`, `λ_si = 1e-7`); robustness to these choices not characterized. | LaT-PFN, Sec 3.3 loss. |
| [TS-JEPA](../papers/ts-jepa.md) | Tiny experimental footprint: 2 attention heads, embedding dim 128, 10 patches per series, 5 datasets total. No scaling study. | TS-JEPA, Sec 3 (Architecture). |
| [TS-JEPA](../papers/ts-jepa.md) | Loses to autoregressive on short-term forecasting on all 3 forecasting datasets — the regime forecasting-trained TS-FMs optimize for. | TS-JEPA, Table 2 (short-term MSE). |
| [TS-JEPA](../papers/ts-jepa.md) | No ablation of 70% mask ratio, L1 vs L2 loss, EMA decay m=0.998, or predictor depth/width. | TS-JEPA, Sec 5 conclusion. |
| [MTS-JEPA](../papers/mts-jepa.md) | Anomaly-prediction-only; cannot be placed alongside the rest of the TS-FM cohort on Monash / GIFT-Eval / fev-bench. | MTS-JEPA, Sec 4 evaluation. |
| [MTS-JEPA](../papers/mts-jepa.md) | High hyperparameter complexity (K codebook size, τ temperature, λ_f, λ_c, γ, λ_r, two EMA decays); sensitivity not characterized. | MTS-JEPA, Sec 3.4 loss. |
| [MTS-JEPA](../papers/mts-jepa.md) | Without the soft codebook bottleneck, performance collapses to near-random (paper Table 3 ablation), suggesting JEPA on continuous TS may not be stable on EMA + stop-gradient alone. | MTS-JEPA, Table 3 (`w/o Codebook Module`). |
| **Cross-cutting (Cluster 8)** | None of the three JEPA papers reports on Monash, GIFT-Eval, Chronos Benchmark II or fev-bench, so head-to-head comparison with the dominant forecasting-first TS-FM lineage remains indirect. Closing this benchmark gap is the cleanest 2026 follow-up. | LaT-PFN / TS-JEPA / MTS-JEPA evaluation sections. |

## Cross-cutting failure modes

Several failures recur across model families and are worth reading
as first-class patterns, not as one-off bugs.

- **Autoregressive rollout compounding.** Every decoder-only AR
  model (TimesFM, Timer, Time-MoE, Lag-Llama) accumulates error as
  the horizon grows. TimesFM's `P_out > P_in` trick, MOIRAI's
  bidirectional mask, Chronos-2's direct multi-horizon head, Sundial's
  flow matching, and Timer-S1's Serial-Token Prediction are all
  different answers to the same underlying problem. No paper has
  published a horizon-stress plot for all six families on the same
  benchmark; see [open-problems.md](open-problems.md#6-long-horizon-forecasting-and-autoregressive-rollout-error).
- **Zero-shot leakage.** LOTSA, Time-300B, Time Series Pile and the
  Chronos training mix all overlap with the Monash archive and LTSF
  suites. Moirai-MoE's Figure 3 explicitly flags Chronos and TimesFM
  with a "*not ZS" asterisk on Monash; Chronos-2 acknowledges partial
  overlap with GIFT-Eval training splits; Timer-S1 removes GIFT-Eval
  from its TimeBench curation. Everyone else inherits the standard
  footnote silently. See
  [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md).
- **Probabilistic miscalibration is undiagnosed.** No paper in the
  wiki reports PIT histograms or reliability diagrams alongside CRPS
  or WQL; all six models that produce distributions (Chronos,
  MOIRAI, Lag-Llama, Sundial, Chronos-2, Timer-S1) treat CRPS as a
  stand-alone measure of "probabilistic quality." Miscalibrated but
  sharp distributions can post excellent CRPS. See
  [../evaluation/probabilistic-evaluation.md](../evaluation/probabilistic-evaluation.md).
- **Covariate-blind models silently drop covariates.** On the
  fev-bench covariates subset (42 tasks), every pretrained model
  except Chronos-2 and TabPFN-TS sits in a 28–40 SQL-skill band
  because they ignore the covariates entirely (Chronos-2, Fig 4b).
  The failure is silent — the model emits a forecast, just not one
  that uses the covariate signal.
- **Non-monotone parameter scaling on LTSF.** MOIRAI-Large is worse
  than MOIRAI-Base on ETTh1 (0.510 vs 0.434 MSE, MOIRAI Table 6);
  Chronos-Large on ETTm1 is worse than Chronos-Base in several
  reports. LTSF results therefore cannot be read as "larger =
  better" inside a model family — see
  [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md).

## Related wiki pages

- [comparison-matrix.md](comparison-matrix.md) — the architectural
  side-by-side.
- [open-problems.md](open-problems.md) — what is still open for
  each failure category.
- [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md)
  — leakage, scaling asterisks, comparability footnotes.
- [../evaluation/probabilistic-evaluation.md](../evaluation/probabilistic-evaluation.md)
  — calibration gaps.
- [../evaluation/what-was-evaluated.md](../evaluation/what-was-evaluated.md)
  — per-paper metric and baseline coverage.
