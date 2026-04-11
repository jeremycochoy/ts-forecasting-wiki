# CLAUDE.md — Schema for the Time-Series Forecasting Foundation Models Wiki

This file is the working contract for a Claude Code (or equivalent) session
operating on this repository. Read it before touching anything. The generic
LLM-wiki pattern this schema instantiates is in
[llm-wiki.md](llm-wiki.md); that file describes the idea in the abstract.
This file is the project-specific conventions, templates, and workflows.

## Project identity

A research wiki on **time-series forecasting foundation models**. It tracks
the literature from roughly 2023 through the current month, organized as a
knowledge graph that goes from general (what is TS forecasting) down to
leaves (one page per paper). The reader is a researcher entering or
working in the field; the wiki exists to help them understand the material,
compare models by performance, pick a setup for their own work, and find
the right paper to cite.

## Architecture

Three layers. Each has a distinct lifetime and ownership discipline.

- **`papers/`** — raw source layer. Immutable PDFs downloaded from arXiv,
  named `<shortname>_<arxiv_id>.pdf`. Never modify. Never delete. New
  ingests go here first.
- **`wiki/`** — the wiki proper. LLM-owned markdown. Every claim is either
  sourced from a paper in `papers/` (cited) or follows from another wiki
  page (linked). This layer is where all the work happens.
- **`CLAUDE.md`** (this file) — schema / conventions / workflows.
  Co-evolves with the wiki; update it whenever a convention changes.

## Top-level layout

```
ts-forecasting-wiki/
├── CLAUDE.md                   (this file — schema)
├── llm-wiki.md                 (reference copy of the generic pattern)
├── README.md                   (repo landing page)
├── .gitignore
├── papers/                     (immutable PDFs)
│   └── <shortname>_<arxiv_id>.pdf
└── wiki/
    ├── README.md               (orientation and reading paths)
    ├── index.md                (flat catalog of every wiki page)
    ├── log.md                  (chronological append-only log)
    ├── foundations/            (classical TS, DL era — precursors)
    ├── foundation-models/      (TS-FM paradigm + taxonomy)
    ├── architectures/          (architecture families)
    ├── concepts/               (cross-cutting technical concepts)
    ├── datasets-benchmarks/    (pretraining corpora + eval suites as corpora)
    ├── benchmarks/             (head-to-head performance + practical guides)
    ├── evaluation/             (metrics, protocols, baselines)
    ├── research/               (open problems, roadmap, comparison matrix, glossary, contributing)
    └── papers/                 (leaf pages — one per paper)
```

## Page types and templates

Every page belongs to one of the types below. Use the template literally
when creating or rewriting. Do not invent new page types without updating
this file first.

### 1. Paper leaf — `wiki/papers/<slug>.md`

One per paper. Slug is the lowercase hyphen-separated model short name
(`timesfm`, `chronos-2`, `moirai-moe`). Target length 600–1000 words,
dense, no padding.

```markdown
# <Full Paper Title>

> **Short name:** `<slug>` · **arXiv:** [<id>](https://arxiv.org/abs/<id>) · **PDF:** [local](../../papers/<filename>.pdf) · **Date:** <YYYY-MM> · **Venue:** <venue or preprint>

**Authors:** <first 3> et al.

## Abstract
(one paragraph; verbatim or lightly paraphrased from the arXiv abstract)

## Key contributions
- bullet
- bullet

## Architecture at a glance
(2-4 sentences: backbone family, pretraining objective, parameter count,
training corpus)

## Why it matters
(2-3 sentences on what this paper moved forward)

## Strengths
- at least 3 bullets — concrete things the paper demonstrates

## Limitations and open critiques
- at least 3 bullets — what the paper admits, what follow-ups dispute,
  structural gaps (benchmark choice, scale evidence, reproducibility)

## Follow-up work and dialogue
(1-2 paragraphs on what papers build on, dispute, or supersede this one.
Cross-link paper leaves by slug.)

## Reproducibility
- **Open weights:** yes / no / partial, where (only if paper states it)
- **Code:** public repo if stated, else "referenced in paper but URL not extracted"
- **Training data:** fully public / partially public / proprietary
- **Compute to retrain:** quote any GPU-hours / FLOPs / step counts the
  paper gives; otherwise "not disclosed"
- **Deployment footprint:** params (d, L if disclosed), context length,
  CPU / small-GPU feasibility

## When to cite this paper
(2-3 sentences: what specific claim is this the canonical reference for,
so a reader knows when to pick this paper vs a successor)

## In the knowledge graph
- **Cluster:** [<name>](../foundation-models/taxonomy.md#<anchor>)
- **Architecture family:** [<page>](../architectures/<page>.md)
- **Related concepts:** [<page>](../concepts/<page>.md), ...
- **Dataset / corpus:** [<page>](../datasets-benchmarks/<page>.md) (if applicable)
- **See also:** [<paper>](./<other>.md)
```

