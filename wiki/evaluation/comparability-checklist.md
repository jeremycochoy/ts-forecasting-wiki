# Comparability Checklist

A practical tool for the moment when you have two numbers from two
TS forecasting papers and you want to know if you can honestly put
them next to each other. Run through the seven items below. **If
three or more fail, the comparison is not meaningful — go open the
original tables and read the protocol sections directly.** No amount
of leaderboard aggregation will recover a comparison that fails on
multiple axes.

## The checklist

### 1. Same metric

Surprisingly often violated. "Both papers report MSE on ETTh1 at
`h = 96`" is not the same as "both papers report comparable MSE".
Watch for:

- MSE on raw series vs MSE on z-normalized series (LTSF convention
  is the latter, silently).
- MASE vs MASE-skill-score vs GM-relative MASE (the three can have
  identical-looking values with wildly different meaning — see
  [metrics.md](metrics.md) section 1.7).
- CRPS vs WQL vs SQL. All proper probabilistic scores, all with
  different normalization, numerical values between them are not
  interchangeable.
- "MAE" scaled by the per-series mean (TimesFM) vs "rMAE" relative
  to a baseline (TimeGPT) — two different quantities that share the
  name "relative MAE."

### 2. Same target series and same split

Two papers can both report "ETTh1" and mean different windows.
Check:

- The TSLib canonical split (`train:val:test` ≈ `12:4:4` months for
  ETTh1) is the defensible default but not universal.
- Monash has had at least two revisions; pinning the version matters.
- GIFT-Eval and fev-bench publish explicit split indices; papers
  that *deviate* from those indices should say so.

### 3. Same horizon `h` and context length `C`

Horizon-dependent rank reversals are routine. A paper reporting
ETTh1 MSE "averaged over `{96, 192, 336, 720}`" and one reporting
ETTh1 MSE at `h = 96` alone are not comparable, and averaging them
to a single "ETTh1 MSE" number is silently wrong.

Context length is more subtle because papers rarely print it next
to the number. A Moirai at `C = 512` and a Moirai at `C = 4000` are
effectively different models and produce visibly different LTSF MSE.
See [protocols.md](protocols.md) section 4.

### 4. Same normalization

RevIN on vs RevIN off can change LTSF MSE by more than 10% relative.
Chronos-2's robust scaling, MOIRAI's per-context standardization,
MOMENT's RevIN, Sundial's per-patch normalization, and Chronos's
mean scaling are *not* the same preprocessor. The metric numbers
are reported *after* normalization, so a number computed on raw
series and a number computed on z-scored series are in different
units even if both are labelled "MSE".

See [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md)
section 4.

### 5. Same aggregation

- Arithmetic mean vs geometric mean across datasets. Geometric mean
  is robust to outliers; arithmetic is not. Chronos-2 uses GM skill
  score; Sundial uses GM-relative; Moirai Table 6 uses arithmetic.
- Per-dataset weighting vs per-series weighting. A large dataset
  with 10k series is weighted either like a single datum or like
  10k independent observations, and the resulting "aggregate" can
  differ by several points.
- Mean rank (GIFT-Eval public leaderboard) vs skill score
  (fev-bench, Chronos-2) vs relative GM (Sundial). Same suite,
  different summary statistic; the conversions exist but are not
  identity. See
  [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md)
  section 2.

### 6. No overlap between pretraining data and eval data

The single biggest axis. A "zero-shot" claim is only meaningful if
the evaluation dataset is actually disjoint from the pretraining
corpus, and several well-known cases in the wiki violate this —
TimesFM and Chronos on Monash, Chronos-2 acknowledging partial
GIFT-Eval overlap, Moirai on LOTSA-overlapping datasets. Read the
paper's corpus declaration before comparing zero-shot claims. See
[../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md)
section 1 and [protocols.md](protocols.md) section 5.

### 7. Reported uncertainty

A 0.4-point lead without a confidence interval is not a lead. Check
whether either paper reports bootstrap CIs (Chronos-2 is the only
one in the wiki that does so consistently), a Friedman + Nemenyi
critical-difference diagram (Moirai-MoE, Mamba4Cast), or any other
statistical significance test. If neither number is accompanied by
uncertainty, assume the practical significance threshold is around
2 percentage points on a skill-score scale or around 5% relative on
a raw-metric scale; smaller gaps are noise.

## Reading the result

- **All seven pass.** The comparison is meaningful. You are
  probably reading two rows of the same table in the same paper,
  which is the only place this happens reliably.
- **One or two fail.** The comparison is interpretable with a
  stated caveat: "under the same split and horizon, ignoring
  normalization differences, Model A beats Model B by X."
- **Three or more fail.** The comparison is not meaningful.
  Presenting it as "A beats B" is misleading. Go to the original
  tables and restate the claim at the resolution where it actually
  holds — often "A beats B on suite S under protocol P with caveat
  C" — or do not state it at all.

Most cross-paper claims you see in preprints, blog posts and
Twitter threads about "the new SOTA TS-FM" fail at least three
items on this checklist. This is not a counsel of nihilism; it is a
counsel for reading [../benchmarks/leaderboard.md](../benchmarks/leaderboard.md)
and its footnotes with the protocol section open in a second tab.

## Related wiki pages

- [metrics.md](metrics.md) — the metric definitions behind item 1.
- [protocols.md](protocols.md) — splits, horizons, contexts, and
  zero-shot discipline behind items 2, 3 and 6.
- [seasonality-and-baselines.md](seasonality-and-baselines.md) —
  baseline choice and seasonal period behind item 5.
- [what-was-evaluated.md](what-was-evaluated.md) — the per-paper
  cross reference you need for items 1-6.
- [../benchmarks/methodology-caveats.md](../benchmarks/methodology-caveats.md)
  — a full discussion of the asterisks this checklist is designed to
  catch.
