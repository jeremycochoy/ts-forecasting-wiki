# How to Read TS-FM Benchmark Results

This page is the intellectual-honesty layer of the benchmark section.
The [leaderboard](leaderboard.md) tables look authoritative, but
every row hides at least one choice that can swing a model's
apparent rank by several points. Read this page before quoting any
number.

## 1. "Zero-shot" is paper-relative, not universal

The single biggest footnote. TimesFM, Chronos and Chronos-2 all
label themselves zero-shot on Monash and on subsets of GIFT-Eval,
but Moirai-MoE's Figure 3 caption (arXiv:2410.10469) explicitly
marks Chronos-Small / Base / Large and TimesFM with an asterisk
because "these models used the evaluation datasets here in their
pretraining corpora." The Chronos-2 paper acknowledges the mirror
problem in Section 5.1: its own pretraining corpus has "partial
overlap with the training portions of some GIFT-Eval datasets,"
though the test portions were scrubbed.

A cell that reads `Model X, zero-shot, Monash` should therefore be
mentally translated to `Model X, as evaluated by X's authors, with
whatever their training-set definition was at submission time.` The
only Monash numbers that are strictly leakage-free across
*all compared* models are the ones from Moirai-MoE's own lineage.
For the other families, see the model card of the specific release
for which datasets were blacklisted.

## 2. Skill score vs. GM-relative MAE vs. raw MASE

Different papers normalize differently. The conversions:

- **Skill score (S)**: `S = 1 − GM_relative_error`, reported as a
  percentage. Higher is better. Used by fev-bench and Chronos-2.
- **Average rank (R)**: rank across N models, averaged over datasets.
  Lower is better. Used by the GIFT-Eval public leaderboard.
- **Geometric-mean-relative MASE / CRPS (G)**: per-dataset metric
  divided by Seasonal Naive, then geometrically averaged. Lower is
  better. Used by Sundial's Table 2.
- **Win rate (W)**: fraction of pairwise matchups won, converted to
  a percentage. Higher is better. Used by fev-bench.

Chronos-2 gives the explicit identity `R = 1 + (1 − W/100)(N−1)`
and `G = 1 − S/100` (Chronos-2, Section 5.1). So a 51.4% skill score
corresponds to G ≈ 0.486, directly comparable to Sundial's reported
GIFT-Eval CRPS of 0.472. The surface-level "51.4" and "0.472" look
unrelated but are near-identical aggregates on the same suite,
differing mostly because Sundial's table is from Feb 2025 and
Chronos-2's is from Oct 2025 (the leaderboard itself moved).

## 3. Context and horizon conventions differ

LTSF papers canonically report {96, 192, 336, 720} horizons, but
what they compute differs:

- TimesFM reports *scaled MAE geometric-mean* with Seasonal Naive
  as the baseline (TimesFM, Figure 2a).
- MOIRAI Table 6 reports raw MSE / MAE averaged over the four
  horizons.
- Sundial Table 1 reports the same but at different context lengths
  depending on how the target-series library was called.
- Chronos evaluated on a subset of LTSF using its own binning
  conventions.

An ETTh1 MSE of 0.434 (MOIRAI-Base) and 0.480 (MOIRAI-Large quoted
in Sundial Table 1) came from the *same* MOIRAI checkpoints on the
*same* dataset — the discrepancy is almost certainly context-length
or aggregation-variant. Do not cross-reference LTSF MSE values
between papers without checking the evaluation protocol first.

## 4. Normalization (RevIN and robust scaling) is hidden

Almost every modern TS-FM applies some form of
reversible-instance-normalization at inference time: Chronos-2
uses "robust scaling" (Chronos-2, Figure 1), MOIRAI applies a
per-context standardization, Sundial normalizes per patch. These
preprocessors can change MSE by >10% relative. A model quoted with
and without RevIN is effectively a different model. LTSF tables in
the literature rarely state which variant is running.

## 5. Metric semantics

- **MSE / MAE**: point forecasting only, scale-dependent; sensitive
  to outliers and to whether RevIN is on.