### 2. Concept page — `wiki/concepts/<slug>.md`

A cross-cutting technical idea that multiple papers instantiate differently
(patch tokenization, value quantization, probabilistic forecasting, scaling
laws, ...). Target length 700–1200 words.

```markdown
# <Concept Name>

<1-2 sentence definition understandable to a reader with general ML background>

## Intuition
(one paragraph — what the idea is, why it exists, what it replaces)

## Mechanics
(concrete algorithm / formula / tensor shapes / short pseudocode;
show the input → output transformation)

## Why it works
(inductive bias, invariance, statistical property, or optimization
advantage; tie to general ML principles)

## Trade-offs and failure modes
(where it breaks, what it gives up, ablation evidence)

## Design choices in the literature
(how specific papers instantiated this differently)

## Open questions
- 3-5 real frontier research questions (not filler)

## Papers that exemplify this
- [<paper>](../papers/<slug>.md) — 1-line note on *how* it uses this

## Related wiki pages
- [<page>](../concepts/<other>.md)
- [<page>](../architectures/<other>.md)
```

Must cross-link to ≥3 other concepts and ≥2 architecture pages.

### 3. Architecture page — `wiki/architectures/<slug>.md`

Describes an architecture family (`decoder-only-autoregressive`,
`masked-encoder`, `encoder-decoder-t5`, `mixture-of-experts`,
`llm-reprogramming`, `lightweight-non-transformer`,
`flow-matching-continuous`). Same template as a concept page.

Must cross-link to ≥3 concept pages and ≥2 sibling architecture pages
(for comparison).

### 4. Dataset / corpus page — `wiki/datasets-benchmarks/<slug>.md`

A pretraining corpus (LOTSA, Time Series Pile, Time-300B, TimeBench) or
eval suite treated as a corpus (Monash Archive, GIFT-Eval).

```markdown
# <Dataset Name>

<1-2 sentence description>

## Overview
(paragraphs on origin, motivation, size, composition, how to obtain)

## Key ideas / variants
- bullet

## Papers that use this
- [<paper>](../papers/<slug>.md) — ...

## Related wiki pages
- ...
```

### 5. Benchmark analysis page — `wiki/benchmarks/*.md`

Head-to-head performance tables or practical comparison pages. Strict
rule: **every numeric claim cites a specific table or figure in a
specific paper.** No orphan numbers. Cross-paper tables must include a
"Source" or "Notes" column that flags comparability.

Pages in this section currently include: `leaderboard.md`,
`state-of-the-art.md`, `methodology-caveats.md`, `efficiency-and-cost.md`,
`univariate-benchmarking.md`, `training-a-small-model.md`,
`model-sizing-cheatsheet.md`.

### 6. Evaluation methodology page — `wiki/evaluation/*.md`

How to measure — metrics, protocols, baselines, statistical significance.
Formulas in backticks or `$$...$$` fenced blocks. Cross-reference
`wiki/benchmarks/` for actual numbers and paper leaves for "who reports what."

Pages currently: `metrics.md`, `seasonality-and-baselines.md`,
`probabilistic-evaluation.md`, `protocols.md`, `what-was-evaluated.md`,
`comparability-checklist.md`.

### 7. Research / frontier page — `wiki/research/*.md`

Open problems, reading roadmap, comparison matrix, reproducibility,
contributing guide, glossary. Research tone. Questions must be concrete
enough that a contributor could act on them.

