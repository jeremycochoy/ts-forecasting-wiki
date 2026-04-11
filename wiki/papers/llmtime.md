# Large Language Models Are Zero-Shot Time Series Forecasters

> **Short name:** `llmtime` · **arXiv:** [2310.07820](https://arxiv.org/abs/2310.07820) · **PDF:** [local](../../papers/llmtime_2310.07820.pdf) · **Date:** 2023-10 · **Venue:** NeurIPS 2023

**Authors:** Nate Gruver, Marc Finzi, Shikai Qiu, Andrew Gordon Wilson (NYU / CMU)

## Abstract
LLMTime shows that off-the-shelf large language models such as GPT-3 and LLaMA-2 can forecast time series zero-shot when numerical values are encoded as carefully formatted text. A token-level distribution over digits is converted into a continuous predictive density, matching or exceeding purpose-built TS models on several benchmarks without any fine-tuning.

## Key contributions
- Numbers-as-text tokenisation scheme (digit-by-digit with explicit separators) designed to avoid BPE-induced misalignment between numerical characters and tokens.
- Change-of-variables construction that turns a hierarchical softmax over digit strings into a flexible continuous mixture density, enabling proper NLL and [CRPS](../evaluation/metrics.md#21-crps--continuous-ranked-probability-score) evaluation.
- Evidence that vanilla GPT-3 and LLaMA-2 match or exceed purpose-built TS models zero-shot on Darts, M4, and Informer benchmarks.
- Observation that GPT-4 sometimes underperforms GPT-3, attributed to its number tokenizer and to alignment/RLHF artefacts; base models fit the MMLU-to-forecasting scaling curve, chat-tuned models do not.
- Demonstrations that LLMs can handle missing values via textual markers (`NaN`) and accept textual side information.

## Architecture at a glance
LLMTime uses unmodified pretrained LLMs as forecasters. Numeric series are serialised into digit-level text with spaces between digits and commas between timesteps; decimals are dropped under a fixed-precision convention. Sampling next-token distributions and reversing the serialisation yields a mixture of per-bin uniform densities, which the paper shows can represent heavy-tailed and multimodal distributions competitively against GMMs and fixed bins.

## Why it matters
LLMTime provided a provocative zero-shot baseline showing that language models trained only on text already contain enough sequence structure to forecast numeric time series. It crystallised the discrete-tokenisation-of-values idea and is the canonical counterexample to the view that TS forecasting requires TS-specific pretraining.

## Strengths
- Truly zero-shot: no fine-tuning, no adapter, no dataset-specific code — the method is essentially a careful string formatter plus a sampler, which makes it trivially reproducible over any text LLM.
- Provides proper probabilistic forecasts with NLL and CRPS, not just point predictions, by converting digit distributions into a continuous density — something reprogramming methods like GPT4TS and Time-LLM lack.
- Demonstrates a clean monotone relationship between underlying LLM quality (MMLU accuracy, LLaMA size) and forecasting quality, supporting the compressor-as-forecaster hypothesis.
- Graceful missing-value handling via literal `NaN` tokens; no imputation required.
- Natural path to text-conditioned forecasting, where textual side information is concatenated in the prompt, which most TS-FMs cannot do at all.

## Limitations and open critiques
- The per-step cost is enormous: each forecast horizon step requires sampling several digit tokens from a 7B-70B LLM, so LLMTime is one to three orders of magnitude slower than purpose-built forecasters (the [TTM](./ttm.md) paper reports LLMTime at thousands of seconds on CPU for benchmarks where TTM runs in milliseconds).
- Digit-level text tokenisation is brittle: GPT-4 performs worse than GPT-3 because its tokenizer merges digit groups differently, and LLaMA needs the opposite space handling from GPT. This is fundamentally a tokenizer hack, not a principled encoder.
- Zero-shot accuracy saturates at the level of a decent PatchTST on standard TSLib benchmarks; Chronos and later TS-native models substantially exceed it with purpose-built vocabularies.
- Uncertainty calibration degrades under RLHF/chat alignment, so the approach is incompatible with most production LLM endpoints.
- Strictly univariate and memory-limited by the LLM's context window, with no native mechanism for cross-channel dependencies.

## Follow-up work and dialogue
[Chronos](./chronos.md) is the direct response: LLMTime showed the value of a discrete-digit vocabulary, but Chronos argues a purpose-built T5 with a fixed bin vocabulary is strictly better than hijacking a text tokenizer. Both discretise values; LLMTime uses text digits through a frozen general LLM while Chronos trains a TS-specific T5 with a dedicated bin vocabulary. [Sundial](./sundial.md) attacks both: its TimeFlow objective argues that any discretisation (LLMTime, Chronos) throws away continuous structure that flow-matching can preserve. [GPT4TS](./gpt4ts.md) and [Time-LLM](./time-llm.md) represent the opposite reprogramming response, keeping the LLM as a backbone rather than a text interface.

## Reproducibility
- **Open weights:** not applicable — the method uses off-the-shelf public LLMs (LLaMA-2, LLaMA, GPT-3/4) plus a serialiser.
- **Code:** public repo referenced in paper at `https://github.com/ngruver/llmtime`.
- **Training data:** none — LLMTime is fully zero-shot. Evaluation uses the Darts, [Monash](../datasets-benchmarks/monash-archive.md), and Informer benchmark collections.
- **Compute to retrain:** no training; inference cost is dominated by LLM API calls / local forward passes of a 7B-70B model per serialised window.
- **Deployment footprint:** bounded by the underlying LLM; the paper evaluates up to LLaMA-2 70B and GPT-4. Latency is high relative to purpose-built TS models — the TTM paper measures ~2500 s CPU for a comparable evaluation setting.

## When to cite this paper
Cite LLMTime as the canonical reference for zero-shot time-series forecasting with an unmodified text LLM, for the digit-tokenisation-plus-change-of-variables recipe that converts next-token distributions into continuous densities, and for the specific observation that alignment/RLHF and BPE tokenisers degrade numerical forecasting.

## In the knowledge graph
- **Cluster:** [LLM-adapted / reprogramming approaches](../foundation-models/taxonomy.md#cluster-4--llm-adapted--reprogramming-approaches)
- **Architecture family:** [LLM reprogramming](../architectures/llm-reprogramming.md)
- **Related concepts:** [value quantization](../concepts/value-quantization.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md), [probabilistic forecasting](../concepts/probabilistic-forecasting.md), [in-context learning](../concepts/in-context-learning.md)
- **See also:** [time-llm](./time-llm.md), [gpt4ts](./gpt4ts.md), [chronos](./chronos.md), [sundial](./sundial.md)
