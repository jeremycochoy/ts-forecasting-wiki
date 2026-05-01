# One Fits All: Power General Time Series Analysis by Pretrained LM (GPT4TS / FPT / OFA)

> **Short name:** `gpt4ts` · **arXiv:** [2302.11939](https://arxiv.org/abs/2302.11939) · **PDF:** [local](../../papers/gpt4ts_2302.11939.pdf) · **Date:** 2023-02 · **Venue:** NeurIPS 2023 Spotlight

**Authors:** Tian Zhou, Peisong Niu, Xue Wang, Liang Sun, Rong Jin (Alibaba DAMO)

## Abstract
GPT4TS, also known as FPT or OFA, demonstrates that a frozen pretrained language model can serve as a strong backbone for a wide range of time-series tasks. Self-attention and feed-forward blocks are frozen, and only layer-norm, positional-embedding, and small input/output projection parameters are fine-tuned downstream.

## Key contributions
- Frozen Pretrained Transformer (FPT) paradigm for time series using GPT-2, BERT, or BEiT backbones.
- Fine-tuning limited to layer-norm, positional embeddings, and a lightweight linear input/output head — everything else is frozen.
- Universal applicability across six TS task families: long-term forecasting, short-term forecasting, classification, anomaly detection, imputation, and few/zero-shot forecasting.
- Theoretical and empirical analysis linking frozen self-attention to principal-component-like operations, motivating why a text LM transfers at all.

## Architecture at a glance
GPT4TS takes a pretrained transformer and passes patched, RevIN-normalised time-series tokens through it. Self-attention and feed-forward layers are frozen; only LayerNorm parameters, positional embeddings, and the input/output linear projections are trained. The GPT2(6) variant (first six blocks of GPT-2) is the default configuration used across tasks. Forecasting is direct: PatchTST-style patched tokens are processed through the frozen GPT-2, and the output is projected to the full forecast horizon H in a single pass (Zhou et al. Section 3).

The choice to **fine-tune only LayerNorm, positional embeddings, and the input/output projections** while freezing self-attention and feed-forward blocks is motivated in paper §3 + §6: the paper analytically links frozen self-attention to a fixed PCA-style dimensionality reducer, so the pretrained transformer already implements a "universal compute engine" that need not change for TS. What does need to adapt is the *normalization statistics* (LayerNorm) and the *positional structure* (PE) — both of which are TS-specific (different value scales, different sequence semantics) but cheap to retune without disturbing the underlying computation. Fine-tuning more (e.g. attention or FFN) risks catastrophic forgetting of the linguistic prior the PCA argument relies on; fine-tuning less (e.g. only the head) leaves the input mismatch unaddressed.

### Sizes (Zhou et al. §3 + §4.10; backbone arch per Radford et al. 2019)

| Variant | Backbone | Layers (kept) | d_model | d_ff | Heads | d_kv | Total params | Trainable params | Patch | Context |
|---|---|---|---|---|---|---|---|---|---|---|
| GPT2(6) — default for forecasting / classification / anomaly detection | GPT-2 small | 6 (of 12) | 768 | 3072 | 12 | 64 | ~62M | ~4M | PatchTST-style; size not stated in paper (typically 16 with stride 8) | task-dependent |
| GPT2(3) — used for imputation | GPT-2 small | 3 (of 12) | 768 | 3072 | 12 | 64 | ~31M | ~4M | same | same |
| GPT2(12) full (ablation) | GPT-2 small | 12 | 768 | 3072 | 12 | 64 | 124M | ~4M | same | same |
| BERT(6) / BEiT(6) ablations | BERT / BEiT | 6 | 768 | 3072 | 12 | 64 | similar | adapter-only | same | same |

Backbone `(L, d, d_ff, h)` is inherited from GPT-2 small (paper does not retabulate). Trainable parameters: positional embeddings + LayerNorm scale/shift + input patch embedder + output linear projection. Self-attention and FFN weights are frozen across all variants.

## Why it matters
GPT4TS is the canonical early demonstration that large language models can be repurposed as generic TS backbones with minimal parameter updates. It is the reference baseline that every later LLM-for-TS paper compares against, and its PCA interpretation of frozen self-attention is frequently cited as intuition for why cross-modality transfer works.

## Strengths
- Establishes a single unified recipe that reaches state-of-the-art or near-state-of-the-art on six heterogeneous task families, including a 9.3% average MSE reduction over TimesNet on long-term forecasting and an 86.72 mean F1 on five anomaly detection benchmarks.
- Fine-tunes only LayerNorm and positional embeddings — the simplest possible adapter — which makes training cheap and ablation clean.
- The PCA-style analysis of self-attention gives a concrete hypothesis for why a text-pretrained transformer can act as a "universal compute engine" for TS, rather than treating the transfer as a black-box empirical result.
- Works on multiple backbone families (GPT-2, BERT, BEiT), showing the effect is not idiosyncratic to GPT-2.

## Limitations and open critiques
- Tan et al. ("Are Language Models Actually Useful for Time Series Forecasting?", 2024) show that ablating attention or replacing the GPT-2 weights with random initialisation yields essentially the same accuracy, directly challenging the claim that language pretraining contributes useful TS structure.
- The paper evaluates on the TSLib benchmark suite (ETT, Weather, ECL, Traffic, ILI, UEA), whose saturation and potential leakage are now widely discussed; more recent zero-shot benchmarks ([GIFT-Eval](../datasets-benchmarks/gift-eval.md), Chronos zero-shot suite) are not covered.
- Uses GPT2(6) but does not systematically study how many layers to keep, or why six is the right number; the choice is empirical.
- Deployment footprint is still that of a GPT-2 forward pass for every window, with no discussion of CPU inference or latency.
- The frozen backbone limits the method's ability to specialise to new modalities, as later TS-native foundation models demonstrate with much smaller parameter counts.

## Follow-up work and dialogue
GPT4TS and [Time-LLM](./time-llm.md) are the two canonical LLM-adaptation strategies: GPT4TS just fine-tunes LayerNorm of a frozen GPT-2, while Time-LLM adds a learnable reprogramming layer plus a Prompt-as-Prefix. [LLMTime](./llmtime.md) is the purist limit — no fine-tuning at all — arguing the pretrained LLM already contains the forecasting prior. Purpose-built TS foundation models such as [chronos](./chronos.md), [moirai](./moirai.md), [timesfm](./timesfm.md), and [time-moe](./time-moe.md) are the counter-argument: train natively on TS data and you can beat the reprogramming recipes with one to two orders of magnitude fewer parameters. [TTM](./ttm.md) pushes this further and outperforms GPT4TS with roughly 1M parameters on a CPU.

## Reproducibility
- **Open weights:** partial — trained adapter checkpoints; GPT-2 / BERT / BEiT backbones come from public hubs.
- **Code:** public repo stated as `https://github.com/DAMO-DI-ML/One_Fits_All`.
- **Training data:** the standard TSLib suite (ETT, Weather, ECL, Traffic, ILI, UEA classification, five anomaly benchmarks, M4 short-term); fully public.
- **Compute to retrain:** not disclosed in detail; because only LayerNorm, positional embeddings, and linear heads are trained, fine-tuning per dataset is cheap and runs on a single GPU.
- **Deployment footprint:** GPT2(6) backbone plus small adapters; inference is a partial GPT-2 forward pass. The TTM paper reports GPT4TS CPU inference at ~0.25s per batch on their evaluation setting.

## When to cite this paper
Cite GPT4TS as the canonical reference for the frozen-LLM-as-TS-backbone recipe (fine-tune only LayerNorm and positional embeddings), for the PCA-style interpretation of frozen self-attention, and as the baseline that every subsequent LLM-for-TS and TS-native foundation model compares against.

## In the knowledge graph
- **Cluster:** [LLM-adapted / reprogramming approaches](../foundation-models/taxonomy.md#cluster-4--llm-adapted--reprogramming-approaches)
- **Architecture family:** [LLM reprogramming](../architectures/llm-reprogramming.md)
- **Related concepts:** [patch tokenization](../concepts/patch-tokenization.md), [multi-task universal](../concepts/multi-task-universal.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md), [revin normalization](../concepts/revin-normalization.md)
- **See also:** [time-llm](./time-llm.md), [llmtime](./llmtime.md), [ttm](./ttm.md), [chronos](./chronos.md)
