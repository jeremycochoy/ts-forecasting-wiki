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
  visible text still said "README.md" (e.g.
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

## [2026-04-12] ingest | Moirai 2.0 (arXiv:2511.11698)

Salesforce's direct successor to [MOIRAI](papers/moirai.md).
Abandons the masked-encoder + multi-patch + mixture-of-Student-t
recipe of Moirai-1 for a plain decoder-only Transformer
(RMSNorm / GLU / RoPE / causal MHA) with a single patch size, a
9-quantile pinball head, and multi-token prediction for fast
inference. Pretrained on a new unnamed 36M-series / ~295B-observation
mixture combining non-leaking GIFT-Eval Pretrain, Chronos-Mixup,
KernelSynth, and ~2.15M-series Salesforce-internal CloudOps
telemetry (the last of which is proprietary, making the full
mixture non-reproducible despite four of five sources being open).
The 11.4M small is the *recommended* size because base (87.1M)
and large (305M) monotonically worsen on GIFT-Eval with training
data held fixed — a rare published negative-scaling result, which
the scaling-laws concept page now cites. Multivariate and covariate
support are dropped entirely versus Moirai-1, narrowing the task
surface. Evaluation is GIFT-Eval only (Table 1, MASE 0.728 /
CRPS 0.516 for small). Open weights at
`Salesforce/moirai-2.0-R-small`; cluster placement pivots from
Cluster 2 (Moirai-1) to Cluster 1 (decoder-only).

## [2026-04-12] ingest | TSPulse (arXiv:2505.13033, ICLR 2026)

IBM Granite team's analysis-side counterpart to [TTM](papers/ttm.md):
same 1-5M parameter TSMixer backbone, same Monash + LibCity
pretraining corpus, but the diagnostic-task destination instead of
forecasting. 1.06M parameters total, 8 TSMixer blocks with
softmax-gated attention, hidden `D = 24 = 3·pl`, patch 8,
context 512. Pretrained with a dual-space masked reconstruction
objective in time and frequency (FFT is computed over the
masked input so the mask propagates through `rfft`; no separate
frequency mask) producing three explicitly disentangled embedding
views — TimeE, FFTE, and register-token RegE — routed through
three different heads. Hybrid patch-level masking supports full
and partial patches, fixing the block-mask bias of MOMENT and
UniTS. Evaluates on TSB-AD anomaly, UEA-29 classification, LTSF
imputation and a custom UCR+synthetic similarity suite, reporting
wins over MOMENT, UniTS, VQShape, Chronos and TimesFM at 10-100x
fewer parameters. Explicitly *not* a forecaster — the Limitations
section defers forecasting to future work, and the
benchmarks/state-of-the-art page now notes TSPulse as a
non-forecaster in the multi-task cluster. Open weights at
`ibm-granite/granite-timeseries-tspulse-r1`; code in
`ibm-granite/granite-tsfm`. Cluster 6 primary, Cluster 5
secondary.

## [2026-04-12] ingest | SEMPO (arXiv:2510.19710, NeurIPS 2025)

Lightweight NeurIPS 2025 foundation model from BIT + SMU + Tongji
(mala-lab). 6.5M-parameter encoder-decoder transformer
(`S = 6, H = 16, D_p = 256, L_p = 64, L = 512`, RMSNorm + SwiGLU)
pretrained on a curated ~83M-point subset of UTSD in 10 hours on
4× A6000-48G — one of the most reproducible pretraining budgets
in the TS-FM literature. Two novel modules: **Energy-Aware
Spectral Decomposition** (FFT the input, partition into high- and
low-energy branches at a learnable threshold, apply independently
parameterized multi-band frequency masks, iFFT — preserves
low-energy-but-informative frequency components that Fourier-based
TS methods typically ignore) and **Mixture-of-Prompts** (128
learnable prompt experts, dense linear-softmax router with no
top-k, reshaped via a 2-layer MLP into `S × 2 × D_p` key-value
prefixes and concatenated into every self-attention layer —
routes a prefix, not an FFN, so it is explicitly distinct from
MoE). Two-stage training: Stage 1 reconstruction pretraining with
EASD masking, Stage 2 MoP tuning with the backbone frozen.
Headline: 23.1% MSE / 10.8% MAE reduction on TSLib zero-shot
(Table 1, 12 out of 14 column wins vs Time-MoE-B/L, Timer,
Moirai-S/B/L, Chronos-S/B/L, TimesFM, Moment) at 1.8x-100x fewer
parameters and 1.3x-3700x fewer pretraining points. Figure 6
reports ETTh1 inference at 22 s vs 205 s for Moirai-S and
14,185 s for Chronos-L. Positioned in Cluster 5 even though the
backbone is a transformer; the scaling-laws page now cites SEMPO
as the cleanest "less is more" counter-narrative to the
billion-parameter TS-FM trajectory.

## [2026-04-12] query-filed-back | HP transfer across scales concept page

User asked about alternatives to muP for transferring hyperparameters
across a model family (Tiny d=512 L=6 through Large d=1024 L=24) on
a consumer-GPU budget. Filed the answer back as
`wiki/concepts/hp-transfer-across-scales.md` covering five methods:
(1) muP with reference to Yang et al. arXiv:2203.03466; (2) poor
man's muP (the `lr ∝ 1/d` scaling rule derived from muP theory);
(3) progressive transfer from previous size; (4) cross-shaped proxy
search — training two cheap proxies at (same-depth, small-width) and
(same-width, small-depth) to bracket the target; (5) architectural
stabilizers (QK-Norm + Pre-RMSNorm + sigma-reparam) that reduce LR
sensitivity to scale and make all other transfer methods more
reliable. Includes a recommended workflow combining methods 1-5 for
a 4090-class single-GPU setup. Cross-linked from concepts/concepts.md,
concepts/scaling-laws.md, and wiki/index.md.

## [2026-04-12] query-filed-back | Data normalization concept page

User asked "how do people use the data as input? Do they differentiate?
Do they all use RevIN?" Filed back as `wiki/concepts/data-normalization.md`
covering the full normalization landscape: RevIN (the de facto standard),
mean-scaling (Chronos), partial-window instance norm (Moirai-2),
any-variate learned norm (MOIRAI), raw numerical text (LLMTime), z-score
non-reversed (LTSF baselines only), and differencing + cumsum (not used
by any TS-FM). Includes a detailed "why not differencing" section covering
error accumulation under cumsum, loss function mismatch, the argument
that RevIN already handles non-stationarity, patch tokenization awkwardness
after differencing, and multi-step probabilistic forecasting complications.
Summary comparison table mapping each method to its inverse quality, error
accumulation behavior, and which papers use it. Cross-linked from
concepts.md section hub, revin-normalization.md (which remains the
RevIN-specific deep dive), and wiki/index.md.

## [2026-04-12] refactor | Expand TimesFM synthetic recipe in synthetic-data-augmentation.md

The "Real + synthetic mixing (TimesFM)" entry was a one-liner. Expanded
to document the full 4-component ARMA recipe from Das et al. Appendix
A.8: piecewise linear trends (2-8 segments), ARMA(p,q) with 1<=p,q<=8,
sin and cos waves with random periods, randomly enabled/disabled and
summed with uniform weights, 50% multiplicative trend, 3M series x 2048
= 6.1B points. Added computational comparison to KernelSynth (O(n) vs
O(n^3)) and noted the spectral trade-off (fixed ARMA poles vs GP
multi-scale composition). Updated the "Papers that exemplify this"
bullet for TimesFM to match the expanded description.

## [2026-04-19] query-filed-back | Data-section expansion + Google Trends page

Expanded `datasets-benchmarks/` into a comprehensive "data section"
covering every named TS-FM dataset, their types, usage, known leakage
status, and scrub tooling. New pages created:

- `datasets-benchmarks/datasets-benchmarks.md` — rewritten as the
  data-section hub with summary tables for corpora and benchmarks.
- `datasets-benchmarks/dataset-types.md` — 8-axis taxonomy
  (pretraining vs eval, real vs synthetic, univariate vs
  multivariate, domain, frequency, license, release form, deprecation).
- `datasets-benchmarks/evaluation-benchmarks.md` — consolidated
  catalog of active vs historical vs retired evaluation suites.
- `datasets-benchmarks/leakage-map.md` — cross-reference matrix of
  pretraining corpora × evaluation benchmarks, verified against
  primary sources (GIFT-Eval Tables 13/14, Moirai-MoE Fig 3,
  Chronos §5.1, Chronos-2 Table 3).
- `datasets-benchmarks/scrub-tools.md` — catalog of public tooling:
  GiftEvalPretrain, `uni2ts`, Timer-S1 pipeline (documented but
  unreleased), pytrends, trendecon, G-TAB, reconstructable
  ~50-line scrub recipe.
- `datasets-benchmarks/google-trends-data.md` — every known way to
  obtain Google Trends time-series data, including the TimesFM
  corpus description, pytrends, BigQuery, G-TAB, trendecon, and
  open contribution opportunities.

Two new paper PDFs added to `papers/` as references for the Google
Trends page:

- `gtab_2007.13861.pdf` — Robert West, "Calibration of Google Trends
  Time Series", CIKM 2020. G-TAB method.
- `gtrends-proper_2104.03065.pdf` — Medeiros & Pires, "The Proper
  Use of Google Trends in Forecasting Models", arXiv:2104.03065.

Updated `index.md` with all new entries. This expansion was
submitted via a PR branch (`data-section-expansion`) and merged
after two self-review iterations.

## [2026-04-19] lint + audit | Leakage-claim verification against primary sources

Full audit of leakage and overlap claims across the wiki, verified
against the source PDFs in `papers/` and the GIFT-Eval paper
(arXiv:2410.10393) fetched from arxiv. Every claim is now traceable
to a specific table/figure. Notable corrections and enrichments:

1. **Chronos v1 `papers/chronos.md`**: rewrote the Benchmark II
   overlap claim. The Chronos paper's own footnote 5 states Benchmark
   II leakage risk is "minimal" — Benchmark II is not the leakage
   source. The load-bearing overlap is **Benchmark I** (in-domain)
   whose datasets (M4 Daily/Hourly/Monthly/Weekly, Electricity, ETT,
   KDD Cup 2018, Temperature-Rain, Pedestrian Counts, London Smart
   Meters) directly overlap GIFT-Eval test per GIFT-Eval Table 13.
   Also documented: Chronos pretraining-only set includes `Wiki Daily
   (100k)` = 100,001 Wikipedia articles daily (Chronos Table 3).

2. **TimesFM `papers/timesfm.md`**: replaced the fabricated
   "22k-query / ~68M-series" with the verified numbers from TimesFM
   Table 1 and §2: ~22k Google Trends head queries, "all Wikimedia
   pageviews" (~300B source points before mixing), 3M synthetic
   series (6.1B points), plus M4 / Electricity / Traffic / Weather /
   Favorita as explicit real-dataset additions.

3. **MOIRAI `papers/moirai.md`**: added that GIFT-Eval paper §F.2
   explicitly retrains MOIRAI on `GiftEvalPretrain` for a
   leakage-free comparison and refers to the original as
   `Moirai-Leakage`. Effect grows with prediction length.

4. **`datasets-benchmarks/gift-eval.md`**: enriched with the full
   23-test-dataset list (Table 13) grouped by domain, plus the
   GIFT-Eval Pretrain composition (88 datasets / 4.5M series / 230B
   obs). Made explicit that the Web/CloudOps test subset is zero
   Wikipedia — all entries are BizITObs + Bitbrains CloudOps.

5. **`datasets-benchmarks/time-300b.md`**: added the real domain
   breakdown (Nature 90.50% of observations) and the Web-domain
   dataset listing from Time-MoE Table 10, including the three
   Wikipedia-derived entries (Kaggle Web Traffic Weekly, Extended
   Web Traffic, Wiki-Rolling) plus Wiki Daily (100k).

6. **`datasets-benchmarks/lotsa.md`**: added the four Wikipedia-derived
   datasets in LOTSA with per-dataset series counts from MOIRAI
   Table 14. Clarified that the Wikipedia content in LOTSA does not
   produce GIFT-Eval test leakage by itself (test has no Wikipedia);
   the leakage vector is shared Monash / M4 / ETT / Electricity
   entries, which `GiftEvalPretrain` scrubs out.

7. **`evaluation/protocols.md`**: rewrote the "Known leakage cases"
   section with verbatim citations from Moirai-MoE Figure 3 and
   Table 2 captions, Chronos-2 §5.1, GIFT-Eval §F.2. Added the
   fev-bench per-model leakage percentages from Chronos-2 Table 3
   (Moirai-2.0 28% is the highest; Chronos-2 / Chronos-Bolt /
   COSMIC / TabPFN-TS at 0%).

8. **`benchmarks/rebuilding-timebench.md`**: corrected Chronos
   corpus size to 890K series / 84B observations (was "tens of B"),
   Time-300B Nature fraction to 90.50%, and the leakage discussion
   to reflect that Chronos v1's problem is Benchmark I, not
   Benchmark II. Added a "crucial finding" callout: raw Wikimedia
   pageviews do not leak against GIFT-Eval test.

9. **`benchmarks/wikipedia-pageviews-leakage.md`**: added a
   cross-corpus Wikipedia-content table showing that LOTSA,
   Time-300B, Chronos v1, and TimesFM all already include
   Wikipedia-derived data, so layering raw Wikimedia on top is
   primarily a deduplication concern.

10. **`benchmarks/training-a-small-model.md`**: replaced vague
    "train on LOTSA with Timer-S1 curation" recommendation with a
    direct pointer to `Salesforce/GiftEvalPretrain` as the simplest
    clean-starting-point for a GIFT-Eval-targeted run.

**Lint pass concurrent with the audit.** No broken relative links
(two apparent failures in `log.md` were inside backticks and render
as inline code). No orphan pages. All 23 paper leaves have the
required template sections. Concept pages
`data-normalization.md` and `hp-transfer-across-scales.md` were
under the ≥3-concept / ≥2-architecture cross-link requirement;
added the missing cross-links. Architecture pages all pass the
cross-link check. 24 paper leaves use `## In the knowledge graph`
instead of `## Related wiki pages` — this is template-correct for
paper leaves per CLAUDE.md and is not a lint violation.

## [2026-04-19] query-filed-back | Wikipedia-pageviews leakage audit against GIFT-Eval

Added `benchmarks/wikipedia-pageviews-leakage.md`. Verified against
the GIFT-Eval paper (arXiv:2410.10393) that the **test split contains
zero Wikipedia-derived series** — all Web/CloudOps test datasets are
CloudOps (BizITObs + Bitbrains), so folding raw Wikimedia data into a
pretraining corpus does not leak w.r.t. GIFT-Eval test. Enumerated
the three Wikipedia-derived datasets inside GIFT-Eval Pretrain
(Table 14: Wiki-Rolling 47,675 series / 40.6M obs via GluonTS;
Kaggle Web Traffic Weekly 145,063 series / 16.5M obs via Monash;
Extended Web Traffic 145,063 series / 370.9M obs via Monash) and
documented the dedup procedure for layering raw Wikimedia pageviews
on top of GiftEvalPretrain. Flagged that no dedicated scrub utility
is published — GiftEvalPretrain is the existing "tool" — and
provided the reconstructable ~50-line recipe. Cross-linked from
index.md and rebuilding-timebench.md.

## [2026-04-19] query-filed-back | Rebuilding-TimeBench practical guide

Added `benchmarks/rebuilding-timebench.md`, a step-by-step recipe for
assembling a TimeBench-equivalent ~1T-point pretraining corpus from
only publicly available data. Starting point is `GIFT-Eval Pretrain`
(230B obs, already scrubbed). Additions are Time-300B, the Chronos
public corpus, KernelSynth / TSMix / canonical signal synthetics, and
optionally the Time Series Pile. Documents the mandatory Timer-S1
curation pipeline (causal imputation, k-sigma/IQR, ADF filter,
GIFT-Eval leakage scrub, resampling, value-flipping) to apply on
every real-data layer. Provides a cumulative size-budget worksheet
and an explicit list of what a public rebuild cannot faithfully
reproduce (TimeBench weights, CloudOps component, Timer-S1 filter
thresholds). Cross-linked from index.md; intended as a companion to
`benchmarks/training-a-small-model.md`.

## [2026-04-19] refactor | Expand Timer-S1 tokenizer description

Enriched `papers/timer-s1.md` "Architecture at a glance" with the
patch-embedder details from the paper's §3.1 Eq. 3: each P=16 patch is
concatenated with a P-length binary padding mask, and the embedder is
a weight-tied residual network `R^{2P} → R^D` (32 → 1024), not a
linear projection. Added that no additive positional embedding is
used (relative position comes exclusively from RoPE inside attention,
Eq. 5-6) and that `N = ⌈T/P⌉` gives N=180 at pretraining context
T=2880 and N=720 after stage-2 RoPE context extension to T=11520.

## [2026-04-12] refactor | Add patch in/out sizes and forecast method to paper leaves

Added input patch size, output patch size, and forecasting strategy
(direct multi-step vs autoregressive rollout) to the "Architecture at
a glance" section of 17 paper leaves. Key findings documented:
Chronos-2 uses patch_in=16, patch_out=16, direct multi-step up to 64
output patches (1024 steps) in a single forward pass — same patch size
as Sundial (16), matching the GIFT-Eval #1 model. TimesFM uses
patch_in=32, patch_out=128 (4x asymmetric to reduce rollout steps).
Timer/Timer-XL use patch=96. Chronos v1 uses scalar tokens (patch=1).
Mamba4Cast does single-pass direct full-horizon with no patch
tokenization. TTM uses adaptive patch sizes with direct multi-step
output. 6 leaves skipped (info already present): moment, moirai,
moirai-moe, moirai-2, timer-s1, timegpt.
