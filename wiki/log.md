# Change Log

Append-only chronological record of ingests, queries-filed-back, lint
passes, refactors, and schema changes. Each entry starts with
`## [YYYY-MM-DD] <op> | <title>` so the log is parseable:

    grep "^## \[" wiki/log.md | tail -10

Ops in use:

- `ingest` — a new paper PDF has been added to `papers/` and a new
  leaf plus cross-links have been created in `wiki/`.
- `query-filed-back` — a non-trivial answer to a user query has been
  turned into a new wiki page, or an existing page has been extended
  in response to a query.
- `lint` — a periodic health-check pass (orphans, broken links,
  template compliance, unlinked model mentions, unattributed numbers).
- `refactor` — a structural rewrite or enrichment of existing pages
  without adding new content.
- `schema` — `CLAUDE.md` itself, or any other schema-level document,
  has been rewritten.

## [2026-04-11] ingest | Initial seed — 19 TS foundation-model papers

Bootstrapped the wiki with the first batch of paper PDFs and 53
markdown pages. Seed set was the three canonical TS-FMs (TimesFM,
Chronos, MOMENT) plus the wave of 2023–2025 adjacent papers (MOIRAI,
Moirai-MoE, Timer, Timer-XL, Lag-Llama, TimeGPT-1, Time-MoE, Time-LLM,
GPT4TS, LLMTime, TTM, UniTS, TOTEM, Sundial, Mamba4Cast, Chronos-2),
for a total of 19 leaves. Created the top-level sections
`foundations/`, `foundation-models/`, `architectures/`, `concepts/`,
`datasets-benchmarks/`, and `papers/`, and the initial seven-cluster
taxonomy in `foundation-models/taxonomy.md`. Commit `003471c`.

## [2026-04-11] query-filed-back | Deepen wiki — benchmarks, research pages, critique, Timer-S1

Built out the `benchmarks/` section with the first versions of
`leaderboard.md`, `state-of-the-art.md`, `methodology-caveats.md`, and
`efficiency-and-cost.md`, plus the `research/` section with
`reading-roadmap.md`, `open-problems.md`, `comparison-matrix.md`,
`reproducibility.md`, `contributing.md`, and `glossary.md`. In the
same pass the Timer-S1 paper was ingested as the 20th leaf and slotted
into the decoder-only and mixture-of-experts clusters. Commit
`ec54409`.

## [2026-04-11] query-filed-back | Add evaluation section — metrics, protocols, per-paper summary

Created the `evaluation/` section to answer the methodology question
that the benchmarks pages assume but do not teach. Added
`metrics.md` as the flagship reference for point and probabilistic
metrics, plus `seasonality-and-baselines.md`,
`probabilistic-evaluation.md`, `protocols.md`,
`what-was-evaluated.md` (one row per paper), and
`comparability-checklist.md`. This is the layer that lets a reader
translate leaderboard cells into what is actually being measured.
Commit `eea950f`.

## [2026-04-11] lint | Auto-link model, dataset, and metric references

Ran the auto-linker over every wiki page to enforce the convention
that the first prose occurrence of a paper, dataset, or metric name is
a relative link to its canonical wiki page. Touched pages across
`benchmarks/`, `concepts/`, `datasets-benchmarks/`, `evaluation/`,
`research/`, and the paper leaves themselves. No new content, only
link additions. Commit `2b2ad05`.

## [2026-04-11] query-filed-back | Add univariate benchmarking recommendation page

In response to a user question about how to compare a univariate-only
forecaster against 2024–2026 TS-FM SOTA, created
`benchmarks/univariate-benchmarking.md`. The page recommends Chronos
Benchmark II plus the univariate subset of GIFT-Eval as the two
suites every recent TS-FM reports on, and walks through the caveats
(leakage, paper-relative zero-shot) that apply. Commit `28c87a3`.

## [2026-04-11] query-filed-back | Add training-a-small-model page

Filed back a training-side companion to the univariate-benchmarking
page. `benchmarks/training-a-small-model.md` answers the question
"where do I train, and how big does my model have to be to be
competitive?" It recommends LOTSA as the default open pretraining
corpus, names the smallest credible TS-FMs (TTM at 1–5M, Chronos-Tiny,
MOIRAI-Small) as the competitive ceiling for the small-model bracket,
and documents the 2026 data-curation recipe adapted from Timer-S1.
Commit `1114611`.

## [2026-04-11] query-filed-back | Add model sizing cheat sheet with (d, L) decoder-only configs

Filed back a sizing reference companion to
`training-a-small-model.md`. `benchmarks/model-sizing-cheatsheet.md`
gives a parameter-count bracket table with decoder-only
`(hidden_dim, num_layers)` combinations that land in each bracket,
plus the `12 * d^2 * L` approximation and a grid over
`d in {512, 768, 1024}`. The intent is that a reader can pick a
target parameter count and read off a concrete starting architecture
without having to grep back through half a dozen paper tables.
Commit `a8734f2`.

## [2026-04-12] refactor | Enrich cheat sheet and training page with actual (d, L) configs

Enriched `model-sizing-cheatsheet.md` and
`training-a-small-model.md` with the `(d, L)` configurations actually
used by released small TS-FMs — TTM, Chronos-Tiny, MOIRAI-Small, and
the Timer family — instead of only formula-based suggestions. No new
pages, just filling in the cells that were previously "—" with
disclosed numbers from the paper leaves. Commit `d9d7af8`.

## [2026-04-12] schema | Rewrite CLAUDE.md as project-specific schema

Replaced the generic `llm-wiki.md` pointer in `CLAUDE.md` with a
project-specific schema: three-layer architecture, eight page types
with templates, style rules, cross-link minimums, three workflows
(ingest / query / lint), anti-patterns, canonical 20-slug set, and
the seven-cluster taxonomy cross-reference. `llm-wiki.md` is
preserved as a reference copy of the generic pattern. Commit
`3f1e3cb`.

## [2026-04-12] lint | Reprocess wiki against new schema, create index.md and log.md

First full lint pass under the new `CLAUDE.md` schema. Created the
two schema-mandated infrastructure files: `wiki/index.md` (flat
catalog of every wiki page, grouped by section, papers in
chronological order) and `wiki/log.md` (this file). Audited every
`.md` under `wiki/` against the schema:

- **Broken relative links.** None found.
- **Orphan pages.** None found after building the inbound-link graph;
  every non-root page has at least one inbound link.
- **Emojis.** None found.
- **Paper-leaf template compliance.** 20/20 leaves have all required
  sections (Abstract through In the knowledge graph).
- **Cross-link minimums.** All 9 concept pages and all 7 architecture
  pages meet the schema's `>=3 concepts + >=2 architectures` and
  `>=3 concepts + >=2 sibling architectures` minimums.
- **Taxonomy consistency.** Timer-S1 (the late 20th slug) was missing
  from `research/comparison-matrix.md` and `research/reproducibility.md`;
  rows were added from the leaf facts. All other master-list pages
  already referenced all 20 slugs.
- **Root pointers.** `wiki/README.md` and the repo-root `README.md`
  were updated to point at `index.md` and `log.md`.

Added `wiki/index.md` and `wiki/log.md`, plus the above fixes; see
the commit for the full diff.
