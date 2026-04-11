# MOMENT: A Family of Open Time-series Foundation Models

> **Short name:** `moment` · **arXiv:** [2402.03885](https://arxiv.org/abs/2402.03885) · **PDF:** [local](../../papers/moment_2402.03885.pdf) · **Date:** 2024-10 · **Venue:** ICML 2024

**Authors:** Mononito Goswami, Konrad Szafer, Arjun Choudhry et al. (CMU Auton Lab)

## Abstract
MOMENT is an open family of T5-shaped encoder-only foundation models pretrained with masked patch reconstruction over the Time Series Pile, a large aggregation of public datasets. Small task heads adapt the same backbone to forecasting, classification, anomaly detection, and imputation.

## Key contributions
- Assembly of the Time Series Pile from Informer, [Monash](../datasets-benchmarks/monash-archive.md), UCR/UEA, and TSB-UAD datasets as a unified pretraining corpus with explicit disjoint train/val/test partitions to guard against leakage.
- Encoder-only transformer with PatchTST-style patching (T=512, P=8, N=64) and RevIN-style instance normalization.
- Masked patch reconstruction pretraining with a 30% mask rate and a learned [MASK] embedding rather than zero-masking.
- Multi-task deployment via lightweight per-task heads for long/short forecasting, classification, anomaly detection, and imputation.
- Open release on Hugging Face in Small (40M), Base (125M), and Large (385M) sizes.

## Architecture at a glance
MOMENT follows the shape of T5 encoder but is randomly initialized (the paper explicitly ablates and prefers random init over language-model init for pretraining loss). It operates on patched, RevIN-normalized real-valued tokens of length T=512 split into 64 patches of length 8. A fraction of patches is masked and reconstructed during pretraining. At downstream time, the backbone is frozen or lightly tuned while a small task-specific head is trained on top.

## Why it matters
MOMENT provided one of the first fully open TS foundation model families with broad multi-task capability, a reproducible pretraining corpus, and competitive performance across four canonical TS tasks, acting as an accessible baseline for the open-source community.

## Strengths
- Fully open: weights at 40M / 125M / 385M, training corpus (Time Series Pile), data splits, and fine-tuning recipes are all released under an open license, which is unusual for the TS-FM space and makes MOMENT a natural reproducibility anchor — see [../research/reproducibility.md](../research/reproducibility.md).
- Multi-task demonstration is genuine, not hand-waved: the same backbone is benchmarked on long-horizon forecasting, short-horizon forecasting, imputation, anomaly detection, and classification, with explicit zero-shot and linear-probe protocols rather than full fine-tuning.
- Provides one of the first clean empirical comparisons showing that a transformer randomly initialized on time series outperforms the same architecture initialized from GPT-2 or T5 language weights, directly refuting the "LLMs-transfer-for-free" narrative that [gpt4ts](./gpt4ts.md) and [time-llm](./time-llm.md) relied on.
- The probing experiments (trend, amplitude, frequency, baseline shift, phase) give concrete evidence that the masked-reconstruction objective learns interpretable time-series features rather than only local denoising.

## Limitations and open critiques
- Fixed 512-step input and 8-step patches are hard-coded; MOMENT cannot natively ingest longer contexts or variable patch sizes, unlike [moirai](./moirai.md) (frequency-aware patches) or [timer-xl](./timer-xl.md).
- Only point forecasts: the forecasting head is a flatten-and-project linear head producing deterministic outputs, so MOMENT cannot be compared against probabilistic foundation models on [CRPS](../evaluation/metrics.md#21-crps--continuous-ranked-probability-score)/[WQL](../evaluation/metrics.md#23-wql--weighted-quantile-loss) without extra scaffolding; [chronos](./chronos.md) and [lag-llama](./lag-llama.md) sit in a different regime.
- Despite the "limited supervision" framing, the headline long-horizon results are obtained by linear probing on each target dataset, so MOMENT is only partially zero-shot. In Table 2 PatchTST, trained per dataset, still edges it out.
- The Time Series Pile overlaps in domain with most downstream benchmarks (ETT, Weather, Electricity, Traffic), so "in-distribution probing" is a more accurate label than "zero-shot transfer" for several of the reported numbers.
- Single pretraining corpus, two epochs, batch 2048: the scaling-law evidence is limited to three model sizes at one token budget, so no Chinchilla-style curve is produced.
- Univariate channel-independent processing: multivariate series are split into channels along the batch axis; there is no cross-channel attention mechanism.

## Follow-up work and dialogue
MOMENT occupies the same "masked-encoder with patch tokens" design space as [moirai](./moirai.md), but trades MOIRAI's frequency-aware input patches and [LOTSA](../datasets-benchmarks/lotsa.md) corpus for a T5-shaped backbone and the Time Series Pile. [moirai-moe](./moirai-moe.md) argues the frequency-specialization axis is the wrong one and replaces it with MoE routing; by extension the same critique applies to MOMENT's single dense encoder. [units](./units.md) pushes the multi-task framing further by supporting generative tasks end-to-end, while [chronos](./chronos.md) and [chronos-2](./chronos-2.md) chose probabilistic tokenization and covariate support rather than multi-task breadth. On the "random init beats LLM init" axis, MOMENT is the clearest empirical rebuttal to the LLM-reprogramming line of work — see [../architectures/llm-reprogramming.md](../architectures/llm-reprogramming.md).

## Reproducibility
- **Open weights:** yes — `AutonLab/MOMENT-1-small`, `-base`, `-large` on HuggingFace.
- **Code:** public, referenced in the paper.
- **Training data:** fully public — The Time Series Pile is released on HuggingFace (`AutonLab/Timeseries-PILE`) with documented splits.
- **Compute to retrain:** 2 epochs over the Time Series Pile, batch size 2048, AdamW with cosine schedule from 1e-4 to 1e-5, mixed precision; GPU hours and exact FLOPs not disclosed in the main text.
- **Deployment footprint:** 40M / 125M / 385M parameters; fixed 512-step context with 8-step patches; linear-probe fine-tuning runs on a single consumer GPU as described in the fine-tuning settings.

## When to cite this paper
Cite MOMENT as the canonical open-weight, open-corpus TS foundation model family for multi-task analysis (forecasting, classification, anomaly detection, imputation). It is also the right citation for the specific finding that random initialization outperforms language-model initialization for masked TS pretraining, and for the Time Series Pile as a reproducible public pretraining corpus.

## In the knowledge graph
- **Cluster:** [Masked-encoder / encoder-decoder TS-FMs](../foundation-models/taxonomy.md#cluster-2--masked-encoder--encoder-decoder-ts-fms)
- **Architecture family:** [Masked encoder](../architectures/masked-encoder.md)
- **Related concepts:** [patch tokenization](../concepts/patch-tokenization.md), [revin normalization](../concepts/revin-normalization.md), [multi-task universal](../concepts/multi-task-universal.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md)
- **Dataset / corpus:** [Time Series Pile](../datasets-benchmarks/time-series-pile.md)
- **See also:** [moirai](./moirai.md), [chronos](./chronos.md), [units](./units.md)
