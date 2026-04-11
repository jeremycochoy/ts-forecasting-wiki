# Benchmarks — Head-to-Head Results

This section is the *performance* layer of the wiki. While
[datasets-benchmarks/](../datasets-benchmarks/README.md) describes the
corpora and evaluation suites themselves, this section reads across
them: which models win on which suite, under which metric, and with
which caveats.

Every numeric claim on these pages is attributed to a specific paper,
table or figure. Where numbers are not directly comparable (different
context lengths, normalization, or splits), the caveats page says so
rather than forcing a false ranking.

## Sub-pages

- [leaderboard.md](leaderboard.md) — normalized head-to-head tables on
  [Monash](../datasets-benchmarks/monash-archive.md), [Chronos](../papers/chronos.md) Benchmark II, [GIFT-Eval](../datasets-benchmarks/gift-eval.md), fev-bench, and the LTSF
  long-horizon suite.
- [state-of-the-art.md](state-of-the-art.md) — narrative "which model
  wins where," organized by forecasting regime, ending in
  recommendations by use case.
- [methodology-caveats.md](methodology-caveats.md) — how to read
  TS-FM benchmark results: leakage, normalization, metric semantics,
  aggregation, and contested findings.
- [efficiency-and-cost.md](efficiency-and-cost.md) — the second axis
  beyond accuracy: parameters, training compute, inference latency,
  memory, and CPU deployability.
- [univariate-benchmarking.md](univariate-benchmarking.md) — a
  practical recommendation page for readers who have built a
  univariate-only model (one series per call, no covariates) and
  want to pick the right suite to compare against 2024–2026 TS-FM
  SOTA.

## How to interpret these numbers

Every column in [leaderboard.md](leaderboard.md) is an instance of a
specific metric, computed under a specific protocol, against a
specific set of baselines. For the *pedagogy* of those choices — what
[MASE](../evaluation/metrics.md#17-mase--mean-absolute-scaled-error), [CRPS](../evaluation/metrics.md#21-crps--continuous-ranked-probability-score), [WQL](../evaluation/metrics.md#23-wql--weighted-quantile-loss) and MSE actually mean, how seasonal periods enter
them, what "zero-shot" requires, and what each paper in the wiki
actually reported — see [../evaluation/README.md](../evaluation/README.md).
The methodology-caveats page here focuses on the specific asterisks
behind specific rows; the evaluation section is the more general
reference.

## Related wiki pages

- [../datasets-benchmarks/README.md](../datasets-benchmarks/README.md) —
  descriptions of the corpora and eval suites themselves.
- [../foundation-models/taxonomy.md](../foundation-models/taxonomy.md) —
  the cluster map of TS-FMs referenced throughout this section.
- [../evaluation/README.md](../evaluation/README.md) — the evaluation
  methodology layer (metrics, protocols, baselines, per-paper table).
