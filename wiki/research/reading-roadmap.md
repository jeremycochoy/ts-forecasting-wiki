# Reading Roadmap for TS Foundation Models

The TS-FM literature is small enough (roughly two dozen serious papers)
that a motivated researcher can cover it end-to-end in a few weeks.
The problem is that the papers landed in a short burst, reuse each
other's terminology loosely, and disagree on benchmarks. The order below
is designed so each paper answers a question raised by the previous one.

All links point at the leaf pages in [../papers/](../papers/papers.md);
follow them to read the per-paper summaries, then open the PDFs in the
top-level `papers/` directory.

## Beginner track — "what is a TS foundation model?"

Read these first. They fix vocabulary (patch, zero-shot, probabilistic,
context window) and present the two canonical recipes.

1. [../foundations/time-series-forecasting.md](../foundations/time-series-forecasting.md)
   — a ground-level definition of the forecasting task.
   *Why next:* without this the rest reads like NLP with extra steps.
2. [../foundation-models/foundation-models.md](../foundation-models/foundation-models.md)
   and [../foundation-models/taxonomy.md](../foundation-models/taxonomy.md).
   *Why next:* the eight-cluster map lets you place every later paper
   in context instead of reading them as independent systems.
3. [../papers/timesfm.md](../papers/timesfm.md) — Google's decoder-only
   TimesFM.
   *Why next:* it is the cleanest and simplest of the "NLP-style"
   recipes: patches in, patches out, next-patch prediction.
4. [../papers/chronos.md](../papers/chronos.md) — AWS Chronos with value
   quantization on an unmodified T5.
   *Why next:* contrast with TimesFM on tokenization (quantized vocab
   vs continuous patches) and objective (cross-entropy vs point loss).
5. [../papers/moirai.md](../papers/moirai.md) — Salesforce MOIRAI,
   masked encoder on the [LOTSA](../datasets-benchmarks/lotsa.md) corpus.
   *Why next:* introduces multi-patch-size projections, any-variate
   attention, and the Student-t mixture output head — three ideas you
   will see reused.
6. [../concepts/zero-shot-forecasting.md](../concepts/zero-shot-forecasting.md)
   and [../concepts/probabilistic-forecasting.md](../concepts/probabilistic-forecasting.md).
   *Why next:* by now you have seen both working; the concept pages
   crystallize what they actually claim.

## Intermediate track — "what are the architectural axes?"

These papers pick a single design dimension and push it. Read them as a
set of ablations against the canonical recipes above.

1. [../papers/moment.md](../papers/moment.md) — masked reconstruction
   with a T5-initialized encoder, the [Time Series Pile](../datasets-benchmarks/time-series-pile.md) corpus, and a
   multi-task head.
   *Why next:* shows that a single pretrained encoder can serve
   forecast, classification, imputation and anomaly jobs.
2. [../papers/timer.md](../papers/timer.md) then
   [../papers/timer-xl.md](../papers/timer-xl.md).
   *Why next:* Timer is the GPT-style decoder applied cleanly to TS;
   Timer-XL then makes it multivariate via flattened tokens and
   "TimeAttention", which is the first serious open attempt at genuinely
   multivariate long-context TS-FM.
3. [../papers/chronos-2.md](../papers/chronos-2.md) — a 120M
   patch-based encoder with group attention for [in-context learning](../concepts/in-context-learning.md)
   over related series.
   *Why next:* Chronos-2 directly argues that multivariate /
   covariate-informed forecasting, not scale, is the next bottleneck.
4. [../papers/ttm.md](../papers/ttm.md) — IBM Tiny Time Mixer, a
   1-5M-parameter MLP-Mixer that is deployable on CPU.
   *Why next:* sharp contrast with the billion-parameter models and a
   reminder that capacity is not always the right axis.
5. [../papers/time-moe.md](../papers/time-moe.md) — a 2.4B-parameter
   sparse MoE decoder-only model trained on [Time-300B](../datasets-benchmarks/time-300b.md).
   *Why next:* the clearest data point on "what happens if we just
   scale a TS-FM like an LLM".
6. [../papers/lag-llama.md](../papers/lag-llama.md).
   *Why next:* the first published empirical scaling laws for a TS-FM,
   to be read in the same sitting as the Time-MoE scale plot.
7. [../concepts/scaling-laws.md](../concepts/scaling-laws.md),
   [../concepts/patch-tokenization.md](../concepts/patch-tokenization.md),
   [../concepts/value-quantization.md](../concepts/value-quantization.md).
