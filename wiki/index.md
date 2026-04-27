# Wiki Index

One-line catalog of every page in the wiki. Updated on every ingest,
query-filed-back, and lint pass. Read this before drilling into the
knowledge graph.

Last updated: 2026-04-27.

(Enrichment pass on 2026-04-12 added four new pages:
[benchmarks/decision-guide.md](benchmarks/decision-guide.md),
[research/training-recipes.md](research/training-recipes.md),
[research/failure-modes.md](research/failure-modes.md), and
[research/timeline.md](research/timeline.md).)

## Orientation

- [overview.md](overview.md) — entry point, reading paths, knowledge-graph sketch
- [index.md](index.md) — this page, the flat catalog of every wiki page
- [log.md](log.md) — chronological append-only change log

## Foundations

- [foundations/foundations.md](foundations/foundations.md) — orientation to the classical and deep-learning precursors
- [foundations/classical-methods.md](foundations/classical-methods.md) — ARIMA, ETS, Theta, Prophet, state-space, TBATS and their role as baselines
- [foundations/deep-learning-era.md](foundations/deep-learning-era.md) — DeepAR, N-BEATS, PatchTST, TSMixer, the pre-FM transformer wave
- [foundations/time-series-forecasting.md](foundations/time-series-forecasting.md) — the forecasting task, axes of variation, canonical loss functions

## Foundation models

- [foundation-models/foundation-models.md](foundation-models/foundation-models.md) — definition of the TS-FM paradigm, motivation, brief history
- [foundation-models/taxonomy.md](foundation-models/taxonomy.md) — the canonical eight-cluster taxonomy with the 26-paper TS-FM summary table

## Architectures

- [architectures/architectures.md](architectures/architectures.md) — index of the architecture-family pages
- [architectures/decoder-only-autoregressive.md](architectures/decoder-only-autoregressive.md) — GPT-style next-patch transformers (TimesFM, Timer, Lag-Llama, Time-MoE)
- [architectures/encoder-decoder-t5.md](architectures/encoder-decoder-t5.md) — T5-style seq2seq on tokenized values (Chronos, Chronos-2)
- [architectures/flow-matching-continuous.md](architectures/flow-matching-continuous.md) — continuous flow-matching objective without vocabularies (Sundial)
- [architectures/lightweight-non-transformer.md](architectures/lightweight-non-transformer.md) — MLP-Mixer and SSM alternatives (TTM, Mamba4Cast)
- [architectures/llm-reprogramming.md](architectures/llm-reprogramming.md) — frozen or aligned LLM backbones (Time-LLM, GPT4TS, LLMTime)
- [architectures/masked-encoder.md](architectures/masked-encoder.md) — BERT-style masked-patch reconstruction (MOMENT, MOIRAI)
- [architectures/mixture-of-experts.md](architectures/mixture-of-experts.md) — sparse top-k routing for billion-scale TS-FMs (Time-MoE, Moirai-MoE, Timer-S1)
- [architectures/jepa-latent-prediction.md](architectures/jepa-latent-prediction.md) — predict the latent embedding of a masked window under an EMA target encoder (LaT-PFN, TS-JEPA, MTS-JEPA)

## Concepts

- [concepts/concepts.md](concepts/concepts.md) — index of the cross-cutting concept pages
- [concepts/in-context-learning.md](concepts/in-context-learning.md) — conditioning on related series or demonstrations at inference time
- [concepts/multi-task-universal.md](concepts/multi-task-universal.md) — one pretrained model for forecast, classify, impute, anomaly
- [concepts/patch-tokenization.md](concepts/patch-tokenization.md) — grouping contiguous values into transformer tokens
- [concepts/probabilistic-forecasting.md](concepts/probabilistic-forecasting.md) — predictive distributions and quantile heads vs point estimates
- [concepts/data-normalization.md](concepts/data-normalization.md) — the full normalization landscape: RevIN, mean-scaling, partial-window, raw values, and why nobody differences
- [concepts/revin-normalization.md](concepts/revin-normalization.md) — deep dive on RevIN mechanics and failure modes
- [concepts/scaling-laws.md](concepts/scaling-laws.md) — loss-vs-compute/data/params curves applied to TS-FMs
- [concepts/synthetic-data-augmentation.md](concepts/synthetic-data-augmentation.md) — KernelSynth, TSMix, PFN priors, and friends
- [concepts/value-quantization.md](concepts/value-quantization.md) — mapping real values to discrete bins or VQ codes
- [concepts/hp-transfer-across-scales.md](concepts/hp-transfer-across-scales.md) — muP, poor man's muP, cross-shaped proxy search, and architectural stabilizers for scaling HPs
- [concepts/zero-shot-forecasting.md](concepts/zero-shot-forecasting.md) — inference on unseen series with no gradient updates
- [concepts/contrastive-representation-learning.md](concepts/contrastive-representation-learning.md) — InfoNCE, hierarchical, and triplet-based pretraining (CPC, TS2Vec, TNC, T-Loss); the representation-learning track parallel to forecasting TS-FMs
- [concepts/joint-embedding-predictive-architecture.md](concepts/joint-embedding-predictive-architecture.md) — JEPA: latent-space prediction with EMA target encoder; non-reconstructive self-supervision that complements contrastive learning

