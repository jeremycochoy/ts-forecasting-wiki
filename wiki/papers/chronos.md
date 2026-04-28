# Chronos: Learning the Language of Time Series

> **Short name:** `chronos` · **arXiv:** [2403.07815](https://arxiv.org/abs/2403.07815) · **PDF:** [local](../../papers/chronos_2403.07815.pdf) · **Date:** 2024-11 · **Venue:** TMLR 2024

**Authors:** Abdul Fatir Ansari, Lorenzo Stella, Caner Turkmen et al. (AWS)

## Abstract
Chronos treats forecasting as language modeling on quantized time series. Values are mean-scaled and uniformly binned into a fixed vocabulary, then fed to an off-the-shelf T5 encoder-decoder trained with categorical cross-entropy. At inference, sampled tokens are dequantized to produce probabilistic forecasts.

## Key contributions
- Tokenization scheme combining mean-scaling with uniform value quantization into a fixed categorical vocabulary.
- Reuse of unmodified T5 architectures across a family of sizes (20M to 710M parameters).
- TSMix data augmentation (convex combinations of real series) and KernelSynth (Gaussian-process-based synthetic generation).
- Broad benchmarking across 42 datasets, showing strong zero-shot performance.

## Architecture at a glance
Chronos is an encoder-decoder transformer based directly on the T5 family. Time series are preprocessed into discrete token sequences via mean-scaling and uniform binning into 4096 bins over the interval [-15, +15]. Training optimizes next-token cross-entropy, and forecasts are obtained by sampling multiple trajectories and dequantizing them to recover continuous predictive distributions. Each token represents a single scalar timestep (1-in, 1-out); forecasting is autoregressive token-by-token, where each timestep is independently quantized into one of ~4096 bins (Ansari et al. Section 3).

Two non-obvious design choices are motivated in paper §3 / Appendix A.2: (a) **uniform** rather than quantile binning — quantile bins encode CDF structure that does not transfer across heterogeneous datasets, and the closed `[-15s, +15s]` interval prevents trending series from overflowing the vocabulary (the failure mode the paper acknowledges in Fig. 13); (b) **cross-entropy over MSE / parametric NLL**, because a categorical head can express arbitrary including multi-modal predictive distributions without committing to a Gaussian / Student-t family — the paper accepts that the loss is not distance-aware (bin `i+1` is no closer to bin `i` than to `i+1000`) and explicitly suggests ordinal or label-smoothed variants as future work.

## Why it matters
Chronos showed that minimal tokenization plus an existing language-model architecture, without any TS-specific inductive bias, can compete with purpose-built time-series models. It crystallized the "language modeling for time series" viewpoint and became a standard baseline for subsequent TS foundation models.

## Strengths
- Deliberately minimal design: a stock T5 with only the vocabulary size changed reaches state-of-the-art in-domain performance and strong zero-shot performance across 42 datasets, showing that categorical cross-entropy over binned values is a surprisingly competitive training signal.
- Regression-via-classification lets Chronos express arbitrary, including multi-modal, predictive distributions without committing to a parametric head like Gaussian or Student-t, which is a concrete advantage over DeepAR-style approaches.
- TSMixup and KernelSynth are released together with the benchmark protocol, providing a reproducible augmentation pipeline (10M mixed series plus 1M GP-synthesized series at a 9:1 ratio) that later papers reuse as a baseline augmentation recipe.
- The family spans 20M / 46M / 200M / 710M parameters trained under identical conditions, making it one of the clearest within-paper size-sweeps for probabilistic TS foundation models.

## Limitations and open critiques
- Because tokens represent bin centers in [-15s, +15s] where s is the context mean absolute value, any future value that drifts outside this range overflows and is clipped. The paper explicitly calls this out and shows Chronos underpredicts strong trends (the Air Passengers example, Fig 13) when context is short.
- The categorical cross-entropy objective is not distance-aware: bin i+1 is no closer to bin i than bin i+1000. The authors acknowledge this and suggest ordinal or label-smoothed variants as future work.
- The 512-token context is short relative to high-frequency datasets and the paper notes the evaluation suite under-represents sub-hourly data, so the context-length ablation cannot cleanly resolve whether longer context would help; see [../benchmarks/state-of-the-art.md](../benchmarks/state-of-the-art.md).
- Univariate only, with no covariate support in v1 — a gap that [chronos-2](./chronos-2.md) was explicitly designed to close.
- Chronos partitions its 55-source collection into *pretraining-only* (13 datasets), *Benchmark I* (15 in-domain, used for both training and evaluation), and *Benchmark II* (27 zero-shot) — Chronos Table 3 / Appendix B. The paper's own footnote 5 judges the Benchmark II leakage risk as "minimal given that the datasets bear no overlap beyond high-level conceptual categorization." The load-bearing caveat is the *Benchmark I* in-domain split, not Benchmark II: Chronos's Benchmark I includes `M4 (Daily/Hourly/Monthly/Weekly)`, `Electricity (15 Min/Hourly/Weekly)`, `ETT (15 Min/Hourly)`, `KDD Cup 2018`, `Temperature-Rain`, `Pedestrian Counts`, and `London Smart Meters` — which **directly overlap with GIFT-Eval's test split** (GIFT-Eval Table 13, arXiv:2410.10393). This is why [Moirai-MoE](./moirai-moe.md) Figure 3 asterisks Chronos-Small/Base/Large as non-zero-shot on Monash. Chronos v1's pretraining-only set also includes `Wiki Daily (100k)` — 100,001 Wikipedia articles daily — but GIFT-Eval test contains no Wikipedia-derived series (per GIFT-Eval Table 13), so that specific inclusion does not leak against GIFT-Eval test. See [../benchmarks/wikipedia-pageviews-leakage.md](../benchmarks/wikipedia-pageviews-leakage.md) and [../research/reproducibility.md](../research/reproducibility.md).

## Follow-up work and dialogue
[chronos-2](./chronos-2.md) is the direct successor: it drops the discrete vocabulary in favour of patch-valued tokens, adds group attention for [in-context learning](../concepts/in-context-learning.md) over related series, and explicitly targets the univariate/covariate gap the original paper left open. [moirai](./moirai.md) and [moirai-moe](./moirai-moe.md) pursue a different axis — a much larger and more diverse pretraining corpus ([LOTSA](../datasets-benchmarks/lotsa.md)) combined with frequency-aware or MoE routing — and argue the Chronos corpus is both too small and too weighted toward a few domains. On the architecture front, [timer](./timer.md), [timer-xl](./timer-xl.md), and [time-moe](./time-moe.md) keep the decoder-only patched approach of [timesfm](./timesfm.md) and treat Chronos' quantization as an inductive-bias dead-end; conversely [lag-llama](./lag-llama.md) keeps a continuous Student-t head, staking out the "parametric probabilistic" alternative to Chronos' categorical head. See [../concepts/value-quantization.md](../concepts/value-quantization.md) for the design tradeoff.

## Reproducibility
- **Open weights:** yes — T5-based checkpoints at 20M, 46M, 200M, 710M are released together with a GPT-2 90M variant.
- **Code:** public at `amazon-science/chronos-forecasting` (cited in the paper).
- **Training data:** partially public — 28 source datasets are public, and the HuggingFace dataset card redistributes them; TSMixup augmentations and KernelSynth synthetics are reproducible from the released code.
- **Compute to retrain:** 200K optimizer steps with AdamW (weight decay 0.01, peak learning rate 1e-3 linearly annealed to 0), effective batch size 256 sequences via distributed data parallel and gradient accumulation, on 8 A100 40GB GPUs per model (Chronos, Section 5). Training windows are slices of TSMixup-augmented series plus synthetic KernelSynth draws at a 9:1 ratio. Per-size wall-clock and USD cost are reported in Appendix Table 6; exact FLOPs are not disclosed. See [../research/training-recipes.md](../research/training-recipes.md) for the consolidated recipe table.
- **Deployment footprint:** 20M-710M parameters; context length 512, prediction length 64; inference demonstrated on standard GPUs, no CPU benchmark reported.

## When to cite this paper
Cite Chronos as the canonical reference for the "regression via classification on a fixed vocabulary of binned values" recipe and for TSMixup/KernelSynth. It is also the right citation for the first clear demonstration that an unmodified T5 architecture, with only a retuned vocabulary, can serve as a probabilistic TS foundation model; later papers should be preferred for covariate-aware or multivariate claims.

## In the knowledge graph
- **Cluster:** [Masked-encoder / encoder-decoder TS-FMs](../foundation-models/taxonomy.md#cluster-2--masked-encoder--encoder-decoder-ts-fms)
- **Architecture family:** [Encoder-decoder T5](../architectures/encoder-decoder-t5.md)
- **Related concepts:** [value quantization](../concepts/value-quantization.md), [probabilistic forecasting](../concepts/probabilistic-forecasting.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md), [synthetic data augmentation](../concepts/synthetic-data-augmentation.md)
- **See also:** [chronos-2](./chronos-2.md), [moirai](./moirai.md), [moment](./moment.md)
