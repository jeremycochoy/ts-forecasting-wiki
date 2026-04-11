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
- **Root pointers.** `wiki/overview.md` (formerly `wiki/README.md`)
  and the repo-root `README.md` were updated to point at `index.md`
  and `log.md`.

Added `wiki/index.md` and `wiki/log.md`, plus the above fixes; see
the commit for the full diff.

## [2026-04-12] refactor | Rename section READMEs to folder-note pattern

User feedback: "There is a lot of README.md and it's confusing. There
should be only one README at the root." The wiki had 11 README files
(repo root + 10 section-level hubs). The Obsidian graph view showed
ten nodes all labeled "README", which is useless for navigation.

Applied the folder-note pattern: each wiki section's hub is now named
after the section itself (`architectures/architectures.md`,
`concepts/concepts.md`, `papers/papers.md`, etc.), and the former
`wiki/README.md` is now `wiki/overview.md`. Repo-root `README.md` is
the only file named README in the whole repository.

Renames (git mv, so history is preserved):

- `wiki/README.md` → `wiki/overview.md`
- `wiki/foundations/README.md` → `wiki/foundations/foundations.md`
- `wiki/foundation-models/README.md` → `wiki/foundation-models/foundation-models.md`
- `wiki/architectures/README.md` → `wiki/architectures/architectures.md`
- `wiki/concepts/README.md` → `wiki/concepts/concepts.md`
- `wiki/datasets-benchmarks/README.md` → `wiki/datasets-benchmarks/datasets-benchmarks.md`
- `wiki/benchmarks/README.md` → `wiki/benchmarks/benchmarks.md`
- `wiki/evaluation/README.md` → `wiki/evaluation/evaluation.md`
- `wiki/research/README.md` → `wiki/research/research.md`
- `wiki/papers/README.md` → `wiki/papers/papers.md`

Two Python scripts handled the mechanical rewrite:

- `/tmp/rewrite_readme_links.py` — resolved every relative markdown
  link in every `.md` file, found the ones pointing at the old
  section READMEs, and replaced them with the correct new relative
  path. 61 link targets rewritten across 18 files.
- `/tmp/fix_readme_labels.py` — second pass on link labels whose
  visible text still said "README.md" (e.g.,
  `[../architectures/README.md](../architectures/architectures.md)`
  became `[../architectures/architectures.md](../architectures/architectures.md)`).
  48 labels rewritten across 18 files.

`CLAUDE.md` updated to document the folder-note convention in the
top-level layout block and in the Page Types section (page type 8
was previously "Section README", now "Section hub"). All step-by-step
workflow references to `wiki/papers/README.md` updated to
`wiki/papers/papers.md`. Prose references in this log file updated.

Total: 10 renames, 109 link rewrites, 59 files scanned.

## [2026-04-12] lint | Full health-check pass

Full schema-driven lint pass under `CLAUDE.md`'s Lint workflow. Scanned
80 `.md` files (repo root + `wiki/`, excluding `papers/`, `.git/`,
`.obsidian/`). Built the inbound-link graph, walked every relative
markdown link, ran the paper-leaf template check, verified cross-link
minimums, re-ran the auto-linker, and spot-checked numeric claims,
concept gaps, and known contradictions.

**Counts.** Files scanned: 80. Broken relative links (hard): 0 fixed,
0 flagged. Broken anchor fragments (soft): 28 fixed — all of them
cluster-heading anchors in `foundation-models/taxonomy.md` that were
missing the extra dash GitHub inserts where the em-dash `—` is
stripped (the H2 `## Cluster 1 — Decoder-only ...` renders as
`#cluster-1--decoder-only-...`, double dash). Rewrote via
`/tmp/fix_cluster_anchors.py` across 21 files. Orphans: 0 in the
baseline state; `wiki/research/training-recipes.md` and
`wiki/research/failure-modes.md` appeared mid-pass from the sibling
"best-wiki-possible" agent and are flagged as expected transient
orphans for that agent to wire up into `research/research.md` and
`index.md`. Paper-leaf
template compliance: 20/20 leaves have all ten required H2 sections.
Cross-link minimums: 0 violations for concepts (≥3 concepts + ≥2
architectures) and 0 for architectures (≥3 concepts + ≥2 sibling
architectures). Every non-hub page has a `## Related wiki pages` or
`## In the knowledge graph` tail section.

**Auto-linker.** Re-ran the canonical auto-linker (`/tmp/autolink_lint.py`)
over the 20 model slugs, 6 dataset slugs, and 3 metric anchors, with
the longest-first rule (Chronos-2 before Chronos, Timer-XL/Timer-S1
before Timer, Moirai-MoE before Moirai) and the first-unlinked-occurrence-
per-file policy. Added 8 links across 6 files:
`benchmarks/benchmarks.md` (+2), `foundation-models/taxonomy.md` (+2),
`benchmarks/univariate-benchmarking.md` (+1), `datasets-benchmarks/lotsa.md` (+1),
`research/reading-roadmap.md` (+1), `research/reproducibility.md` (+1).
That is well under the ~30-link "significant new prose" threshold, so
the wiki was already well-linked before this pass.