## Datasets and corpora (the "data" section)

- [datasets-benchmarks/datasets-benchmarks.md](datasets-benchmarks/datasets-benchmarks.md) — data-section hub: summary tables of pretraining corpora, evaluation benchmarks, and cross-cutting pages
- [datasets-benchmarks/dataset-types.md](datasets-benchmarks/dataset-types.md) — 8-axis taxonomy of TS datasets (pretraining vs evaluation, real vs synthetic, univariate vs multivariate, domain, frequency, license, release form, deprecation status)
- [datasets-benchmarks/evaluation-benchmarks.md](datasets-benchmarks/evaluation-benchmarks.md) — consolidated catalog of evaluation suites in current use vs historical vs retired
- [datasets-benchmarks/leakage-map.md](datasets-benchmarks/leakage-map.md) — cross-reference matrix of pretraining corpora × evaluation benchmarks with verified leakage status
- [datasets-benchmarks/scrub-tools.md](datasets-benchmarks/scrub-tools.md) — catalog of public tooling for corpus curation and leakage removal
- [datasets-benchmarks/google-trends-data.md](datasets-benchmarks/google-trends-data.md) — every known way to obtain Google Trends time series, with the TS-FM angle
- [datasets-benchmarks/gift-eval.md](datasets-benchmarks/gift-eval.md) — 23-dataset zero-shot benchmark + `GiftEvalPretrain` pretraining subset
- [datasets-benchmarks/lotsa.md](datasets-benchmarks/lotsa.md) — MOIRAI's ~27B-observation open 9-domain pretraining corpus
- [datasets-benchmarks/monash-archive.md](datasets-benchmarks/monash-archive.md) — the canonical multi-domain forecasting benchmark archive
- [datasets-benchmarks/time-300b.md](datasets-benchmarks/time-300b.md) — Time-MoE's ~309B-point 9-domain pretraining corpus (Nature 90.5%)
- [datasets-benchmarks/time-series-pile.md](datasets-benchmarks/time-series-pile.md) — MOMENT's multi-task pretraining corpus (CMU Auton Lab)
- [datasets-benchmarks/timebench.md](datasets-benchmarks/timebench.md) — Sundial/Timer-S1's ~1T-point curated and augmented corpus

## Benchmarks (head-to-head + practical guides)