Pages currently: `reading-roadmap.md`, `open-problems.md`,
`comparison-matrix.md`, `reproducibility.md`, `contributing.md`,
`glossary.md`.

### 8. Section README — `wiki/<section>/README.md`

One-paragraph orientation + bullet list with one-line gloss per sub-page.
Target 120–180 words. No content beyond the index.

### 9. Wiki root files

- `wiki/README.md` — entry point, orientation, "start here" reading paths,
  knowledge-graph sketch.
- `wiki/index.md` — flat catalog of every wiki page, one line each.
  First stop at query time. Must be kept in sync with every add / rename /
  delete.
- `wiki/log.md` — chronological append-only record of ingests, queries
  filed back, lint passes, and refactors.

## Style rules

- **No emojis.** Anywhere.
- **No fabricated numbers.** Every numeric claim cites a specific paper's
  table or figure, or a public source with an access date. Where a value
  is not disclosed, write "—" and say so in a caveat.
- **No marketing language.** No "revolutionary", "game-changing",
  "state-of-the-art" without citation.
- **Relative links for internal navigation.** Never absolute URLs to
  other wiki pages.
- **First occurrence of a model name in prose is linked to its leaf**
  (`[Chronos](../papers/chronos.md)`). Subsequent occurrences are plain
  text. An auto-linker enforces this for ingested text; new edits must
  respect it.
- **Every page ends with a "Related wiki pages" section** with ≥2
  cross-links.
- **Math in backticks** for inline (`y_t = f(x_{t-k:t})`) or `$$...$$`
  fenced blocks for display (GitHub renders both).
- **Fenced code blocks** for pseudocode and tensor-shape walkthroughs.
- **Research tone.** Direct, qualified, willing to write "disputed",
  "not comparable", "not disclosed".
- **ATX headings** (`##`), not setext. Title case for H1, sentence case
  for H2+ by default.
- **American English spelling**, Oxford commas OK.

## Cross-link discipline

- Concept → concept: ≥3
- Concept → architecture: ≥2
- Architecture → concept: ≥3
- Architecture → sibling architecture: ≥2
- Paper leaf → complete "In the knowledge graph" block (cluster,
  architecture family, concepts, corpus, see-also)
- Broken relative links are a lint failure. Fix on sight.
- When you rename or delete a page, grep for inbound links and update
  them in the same commit.

## Workflows

### Ingest: adding a new paper

1. Download PDF to `papers/<shortname>_<arxiv_id>.pdf` (descriptive
   User-Agent, polite spacing between arXiv requests).
2. Read the PDF targeted sections: abstract, method, architecture
   table (often appendix), benchmark tables, limitations / discussion.
   Extract title, authors, date, `(d, L, heads, d_ff, patch, context,
   params)` if disclosed, contributions, strengths, limitations,
   reproducibility facts.
3. Create leaf at `wiki/papers/<slug>.md` using the paper-leaf template.
4. Update `wiki/papers/README.md` index table AND cluster grouping.
5. Update `wiki/foundation-models/taxonomy.md` summary table AND cluster
   bullet list.
6. Update `wiki/research/comparison-matrix.md` with a new row.
7. Update `wiki/research/reproducibility.md` with a new row.
8. Update `wiki/evaluation/what-was-evaluated.md` with a new row.
9. If the paper introduces a new corpus, create
   `wiki/datasets-benchmarks/<slug>.md` using the corpus template, and
   update any page that already references "the corpus named X."
10. Update the relevant concept and architecture pages to mention the
    new paper in their "Papers that exemplify this" sections.
11. Update `wiki/benchmarks/leaderboard.md` and
    `wiki/benchmarks/state-of-the-art.md` if the paper's numbers are
    comparable to anything already tabulated.
12. Update `wiki/benchmarks/efficiency-and-cost.md` with the param count
    and any inference footprint the paper discloses.
13. Update `wiki/benchmarks/model-sizing-cheatsheet.md` if the paper
    discloses `(d, L)` and the bracket is already in the table.
14. Update `wiki/index.md` with the new leaf entry.
15. Append to `wiki/log.md`:
    `## [YYYY-MM-DD] ingest | <model name> (arXiv:<id>)` + 2-3 sentences.
16. Commit with a message naming the paper, the slug, and the touched
    sections.