**README.** Repo-root `README.md` was extended with a new "Section hubs"
block linking each wiki section's folder-note hub
(`foundations/foundations.md`, `architectures/architectures.md`,
`concepts/concepts.md`, `datasets-benchmarks/datasets-benchmarks.md`,
`benchmarks/benchmarks.md`, `evaluation/evaluation.md`,
`research/research.md`, `foundation-models/foundation-models.md`,
`papers/papers.md`). Before this pass, seven of the nine section hubs
had no inbound link from the repo-root README, which is a regression
against the schema's "every hub is reachable from overview / index /
README" rule.

**Contradictions flagged (not fully fixed).** One real mismatch caught
and patched in place: `benchmarks/efficiency-and-cost.md` listed
"Chronos-Tiny 8M" rows sourced from TTM Table 3, but the Chronos paper
itself only releases Mini (20M) / Small (46M) / Base (200M) / Large
(710M) — there is no "Chronos-Tiny" in the official family. The rows
were relabeled "Chronos-Tiny (TTM label)" with a footnote pointing at
`model-sizing-cheatsheet.md` as the source of truth, so the TTM-sourced
numbers are preserved without the implication that a Chronos-Tiny
checkpoint exists. Two softer mismatches were flagged but not fixed:
(1) the Chronos-2 / MOMENT / Timer-XL paper leaves list only their
primary cluster, while `foundation-models/taxonomy.md` gives them a
secondary cluster-6 (multi-task) tag — worth reconciling; (2) the
phrase "biggest of the smallest" means different things in
`benchmarks/training-a-small-model.md` (the base-tier ceiling: MOIRAI-Base
91M / MOMENT-Base 125M) versus the "Small ceiling" row in
`benchmarks/model-sizing-cheatsheet.md` (~40–50M, Chronos-Small /
MOMENT-Small). These describe different brackets rather than outright
disagreeing, but the shared vocabulary is confusing.

**Concept-gap and numeric-claim spot checks.** Term frequencies above
the 5-hits-in-3-files threshold: `ablation` (37 / 27), `pretraining
corpus` (39 / 21), `context length` (18 / 17), `horizon` (163 / 49),
`tokenization` (64 / 39), `fine-tuning` (35 / 21), `Student-t` (56 /
23), `RoPE` (12 / 7), `LayerNorm` (10 / 5). Of these, `horizon`,
`Student-t`, and `fine-tuning` already have glossary entries in
`research/glossary.md`; the rest are plausible candidates for a
dedicated concept page but nothing was created in this pass. No
unattributed numeric claims jumped out in a ~30-sample spot check of
`benchmarks/efficiency-and-cost.md`, `benchmarks/state-of-the-art.md`,
`benchmarks/training-a-small-model.md` and `benchmarks/model-sizing-cheatsheet.md` —
every ms/batch, GB, and parameter count was followed by a `(source,
Table N / Section X)` citation.

**Papers directory check.** `papers/` has 20 PDFs, `wiki/papers/` has
20 leaves plus `papers.md`. 1:1 match, no stale PDFs, no orphan leaves.

`wiki/index.md` not updated — no file names or one-line glosses
changed in this pass.

## [2026-04-12] refactor | Best-wiki-possible enrichment pass

Intellectual enrichment pass: asked "what would a first-time reader
still be missing given only the 20 PDFs in papers/?" and implemented
the highest-leverage fixes. Created four new synthesis pages:
`wiki/research/training-recipes.md` (consolidated optimizer / LR /
batch / steps / hardware table with direct paper citations),
`wiki/research/failure-modes.md` (per-paper documented weaknesses
plus five cross-cutting failure patterns), `wiki/research/timeline.md`
(2023-2026 chronological walk), and
`wiki/benchmarks/decision-guide.md` ("which model should I pick?"
indexed by use-case axes). Enriched three paper leaves with disclosed
training-recipe numbers pulled from the PDFs: `chronos.md`
(AdamW, wd 0.01, lr 1e-3, 200K steps, 8xA100-40GB), `moirai.md`
(AdamW, wd 0.1, linear warmup 10K + cosine, Small 100K / Base+Large
1M steps), and `time-moe.md` (full Appendix B recipe including
128xA100-80G cluster and BF16 precision). Added 15 glossary entries
for terms that appear across the wiki but had no definition (STP,
fev-bench, group attention, SQL, skill score, pinball loss, RoPE,
QK-Norm, Huber, NLL, BF16, channel independence, exposure bias,
leakage, TiRex/TimesFM-2.5/Toto/Moirai-2.0 as "referenced but
unleafed"). Updated `overview.md`, `index.md` and `scaling-laws.md`
to cross-link the new pages. Flagged for a later pass: a dedicated
"prior-art map" page tracing architecture choices back to NLP/CV
origins, a shift-aware calibration benchmark, and leaf enrichments
for TimesFM / MOMENT / Sundial with their training recipes (same
shape as the Chronos / MOIRAI / Time-MoE edits done here).
