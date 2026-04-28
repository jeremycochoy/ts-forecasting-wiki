# Real-Time Trend Prediction via Continually-Aligned LLM Query Generation

> **Short name:** `rttp` · **arXiv:** [2601.17567](https://arxiv.org/abs/2601.17567) · **PDF:** [local](../../papers/rttp_2601.17567.pdf) · **Date:** 2026-01 · **Venue:** WWW 2026 (Meta Platforms)

**Authors:** Zijing Hui, Wenhan Lyu, Shusen Wang, Li Chen, Chu Wang (Meta Platforms)

## Abstract
Trending-news detection in low-traffic search faces a cold-start problem: query volume is too sparse for the Poisson-spike or keyword-frequency methods that X and Google Trends rely on. RTTP inverts the pipeline. Instead of waiting for users to issue queries, a continually-aligned LLM (CL-LLM) *generates* search-style queries from newly-created Facebook posts, and synthetic queries are then scored by engagement strength and creator authority. A `Mix-Policy DPO` continual-training strategy combines on-policy and off-policy preference data to keep the LLM aligned with shifting content while preserving general reasoning. Deployed on Facebook and Meta AI, RTTP reports +91.4% precision@500 on tail-trend detection and +19% query generation accuracy over baselines (RTTP §4.2, §4.3).

## Key contributions
- Inverts the trend-detection pipeline: generate queries from posts instead of waiting for query volume to form, addressing the cold-start problem in low-traffic search (RTTP §1).
- `CL-LLM`: a LLaMA 3.3 70B continually trained on user-query / generated-query pairs to convert post titles + bodies into search-style queries plus location metadata (RTTP §3.2).
- `Mix-Policy DPO`: a preference-based continual-learning recipe that mixes on-policy (model-correct) and off-policy (model-mispredicted) examples at a 1:9 off-to-on ratio to avoid the "squeezing effect" of off-policy DPO and preserve reasoning under frequent retraining (RTTP §3.2.1).
- Engagement-weighted ranking distinguishing deep engagement (comments, reshares) from shallow engagement (likes, reactions), combined with a follower-count + verification-based `CreatorAuthority` term (RTTP Eqs. 1-2).

## Method at a glance
A new post triggers CL-LLM (LLaMA 3.3 70B), which generates candidate queries plus location metadata. Queries are ranked by `TrendingScore = CreatorQuality + Σ w_i · E_i`, with `w_i` weighting deeper engagement (comments, reshares) over shallow reactions (RTTP Eqs. 1-2). Retraining triggers when `Recall@3` (success if any ground-truth query for the post appears in the top-3 generated set) drops by more than ~10%, and uses Mix-Policy DPO with a fixed `ρ = 1:9` off-to-on ratio (RTTP §3.2.1).

Two design rationales are explicit in the paper. (a) **Inverting the pipeline** (post → query, rather than query → trend) addresses the cold-start regime where Poisson-on-volume methods need search traffic that does not yet exist — generating queries directly from new posts surfaces tail trends that volume-based detectors miss entirely (paper §1, Fig. 2 reports +91.4% precision@500 over a Poisson-on-volume baseline). (b) **Mix-Policy DPO at the 1:9 off-to-on ratio** (paper §3.2.1, Fig. 4) avoids the *squeezing effect* of pure off-policy DPO, where probability mass collapses onto a single answer and reasoning degrades — the on-policy mix preserves model coherence while off-policy examples teach the new behavior, letting MMLU stay within ~5% across a month of retraining vs. SFT-only collapse to near-zero after two rounds.

## Why it matters
RTTP modernizes the Google-Trends-style trend-detection pipeline by inserting an LLM as a *query generator*, not a forecaster. It is the closest published analog to Google Correlate — discontinued in 2019 — for finding queries that anticipate a target signal. Mix-Policy DPO is the more general artifact: a recipe for continual preference-tuning of instruction-aligned LLMs without catastrophic forgetting of MMLU-level reasoning.

## Strengths
- Production deployment at Meta scale, with metrics on internal Facebook logs and human-curated trend annotations.
- Addresses a problem (cold-start in low-traffic search) where pure-volume methods are demonstrably broken; the +91.4% precision@500 improvement over a Poisson-on-volume baseline (RTTP Fig. 2) is large and statistically significant (`p < 0.001`, 300 days).
- Mix-Policy DPO has independent value: on MMLU, SFT-based continual training collapses to near-zero after two retraining rounds, while Mix-Policy DPO loses only ~5% across a month (RTTP Fig. 4).
- Honest about why prior continual-pretraining recipes (Ibrahim 2024, Parmar 2024) fail on a post-aligned LLaMA 3.3 70B with distribution-shifted social-media data (RTTP §4.4).

## Limitations and open critiques
- Short paper (4 pages, WWW 2026) — design choices (prompt template, engagement weights, the `1:9` off-to-on ratio) are stated without ablation.
- Evaluation is on Meta-internal Facebook logs and human-rated trend labels; not externally reproducible. No public code, eval set, or weights.
- Reasoning evaluation uses MMLU only (linguistic / general-knowledge subset); no numeric or time-series reasoning benchmarks despite the framing as a trend *prediction* system.
- Does not connect to the Google-Trends-quality literature — Medeiros-Pires sampling noise, G-TAB calibration, or BSTS / spike-and-slab keyword selection from the [nowcasting tradition](../benchmarks/rebuilding-google-trends-corpus.md). The "synthetic query generation" framing reinvents the keyword-selection problem without citing the econometric prior art.
- The synthetic queries still need a downstream search-volume / engagement signal to be ranked as trending. The LLM contributes recall, not the trend judgment.

## Follow-up work and dialogue
RTTP sits at an unusual intersection. On one axis it inherits from the Google-Trends nowcasting lineage — Choi-Varian 2011 "Predicting the Present", Scott-Varian 2014 BSTS with spike-and-slab selection, and the Twittermonitor Poisson-burst model the paper cites directly (Mathioudakis-Koudas 2010). On the other it draws on the [LLM-reprogramming cluster](../architectures/llm-reprogramming.md) — [LLMTime](./llmtime.md), [Time-LLM](./time-llm.md), [GPT4TS](./gpt4ts.md) — but uses the LLM as a *query-string generator* rather than a numeric forecaster. The closest sibling is LLMTime in spirit (frozen-LLM-as-engine for a TS application), not mechanism. Mix-Policy DPO contributes to a separate continual-learning thread (Step-DPO, Full-Step-DPO) that future TS-FM continual-pretraining work may borrow when retrofitting [TimesFM](./timesfm.md)-style models to drift-prone settings.

## Reproducibility
- **Open weights:** no — internally fine-tuned LLaMA 3.3 70B, not released.
- **Code:** not stated.
- **Training data:** Meta-internal — one month of Facebook search logs, engagement signals, and human-annotated trend labels. Privacy-governed; not externally accessible.
- **Compute:** not disclosed; §4.4 notes GPU constraints force aggressive per-cycle sampling.
- **Deployment footprint:** Facebook + Meta AI production; LLaMA 3.3 70B backbone, retrained when `Recall@3` regresses by >10%.

## When to cite this paper
Cite RTTP as the canonical reference for **LLM-driven query generation for cold-start trend detection in low-traffic search** and for **Mix-Policy DPO continual training** of an instruction-aligned LLM. Also the right citation for the pattern "use an LLM to *propose* queries, then a classical signal to *rank* them" that bridges the Google-Trends nowcasting tradition with LLM-application work.

## In the knowledge graph
- **Cluster:** Pre-FM trend-detection methodology — LLM query generation. Not in the eight-cluster TS-FM taxonomy; adjacent to [LLM reprogramming](../architectures/llm-reprogramming.md) but applied to query strings rather than numeric series.
- **Methodology hubs:** [Google Trends data](../datasets-benchmarks/google-trends-data.md), [LLM reprogramming](../architectures/llm-reprogramming.md), [Rebuilding the Google Trends corpus](../benchmarks/rebuilding-google-trends-corpus.md) (RTTP is referenced there as a candidate-expansion mechanism).
- **Related concepts:** [zero-shot forecasting](../concepts/zero-shot-forecasting.md) (CL-LLM runs zero-shot per post), [synthetic data augmentation](../concepts/synthetic-data-augmentation.md) (synthetic queries as search inputs).
- **See also:** [LLMTime](./llmtime.md), [Time-LLM](./time-llm.md), [TimesFM](./timesfm.md).

## Related wiki pages
- [Google Trends data](../datasets-benchmarks/google-trends-data.md)
- [LLM reprogramming](../architectures/llm-reprogramming.md)
- [Rebuilding a TimesFM-quality Google Trends corpus](../benchmarks/rebuilding-google-trends-corpus.md)
