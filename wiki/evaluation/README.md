# Evaluation — Metrics, Protocols, and What Each Paper Actually Ran

This section is the *methodology* layer of the wiki. Where
[../benchmarks/](../benchmarks/README.md) gives you the raw numbers and
[../datasets-benchmarks/](../datasets-benchmarks/README.md) describes the
corpora those numbers are computed on, this section explains how to
read a TS forecasting evaluation in the first place: what each metric
means, which baselines to compare against, how protocols differ, and
what each paper in the wiki actually reports. A number on a
leaderboard is only as meaningful as the procedure behind it; this
section teaches the procedure.

## Sub-pages

- [metrics.md](metrics.md) — the flagship reference. Every point and
  probabilistic metric that appears in TS-FM papers, with formulas,
  interpretation, failure modes, and which papers use which.
- [seasonality-and-baselines.md](seasonality-and-baselines.md) — why
  Seasonal Naive is the baseline, how to pick the seasonal period `m`,
  and what classical and deep baselines belong in a fair comparison.
- [probabilistic-evaluation.md](probabilistic-evaluation.md) — CRPS,
  WQL, quantile loss, calibration vs. sharpness, and how to evaluate
  sample-based probabilistic forecasters.
- [protocols.md](protocols.md) — splits, rolling-origin, horizon and
  context-length conventions, the real meaning of "zero-shot", and
  statistical significance.
- [what-was-evaluated.md](what-was-evaluated.md) — a one-row-per-paper
  table summarizing metrics, datasets, protocol, and baselines for all
  20 wiki papers.
- [comparability-checklist.md](comparability-checklist.md) — a short
  practical checklist for deciding whether two papers' numbers can be
  compared at all.

## Related wiki pages

- [../benchmarks/leaderboard.md](../benchmarks/leaderboard.md) — the
  actual numbers this section teaches you to read.
- [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md)
  — the asterisks hiding behind those numbers.
- [../benchmarks/univariate-benchmarking.md](../benchmarks/univariate-benchmarking.md)
  — practical recommendation for readers whose model is
  univariate-only and who want a head-to-head comparison against
  2024–2026 TS-FM SOTA under the same metric conventions this
  section defines.
- [../datasets-benchmarks/README.md](../datasets-benchmarks/README.md)
  — the corpora and evaluation suites themselves.
- [../research/glossary.md](../research/glossary.md) — one-line
  definitions linking back into this section.