8. [../evaluation/metrics.md](../evaluation/metrics.md) and
   [../evaluation/protocols.md](../evaluation/protocols.md) —
   **required reading on the intermediate track.** Without the metric
   definitions (MASE, CRPS, WQL, MSE-on-normalized) and the
   zero-shot / rolling-origin / context-length protocol conventions,
   every table in the papers above collapses into undifferentiated
   numbers. Read these before the advanced track so that the
   contrarian claims there (Moirai-MoE vs Chronos on [Monash](../datasets-benchmarks/monash-archive.md), Sundial
   vs Chronos-2 on [GIFT-Eval](../datasets-benchmarks/gift-eval.md)) are legible.
9. [../benchmarks/univariate-benchmarking.md](../benchmarks/univariate-benchmarking.md)
   — optional but recommended if you are also building a model of
   your own. Translates the metric and protocol reading above into a
   concrete "pick this suite, report these metrics, use this
   protocol label" recipe for a univariate-only model that needs to
   compare against 2024–2026 TS-FM SOTA.
10. [../benchmarks/training-a-small-model.md](../benchmarks/training-a-small-model.md)
    — the training-side companion to the previous bullet, for readers
    who are not just evaluating but pretraining. Recommends LOTSA as
    the default corpus, walks through the [Timer-S1](../papers/timer-s1.md) curation pipeline,
    and names the "biggest of the smallest" TS-FMs you have to beat
    to sit in the small-model bracket.

## Advanced track — "what are people arguing about?"

These papers either stake out a contrarian position or open a new
direction. Each one is best read with the counter-argument in mind.

1. [../papers/moirai-moe.md](../papers/moirai-moe.md) — unified
   projection plus token-level sparse MoE routing.
   *Why next:* argues that most of MOIRAI's multi-patch-size hack
   collapses into a gating problem.
2. [../papers/mamba4cast.md](../papers/mamba4cast.md) — Mamba / SSM
   backbone with PFN-style synthetic-only pretraining, single-pass
   horizon generation.
   *Why next:* two bets at once — synthetic-only data and non-transformer
   backbone. Both are open questions.
3. [../papers/sundial.md](../papers/sundial.md) — flow matching as the
   generative head on the [TimeBench](../datasets-benchmarks/timebench.md) ~1T-point corpus.
   *Why next:* the first flow-matching TS-FM to read, contrast with
   Chronos's categorical sampling and MOIRAI's Student-t mixture.
4. [../papers/units.md](../papers/units.md) and
   [../papers/totem.md](../papers/totem.md).
   *Why next:* both push "one model, many tasks" but from very
   different ends (task tokens vs VQ-VAE codebook).
5. [../papers/time-llm.md](../papers/time-llm.md),
   [../papers/gpt4ts.md](../papers/gpt4ts.md),
   [../papers/llmtime.md](../papers/llmtime.md).
   *Why next:* the "LLM reprogramming" cluster, read as a single debate
   about whether text LLMs have transferable temporal priors at all.
6. [../papers/lat-pfn.md](../papers/lat-pfn.md),
   [../papers/ts-jepa.md](../papers/ts-jepa.md),
   [../papers/mts-jepa.md](../papers/mts-jepa.md), then
   [../concepts/joint-embedding-predictive-architecture.md](../concepts/joint-embedding-predictive-architecture.md).
   *Why next:* the [Cluster 8](../foundation-models/taxonomy.md#cluster-8--jepa--latent-space-prediction)
   "predict in latent space, not in input space" branch. Read in
   order — LaT-PFN frames the JEPA + PFN combination, TS-JEPA gives
   the cleanest matched-capacity ablation against MAE / TS2Vec /
   autoregressive, MTS-JEPA shows the recipe extends to
   anomaly-prediction with multi-resolution targets and a
   collapse-preventing soft codebook. The concept page closes the
   loop by contrasting JEPA against masked reconstruction and
   contrastive learning.
7. [../papers/sempo.md](../papers/sempo.md) and
   [../papers/moirai-2.md](../papers/moirai-2.md) and
   [../papers/tspulse.md](../papers/tspulse.md).
   *Why next:* three "less is more" papers — SEMPO at 6.5M beats
   100–700M baselines, Moirai-2 documents *negative* scaling on
   GIFT-Eval with the 11.4M variant beating its 87M and 305M
   siblings, TSPulse at 1.06M wins TSB-AD against multi-task
   competitors. Together they make the case that the 2025–2026
   frontier is not raw parameter count.
8. [open-problems.md](open-problems.md) and
   [comparison-matrix.md](comparison-matrix.md).
   *Why next:* with every paper in head, go back and look at where the
   map still has holes — including [§10 latent-space pretraining vs
   forecast-loss pretraining](open-problems.md#10-latent-space-pretraining-vs-forecast-loss-pretraining), the cleanest
   open question for Cluster 8.

## Related wiki pages

- [open-problems.md](open-problems.md)
- [comparison-matrix.md](comparison-matrix.md)
- [../foundation-models/taxonomy.md](../foundation-models/taxonomy.md)