- **MASE**: MAE divided by in-sample Seasonal Naive MAE; scale-free,
  robust across datasets, the preferred aggregator.
- **CRPS**: Continuous Ranked Probability Score; proper probabilistic
  metric that rewards well-calibrated quantile predictions.
  Reduces to MAE for a point forecast. Used by MOIRAI Table 5.
- **WQL**: Weighted Quantile Loss (pinball loss weighted by the
  quantile level); similar to CRPS but computed over a fixed
  quantile grid. Used by GIFT-Eval and Chronos Benchmark II.
- **SQL**: Scaled Quantile Loss; fev-bench's WQL variant scaled by
  Seasonal Naive. Used by fev-bench and Chronos-2.

CRPS, WQL and SQL are all proper probabilistic scores — lower is
better — but their *levels* are not directly comparable. A model
with WQL 0.47 on GIFT-Eval and a model with SQL 0.47 on fev-bench
are not "equally good."

## 6. Aggregate-mean vs per-dataset variance

Average numbers hide enormous per-dataset spread. On the fev-bench
pairwise skill-score plot (Chronos-2, Figure 2b), a 2-point
aggregate lead can come from *huge* wins on 3 datasets and losses
on 8 others. The published bootstrap 95% CIs are the correct way
to read those gaps. Chronos-2's lead over TiRex is 4.7 points on
fev-bench SQL with a CI roughly (3.6, 5.9) — statistically
significant. Its lead over TimesFM-2.5 on GIFT-Eval WQL is 0.4
points — not statistically significant.

## 7. Contested / disputed results

- **Moirai-Large is worse than Moirai-Base on ETTh1.** Both MOIRAI's
  own Table 6 (0.434 → 0.510 MSE) and Sundial's Table 1 (0.480 for
  Large) show Large doing *worse*. This is the clearest known
  reverse-scaling instance in the TS-FM literature and is discussed
  in the Moirai-MoE paper as motivation for mixture-of-experts.
- **Chronos vs Moirai on Monash.** Moirai-MoE Figure 3 shows
  Chronos dominating Moirai on Monash (aggregate MAE 0.70 vs 0.89)
  *but* with the asterisk that Chronos had Monash in its pretraining
  corpus. The "real" zero-shot comparison there is closer to Moirai-MoE
  0.63 vs no Chronos data point — Moirai-MoE is the honest winner.
- **Sundial on Chronos Benchmark II vs Chronos-2's ranking of Sundial.**
  Sundial's own FEV-leaderboard plot (Sundial Figure 4) ranks it at the
  top; Chronos-2 Table 5 ranks it near the bottom (WQL skill score
  24.1, vs Chronos-2 46.6). Part of this is date (Feb → Oct 2025,
  new baselines added), part is the specific subset evaluated.
- **TimesFM on LTSF.** Because TimesFM's pretraining corpus includes
  Traffic, ETT and Wiki, most modern papers mark TimesFM's LTSF
  results with an asterisk or omit them entirely (Time-MoE omits
  TimesFM Weather, Chronos-2 does not use LTSF at all).

## 8. What this means for the reader

Treat any single-paper claim of "state-of-the-art" as a *local*
claim: SOTA on the benchmarks that paper reports, in the
configuration that paper used. The honest cross-paper picture is
the one in [state-of-the-art.md](state-of-the-art.md) — four or
five models clustered at the top of most suites, with the
meaningful separations appearing only on regimes that isolate a
specific capability (covariates, short context, efficiency).

## Related wiki pages

- [leaderboard.md](leaderboard.md)
- [state-of-the-art.md](state-of-the-art.md)
- [../concepts/probabilistic-forecasting.md](../concepts/probabilistic-forecasting.md)
- [../concepts/zero-shot-forecasting.md](../concepts/zero-shot-forecasting.md)
- [../evaluation/protocols.md](../evaluation/protocols.md) — splits,
  horizons, context lengths, zero-shot discipline, bootstrap CIs.
- [../evaluation/metrics.md](../evaluation/metrics.md) — the metric
  definitions behind every column above, including the WQL / SQL /
  skill-score / GM-relative-MASE conversions referenced in §2.