- [benchmarks/benchmarks.md](benchmarks/benchmarks.md) — index and reading order for the performance-layer pages
- [benchmarks/efficiency-and-cost.md](benchmarks/efficiency-and-cost.md) — params, training compute, inference latency, CPU deployability
- [benchmarks/leaderboard.md](benchmarks/leaderboard.md) — normalized tables on GIFT-Eval, Monash, Chronos Benchmark II, fev-bench, LTSF
- [benchmarks/methodology-caveats.md](benchmarks/methodology-caveats.md) — leakage, normalization, metric semantics, aggregation, asterisks
- [benchmarks/model-sizing-cheatsheet.md](benchmarks/model-sizing-cheatsheet.md) — `(d, L)` configs per parameter bracket for decoder-only TS-FMs
- [benchmarks/state-of-the-art.md](benchmarks/state-of-the-art.md) — which model wins where, partitioned by forecasting regime
- [benchmarks/training-a-small-model.md](benchmarks/training-a-small-model.md) — where to train, the smallest competitors, the 2026 curation recipe
- [benchmarks/univariate-benchmarking.md](benchmarks/univariate-benchmarking.md) — practical recommendation for univariate-only models vs TS-FM SOTA
- [benchmarks/decision-guide.md](benchmarks/decision-guide.md) — "which model should I pick?" indexed by use-case axes (probabilistic, multivariate, compute budget, horizon)
- [benchmarks/rebuilding-timebench.md](benchmarks/rebuilding-timebench.md) — practical recipe for assembling a TimeBench-style ~1T-point pretraining corpus from GIFT-Eval Pretrain + Time-300B + Chronos public corpus + synthetic generators, with the Timer-S1 leakage scrub
- [benchmarks/rebuilding-google-trends-corpus.md](benchmarks/rebuilding-google-trends-corpus.md) — end-to-end recipe for rebuilding a TimesFM-quality Google Trends corpus (extended through present-day): seed query selection, G-TAB calibration, Medeiros-style repeated sampling, temporal stitching, trendecon-style frequency reconciliation, audits, budget estimate
- [benchmarks/wikipedia-pageviews-leakage.md](benchmarks/wikipedia-pageviews-leakage.md) — concrete audit of Wikipedia-derived series in GIFT-Eval (test: none; Pretrain: Wiki-Rolling + Kaggle Web Traffic Weekly + Extended Web Traffic), with scrub procedure for adding raw Wikimedia on top

## Evaluation (methodology)

- [evaluation/evaluation.md](evaluation/evaluation.md) — index and reading order for metrics, protocols, and per-paper evaluations
- [evaluation/comparability-checklist.md](evaluation/comparability-checklist.md) — seven-item checklist for whether two papers' numbers can be compared
- [evaluation/metrics.md](evaluation/metrics.md) — every point and probabilistic metric used in TS-FM papers, with formulas and failure modes
- [evaluation/probabilistic-evaluation.md](evaluation/probabilistic-evaluation.md) — CRPS, WQL, quantile loss, calibration vs sharpness
- [evaluation/protocols.md](evaluation/protocols.md) — splits, rolling-origin, horizon and context conventions, what "zero-shot" means
- [evaluation/seasonality-and-baselines.md](evaluation/seasonality-and-baselines.md) — seasonal period `m`, Seasonal Naive, classical and deep baselines
- [evaluation/what-was-evaluated.md](evaluation/what-was-evaluated.md) — per-paper cross-reference of metrics, datasets, protocol, baselines (26 TS-FMs + JEPA cluster + TiDE)

## Research / frontier

- [research/research.md](research/research.md) — index and reading order for the research-corner pages
- [research/comparison-matrix.md](research/comparison-matrix.md) — dense side-by-side matrix of all paper design choices with derived takeaways
- [research/contributing.md](research/contributing.md) — how to add a paper or extend the wiki, plus research etiquette
- [research/glossary.md](research/glossary.md) — compact term definitions linking back into the wiki
- [research/open-problems.md](research/open-problems.md) — frontier research questions as of 2026-04 with concrete directions
- [research/reading-roadmap.md](research/reading-roadmap.md) — beginner / intermediate / advanced reading order through the papers
- [research/reproducibility.md](research/reproducibility.md) — per-paper open weights, code, data, cost, deployment footprint
- [research/training-recipes.md](research/training-recipes.md) — consolidated optimizer / LR / batch / steps / precision / hardware table for the 26 papers
- [research/failure-modes.md](research/failure-modes.md) — per-paper documented weaknesses and cross-cutting failure patterns
- [research/timeline.md](research/timeline.md) — chronological 2023-2026 walk through the 26 papers with each one's signature contribution

## Papers (leaves, 26 TS-FMs + 1 pre-FM precursor)

Papers are listed in chronological order by the date on the arXiv metadata, oldest first.

