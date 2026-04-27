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
    ├── overview.md             (orientation and reading paths)
    ├── index.md                (flat catalog of every wiki page)
    ├── log.md                  (chronological append-only log)
    ├── foundations/
    │   ├── foundations.md      (section hub)
    │   └── ... topic pages
    ├── foundation-models/
    │   ├── foundation-models.md (section hub)
    │   └── taxonomy.md
    ├── architectures/
    │   ├── architectures.md    (section hub)
    │   └── ... family pages
    ├── concepts/
    │   ├── concepts.md         (section hub)
    │   └── ... concept pages
    ├── datasets-benchmarks/
    │   ├── datasets-benchmarks.md (section hub)
    │   └── ... corpus pages
    ├── benchmarks/
    │   ├── benchmarks.md       (section hub)
    │   └── ... analysis + practical guides
    ├── evaluation/
    │   ├── evaluation.md       (section hub)
    │   └── ... methodology pages
    ├── research/
    │   ├── research.md         (section hub)
    │   └── ... frontier pages
    └── papers/
        ├── papers.md           (section hub / paper index)
        └── <slug>.md           (one per paper)
```

**Naming convention (folder-note pattern):** there is exactly ONE
`README.md` in the entire repo, at the root. Every wiki section uses
a folder-note named after the section itself (`architectures.md`,
`concepts.md`, etc.) as its hub. The wiki's own hub is `overview.md`.
This keeps the Obsidian graph view readable — every node has a
descriptive name, no collisions on the label "README".

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

### 8. Section hub (folder note) — `wiki/<section>/<section>.md`

Each wiki section has a hub page named after the section itself:
`foundations/foundations.md`, `architectures/architectures.md`,
`concepts/concepts.md`, `datasets-benchmarks/datasets-benchmarks.md`,
`benchmarks/benchmarks.md`, `evaluation/evaluation.md`,
`research/research.md`, `papers/papers.md`,
`foundation-models/foundation-models.md`. One-paragraph orientation
plus a bullet list with a one-line gloss per sub-page. Target
120–180 words. No content beyond the index. There are NO files
named `README.md` inside `wiki/`; the only `README.md` in the repo
is the one at the root.

### 9. Wiki root files

- `wiki/overview.md` — entry point, orientation, "start here" reading
  paths, knowledge-graph sketch. (This file was historically called
  `wiki/README.md` and was renamed so the Obsidian graph view has a
  descriptive node instead of ten nodes all called "README".)
- `wiki/index.md` — flat catalog of every wiki page, one line each.
  First stop at query time. Must be kept in sync with every add /
  rename / delete.
- `wiki/log.md` — chronological append-only record of ingests,
  queries filed back, lint passes, and refactors.

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

The wiki is a graph held together by a small set of **concept nodes**
that act as hubs. A concept node is a page in `wiki/concepts/`,
`wiki/architectures/`, or the `wiki/foundation-models/taxonomy.md`
cluster index whose primary job is to compare, contrast, and bridge
the papers below it — not to introduce a single artifact. Three
properties make a page a concept node:

- It defines an idea or family that multiple papers instantiate
  differently.
- It points outward: links to several leaves and to several other
  concept nodes, with the *why* of each link, not just the URL.
- It carries comparative content: a "design choices in the
  literature", "trade-offs and failure modes", or "open questions"
  section that summarizes how the linked artifacts differ and when
  to pick which.

The reader-side promise is that any leaf is reachable from any
other leaf in two or three hops via concept nodes. The author-side
promise is that adding a new paper means updating the relevant
concept nodes' comparative sections, not just dropping a leaf and
hoping someone connects it later.

Minimum link counts:

- Concept → concept: ≥3
- Concept → architecture: ≥2
- Concept → ≥3 paper leaves (in "Papers that exemplify this") —
  if the paper count drops below this, fold the page into a sibling
  concept node rather than letting it orphan.
- Architecture → concept: ≥3
- Architecture → sibling architecture: ≥2
- Paper leaf → complete "In the knowledge graph" block (cluster,
  architecture family, concepts, corpus, see-also)
- Broken relative links are a lint failure. Fix on sight.
- When you rename or delete a page, grep for inbound links and update
  them in the same commit.

A high-level prose summary of the concept-node pattern lives in the
top-level [README.md](README.md) for human readers; this section is
the machine-enforceable version.

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
4. Update `wiki/papers/papers.md` index table AND cluster grouping.
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

## Canonical paper slugs (as of 2026-04-27)

26 TS-FM paper leaves plus 11 pre-FM / methodology references. Use
these slugs in all cross-links. Naming convention: lowercase,
hyphen-separated; arXiv ID (when present) follows underscore in PDF
filenames, e.g. `papers/<slug>_<arxiv>.pdf`. The convention is now
**frozen**: every PDF in `papers/` matches its leaf slug exactly.

```
# Cluster 1 — Decoder-only autoregressive
timesfm       timer          timer-xl      timer-s1      lag-llama
timegpt       moirai-2

# Cluster 2 — Masked-encoder / encoder-decoder
chronos       chronos-2      moment        moirai

# Cluster 3 — Mixture-of-experts
time-moe      moirai-moe

# Cluster 4 — LLM-adapted / reprogramming
time-llm      gpt4ts         llmtime

# Cluster 5 — Lightweight / non-transformer
ttm           mamba4cast     sempo

# Cluster 6 — Multi-task / universal
units         totem          tspulse

# Cluster 7 — Continuous / flow-matching
sundial

# Cluster 8 — JEPA / latent-space prediction
lat-pfn       ts-jepa        mts-jepa

# Pre-FM precursors and baselines (no cluster)
tide

# Pre-FM representation learning (no cluster)
cpc           ts2vec

# Google Trends methodology and nowcasting (no cluster)
choi-varian   scott-varian   ross-backward-induction   gtab
ferrara-simoni   kohns-nowcast   gtrends-proper   rttp
```

When ingesting a new paper, pick a slug by lowercasing the model short
name and hyphenating. Register the new slug in:

- `wiki/papers/papers.md`
- `wiki/foundation-models/taxonomy.md`
- `wiki/index.md`
- the auto-linker mapping (see `wiki/log.md` for the most recent run)

## Canonical cluster taxonomy

The 8-cluster taxonomy lives in `wiki/foundation-models/taxonomy.md`.
H2 headings use these exact names so anchors are stable:

1. `## Cluster 1 — Decoder-only autoregressive TS-FMs`
2. `## Cluster 2 — Masked-encoder / encoder-decoder TS-FMs`
3. `## Cluster 3 — Mixture-of-experts TS-FMs`
4. `## Cluster 4 — LLM-adapted / reprogramming approaches`
5. `## Cluster 5 — Lightweight / non-transformer FMs`
6. `## Cluster 6 — Multi-task / universal unified TS models`
7. `## Cluster 7 — Continuous / flow-matching FMs`
8. `## Cluster 8 — JEPA / latent-space prediction`

A paper may belong to multiple clusters (primary + secondary). The
summary table in `taxonomy.md` is the source of truth.

## Getting started in a new session

1. Read this file.
2. Read `wiki/index.md` to see what exists.
3. Read `wiki/log.md` tail to see what was done recently.
4. Do the work.
5. Update `wiki/index.md` and `wiki/log.md` before committing.