### Query: answering a user question

1. First stop: `wiki/index.md`. Identify candidate pages.
2. Read those pages plus any referenced paper leaves.
3. Answer with citations (paper leaf, table or figure, or wiki page link).
4. **If the answer is non-trivial and generally useful**, file it back
   as a new wiki page under the appropriate section (`research/`,
   `benchmarks/`, `evaluation/`, `concepts/`). Update `wiki/index.md`.
   Append to `wiki/log.md`:
   `## [YYYY-MM-DD] query-filed-back | <topic>` + 1-2 sentences.
5. If the answer was trivial fact retrieval (one number, one definition),
   do NOT create a page. Chat-only.

### Lint: periodic health check

Run when asked, or after a batch of ingests. A full lint pass checks:

- **Orphan detection** — pages with zero inbound links from elsewhere in
  `wiki/`. Either link from a relevant parent page or delete.
- **Broken relative links** — every `../x.md` must resolve.
- **Template compliance** — paper leaves missing required sections,
  concept pages missing "Related wiki pages" or the `≥3 concepts + ≥2
  architectures` link minimum.
- **Unlinked model mentions** — re-run the auto-linker to catch new
  prose since the last pass.
- **Unattributed numbers** — grep for `%|M|B|params` in prose lines that
  don't contain `(...)` citations.
- **Concept gaps** — terms that appear ≥3 times across pages but have
  no dedicated concept or architecture page. Propose a new page.
- **Contradictions** — claims that disagree across pages. Flag in the
  lint report even if you can't resolve them.
- Append `## [YYYY-MM-DD] lint | <summary>` to `wiki/log.md` with
  counts and links to the fixes.

## Anti-patterns (do NOT do these)

- Do not fabricate numbers, URLs, model sizes, or hyperparameters.
- Do not add emojis.
- Do not create marketing-tone prose.
- Do not duplicate content across pages; cross-link instead.
- Do not write to `papers/` — the raw source layer is immutable.
- Do not use absolute URLs for internal wiki navigation.
- Do not leave a page without a "Related wiki pages" block.
- Do not edit `llm-wiki.md`; it is the reference copy of the generic
  pattern. Schema changes go in this file (`CLAUDE.md`).
- Do not commit auto-generated helper scripts (put them in `/tmp/`).
- Do not touch `.obsidian/` or `.DS_Store` (gitignored).
- Do not amend an already-pushed commit.
- Do not skip log entries for non-trivial changes. The log is how
  future sessions understand what happened.

## Canonical paper slugs (as of 2026-04-12)

20 paper leaves exist. Use these slugs in all cross-links:

```
timesfm       chronos        chronos-2     moment        moirai
moirai-moe    timer          timer-xl      timer-s1      lag-llama
timegpt       time-moe       time-llm      gpt4ts        llmtime
ttm           units          totem         sundial       mamba4cast
```

When ingesting a new paper, pick a slug by lowercasing the model short
name and hyphenating. Register the new slug in:

- `wiki/papers/README.md`
- `wiki/foundation-models/taxonomy.md`
- `wiki/index.md`
- the auto-linker mapping (see `wiki/log.md` for the most recent run)

## Canonical cluster taxonomy

The 7-cluster taxonomy lives in `wiki/foundation-models/taxonomy.md`.
H2 headings use these exact names so anchors are stable:

1. `## Cluster 1 — Decoder-only autoregressive TS-FMs`
2. `## Cluster 2 — Masked-encoder / encoder-decoder TS-FMs`
3. `## Cluster 3 — Mixture-of-experts TS-FMs`
4. `## Cluster 4 — LLM-adapted / reprogramming approaches`
5. `## Cluster 5 — Lightweight / non-transformer FMs`
6. `## Cluster 6 — Multi-task / universal unified TS models`
7. `## Cluster 7 — Continuous / flow-matching FMs`

A paper may belong to multiple clusters (primary + secondary). The
summary table in `taxonomy.md` is the source of truth.

## Getting started in a new session

1. Read this file.
2. Read `wiki/index.md` to see what exists.
3. Read `wiki/log.md` tail to see what was done recently.
4. Do the work.
5. Update `wiki/index.md` and `wiki/log.md` before committing.
