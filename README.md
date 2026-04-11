# Time-Series Forecasting — Research Wiki

This repository organizes knowledge about time-series forecasting foundation
models as a navigable knowledge graph. It moves from general concepts (what
forecasting is, classical methods, the deep-learning era) down through
architecture families and cross-cutting technical ideas, all the way to the
individual research papers that sit as leaves of the graph. The goal is to
make it easy to locate a paper inside a broader intellectual context, to
compare approaches across clusters, and to understand how the field arrived
at the current wave of time-series foundation models (TS-FMs).

## Structure

- `papers/` — PDF leaves. One file per primary research paper, named by
  slug and arXiv id (for example `timesfm_2310.10688.pdf`,
  `chronos_2403.07815.pdf`). These are the empirical ground truth the wiki
  points at.
- `wiki/` — the markdown knowledge graph. Top-level folders are
  `foundations/`, `foundation-models/`, `architectures/`, `concepts/`,
  `datasets-benchmarks/`, and `papers/`. Every page links up to its parent
  section and across to related nodes so that a reader can walk the graph
  rather than hunt through a flat list.

## Start here

- [wiki/overview.md](wiki/overview.md) — the wiki entry point and map of the
  territory.
- [wiki/index.md](wiki/index.md) — flat catalog of every wiki page with
  a one-line gloss; the first stop when you know a topic but not where
  it lives.
- [wiki/log.md](wiki/log.md) — chronological append-only record of
  ingests, queries-filed-back, lint passes, refactors, and schema
  changes.
- [wiki/foundation-models/taxonomy.md](wiki/foundation-models/taxonomy.md) —
  the seven-cluster taxonomy that most other pages anchor into.
- [wiki/papers/papers.md](wiki/papers/papers.md) — index of the paper
  leaves, with one short page per paper.

## Section hubs

Every wiki section has a folder-note hub named after the section itself.
The full set, in reading order:

- [wiki/foundations/foundations.md](wiki/foundations/foundations.md) — classical and
  deep-learning precursors to TS-FMs.
- [wiki/foundation-models/foundation-models.md](wiki/foundation-models/foundation-models.md) —
  the TS-FM paradigm, its brief history, and the canonical seven-cluster
  taxonomy.
- [wiki/architectures/architectures.md](wiki/architectures/architectures.md) — the
  architecture-family pages (decoder-only, masked encoder, encoder-decoder,
  MoE, LLM reprogramming, lightweight, flow matching).
- [wiki/concepts/concepts.md](wiki/concepts/concepts.md) — cross-cutting
  technical ideas shared across papers.
- [wiki/datasets-benchmarks/datasets-benchmarks.md](wiki/datasets-benchmarks/datasets-benchmarks.md) —
  pretraining corpora and shared evaluation suites.
- [wiki/benchmarks/benchmarks.md](wiki/benchmarks/benchmarks.md) — head-to-head
  performance tables and practical guides.
- [wiki/evaluation/evaluation.md](wiki/evaluation/evaluation.md) — metrics,
  protocols, baselines, and the per-paper methodology summary.
- [wiki/research/research.md](wiki/research/research.md) — the research
  corner: reading roadmap, open problems, comparison matrix,
  reproducibility table, contributing guide, glossary.
- [wiki/papers/papers.md](wiki/papers/papers.md) — the 20 paper leaves.

## Scope (as of 2026-04)

Version 0 of the wiki covers 20 foundation-model papers published between
2023 and 2025, spanning decoder-only autoregressive models, masked and
encoder-decoder models, mixture-of-experts variants, LLM-reprogramming
approaches, lightweight non-transformer models, multi-task universal
models, and the first continuous / flow-matching entrants. Additions and
corrections are welcome via pull request; please follow the existing page
layout and link conventions.

## Credits and license

PDFs in `papers/` are reproduced under their original licenses from
arXiv; copyright remains with the respective authors. The wiki text
(everything under `wiki/`, and this README) is released under
CC-BY-4.0.

## Related wiki pages

- [wiki/overview.md](wiki/overview.md)
- [wiki/foundation-models/foundation-models.md](wiki/foundation-models/foundation-models.md)