- [papers/papers.md](papers/papers.md) — paper-leaf index with cluster grouping and the full 26-paper metadata table (plus pre-FM precursors)
- [papers/gpt4ts.md](papers/gpt4ts.md) — frozen GPT-2/BERT/BEiT TS adapter (OFA/FPT), Alibaba, ~GPT-2 base (2023-02)
- [papers/time-llm.md](papers/time-llm.md) — frozen Llama reprogrammed via patch-to-prototype cross-attention (2023-10)
- [papers/timegpt.md](papers/timegpt.md) — Nixtla's closed encoder-decoder commercial API, conformal intervals, undisclosed size (2023-10)
- [papers/timesfm.md](papers/timesfm.md) — Google's decoder-only patched transformer, ~200M params (2023-10)
- [papers/lag-llama.md](papers/lag-llama.md) — Llama-style decoder with lag-feature tokens and Student-t head (2023-10)
- [papers/llmtime.md](papers/llmtime.md) — zero-shot GPT-3/Llama-2 forecasting via digit-level text tokenization (2023-10)
- [papers/ttm.md](papers/ttm.md) — IBM Tiny Time Mixers, TSMixer backbone, 1–5M params, CPU-deployable (2024-01)
- [papers/moirai.md](papers/moirai.md) — Salesforce masked-encoder with any-variate attention, 14M/91M/311M (2024-02)
- [papers/timer.md](papers/timer.md) — Tsinghua decoder-only with S3 format and UTSD corpus, ~100M class (2024-02)
- [papers/totem.md](papers/totem.md) — cross-domain VQ-VAE codebook plus specialist/generalist transformers (2024-02)
- [papers/units.md](papers/units.md) — Harvard/MIT unified transformer with task-token interface across 38 datasets (2024-02)
- [papers/chronos.md](papers/chronos.md) — AWS T5 encoder-decoder on quantized TS vocab, 20M–710M family (2024-03)
- [papers/time-moe.md](papers/time-moe.md) — first billion-parameter TS-FM, sparse MoE decoder on Time-300B (2024-09)
- [papers/mamba4cast.md](papers/mamba4cast.md) — synthetic-only Mamba-2 SSM trained PFN-style, single-pass horizon (2024-10)
- [papers/moirai-moe.md](papers/moirai-moe.md) — sparse-MoE decoder-only successor to MOIRAI on LOTSA (2024-10)
- [papers/moment.md](papers/moment.md) — CMU open masked-encoder family, 40M/125M/385M on the Time Series Pile (2024-10)
- [papers/timer-xl.md](papers/timer-xl.md) — long-context multivariate Timer with TimeAttention, 84M (2024-10)
- [papers/sundial.md](papers/sundial.md) — Tsinghua continuous flow-matching TS-FM on ~1T-point TimeBench (2025-02)
- [papers/chronos-2.md](papers/chronos-2.md) — AWS 120M encoder-only with group attention for ICL over related series (2025-10)
- [papers/sempo.md](papers/sempo.md) — 6.5M encoder-decoder transformer with energy-aware spectral decomposition and mixture-of-prompts, NeurIPS 2025 (2025-10)
- [papers/moirai-2.md](papers/moirai-2.md) — Salesforce decoder-only successor to MOIRAI with 9-quantile pinball head; 11.4M small beats its own 87M and 305M siblings on GIFT-Eval (2025-11)
- [papers/timer-s1.md](papers/timer-s1.md) — Tsinghua/ByteDance 8.3B sparse-MoE with Serial-Token Prediction, GIFT-Eval SOTA (2026-03)
- [papers/tspulse.md](papers/tspulse.md) — IBM Granite 1.06M TSMixer with dual-space time+FFT masked reconstruction for multi-task analysis, ICLR 2026 (2026-03)
- [papers/lat-pfn.md](papers/lat-pfn.md) — WAIR JEPA + PFN hybrid trained on synthetic data with normalized abstract time axis; first JEPA-for-TS (2024-05)
- [papers/ts-jepa.md](papers/ts-jepa.md) — KTH/NYU/Flatiron clean four-component JEPA-for-TS, 70% mask, L1 latent loss; first systematic JEPA-for-TS study (2025-09)
- [papers/mts-jepa.md](papers/mts-jepa.md) — Purdue/RPI/Stony Brook multi-resolution JEPA + soft codebook for multivariate anomaly *prediction*, top F1 + AUC on MSL/SMAP/SWaT/PSM (2026-02)

### Pre-FM precursors and baselines

- [papers/tide.md](papers/tide.md) — Google MLP encoder-decoder with covariate highway and linear residual, TMLR 2023; design ancestor of [TimesFM](papers/timesfm.md) and standard full-shot baseline (2023-04)
