# Time-LLM: Time Series Forecasting by Reprogramming Large Language Models

> **Short name:** `time-llm` · **arXiv:** [2310.01728](https://arxiv.org/abs/2310.01728) · **PDF:** [local](../../papers/time-llm_2310.01728.pdf) · **Date:** 2023-10 · **Venue:** ICLR 2024

**Authors:** Ming Jin, Shiyu Wang, Lintao Ma et al.

## Abstract
Time-LLM reprograms a frozen pretrained language model for time-series forecasting. Time-series patches are aligned with LLM word embeddings through a lightweight reprogramming layer, and domain context is injected via a Prompt-as-Prefix scheme, keeping the LLM backbone entirely frozen.

## Key contributions
- Patch reprogramming: a multi-head cross-attention layer that maps patched TS tokens into a small set of "text prototypes" linearly probed from the frozen LLM's word-embedding matrix.
- Prompt-as-Prefix (PaP): a textual prefix describing dataset context, task instruction, and input statistics (trends, lags) prepended to the reprogrammed patches.
- No language-model fine-tuning; only the reprogramming module, prefix embedder, and output projection are trained.
- State-of-the-art few-shot and zero-shot results on eight standard TSLib benchmarks using Llama-7B as the default backbone.

## Architecture at a glance
Time-LLM keeps a pretrained LLM (Llama-7B or GPT-2) frozen and learns only a small reprogramming module that projects RevIN-normalised TS patches into a compact set of pseudo-word embeddings via cross-attention over condensed text prototypes. These embeddings are concatenated with a Prompt-as-Prefix token sequence and passed through the frozen LLM; the prefix positions are discarded and the remaining output embeddings are flattened and linearly projected to the forecast horizon. Forecasting is direct (not autoregressive): patched TS tokens are reprogrammed into LLM word embeddings, passed through the frozen LLM in a single forward pass, and the output is linearly projected to the full forecast horizon H (Jin et al. Section 3).

## Why it matters
Time-LLM showed that substantial TS forecasting capability can be unlocked from frozen LLMs with only modest adapter modules. Together with GPT4TS it defined the "reprogramming" branch of LLM-for-TS research and demonstrated that a small bridge between patch space and word-embedding space is enough to turn a text LLM into a competitive forecaster.

## Strengths
- Leaves the LLM backbone entirely frozen, avoiding catastrophic forgetting and keeping parameter updates to a small module that trains in a few epochs on a single GPU.
- Reports consistent improvements over GPT4TS, PatchTST, DLinear, and the TSLib transformer family on long-horizon forecasting in both full-shot and few-shot settings.
- Prompt-as-Prefix is a concrete mechanism for injecting prior knowledge (domain, statistics) that other TS-FMs have no interface for; this is one of the few approaches that can use textual side information.
- Compatible with off-the-shelf quantisation of the frozen LLM, so memory footprint is controllable independently of training.

## Limitations and open critiques
- Inference cost is dominated by a Llama-7B forward pass for every window, making Time-LLM far heavier at deploy time than purpose-built TS foundation models of comparable accuracy.
- Subsequent work has argued that frozen LLMs add little TS-specific signal: ablation studies (Tan et al., "Are Language Models Actually Useful for Time Series Forecasting?") show that randomly initialised or attention-ablated backbones give similar accuracy, challenging the "reprogramming unlocks pretrained knowledge" narrative.
- Text prototype selection via linear probing is presented heuristically; the paper does not rigorously analyse which word embeddings the attention heads actually attend to.
- Prompt-as-Prefix requires hand-authored dataset descriptions; there is no prescription for automatically constructing prompts across heterogeneous corpora.
- Multivariate forecasting is handled via channel independence, like PatchTST, rather than modelling cross-channel dependencies.

## Follow-up work and dialogue
Time-LLM and [GPT4TS](./gpt4ts.md) are the two canonical LLM-adaptation strategies: GPT4TS fine-tunes only LayerNorm and positional embeddings of a frozen GPT-2, while Time-LLM adds a reprogramming bridge plus a textual prompt prefix. This is the foundational axis for later work. [LLMTime](./llmtime.md) sits at the far end by keeping the LLM fully frozen and encoding numbers as text, without any adapter at all. [Chronos](./chronos.md) is an explicit response that argues a purpose-built T5 with a dedicated bin vocabulary beats reprogramming text LLMs, and billion-parameter TS-native models such as [time-moe](./time-moe.md) and [timesfm](./timesfm.md) argue that native pretraining is a more direct path than adapting text models at all.

## Reproducibility
- **Open weights:** partial — adapter weights trained on the TSLib benchmarks; LLM backbones (Llama-7B, GPT-2) are loaded from public hubs.
- **Code:** public repo stated as `https://github.com/KimMeen/Time-LLM`.
- **Training data:** long-horizon TSLib datasets (ETT, Weather, Electricity, Traffic, ILI) following Wu et al. experimental pipeline; fully public.
- **Compute to retrain:** the paper emphasises that only a small adapter is trained and the method is "optimised with only a small set of time series and a few training epochs"; exact GPU-hours are not disclosed.
- **Deployment footprint:** dominated by the frozen LLM (Llama-7B or GPT-2); the paper notes compatibility with quantisation but does not report CPU inference latency.

## When to cite this paper
Cite Time-LLM as the canonical reference for LLM reprogramming via a cross-attention bridge into text-prototype space and for Prompt-as-Prefix as a way to inject textual side information into a forecasting model. It is the natural counterpart to GPT4TS whenever the discussion is about how much of the LLM to touch.

## In the knowledge graph
- **Cluster:** [LLM-adapted / reprogramming approaches](../foundation-models/taxonomy.md#cluster-4--llm-adapted--reprogramming-approaches)
- **Architecture family:** [LLM reprogramming](../architectures/llm-reprogramming.md)
- **Related concepts:** [patch tokenization](../concepts/patch-tokenization.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md), [in-context learning](../concepts/in-context-learning.md), [revin normalization](../concepts/revin-normalization.md)
- **See also:** [gpt4ts](./gpt4ts.md), [llmtime](./llmtime.md), [chronos](./chronos.md), [time-moe](./time-moe.md)
