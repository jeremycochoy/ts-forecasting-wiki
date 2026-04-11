# TOTEM: TOkenized Time Series EMbeddings

> **Short name:** `totem` · **arXiv:** [2402.16412](https://arxiv.org/abs/2402.16412) · **PDF:** [local](../../papers/totem_2402.16412.pdf) · **Date:** 2024-02 · **Venue:** TMLR 12/2024

**Authors:** Sabera Talukder, Yisong Yue, Georgia Gkioxari (Caltech)

## Abstract
TOTEM learns a cross-domain discrete time-series codebook via a VQ-VAE and evaluates both specialist and generalist models on top of it at scale. It argues that discrete-token TS pretraining is a principled alternative to raw continuous patches and shows the same codebook supports imputation, anomaly detection, and forecasting across many domains.

## Key contributions
- VQ-VAE-based cross-domain codebook for time-series tokens, with a 1D strided convolutional encoder / decoder and exclusively temporal (not spatial) quantisation.
- Systematic comparison of specialist (one domain) versus generalist (many domains) training on the shared codebook, across 3 tasks, with 14-19 baselines per task and up to 25 evaluation datasets.
- Demonstration that the learned codebook transfers zero-shot to held-out domains (neuroscience, river flow, sunspots, birth rate) without fine-tuning the tokenizer.
- Case against hand-engineered Fourier features and domain-specific preprocessing: only RevIN is used for normalisation, no frequency-space tricks.

## Architecture at a glance
TOTEM trains a convolutional VQ-VAE whose codebook provides a discrete vocabulary for time-series patches. The encoder is a stack of strided 1D convolutions with dilation 1 so every time step contributes; the decoder mirrors it with transposed convolutions. Downstream transformers then consume the resulting token sequences for each task. The tokenizer is frozen after pretraining and reused across specialists and generalists.

## Why it matters
TOTEM anchors the discrete-tokenisation branch of TS foundation models. By quantifying specialist-versus-generalist trade-offs with a common codebook, it motivates subsequent approaches that combine VQ-style tokenisation with universal pretraining objectives and provides the cleanest evidence that a learned (rather than uniformly binned) discrete vocabulary is viable across many TS domains.

## Strengths
- The evaluation is unusually thorough: ~500 experiments covering imputation (17 baselines, 12 datasets), anomaly detection (19 baselines, 25 datasets), and forecasting (14 baselines, 12 datasets), in both specialist and generalist settings.
- Zero-shot evaluation is on held-out domains with no shared sampling rate or sensor count with the training data (neuro 2 ms, river flow daily, birth rate daily), which is a stricter test than most TS zero-shot papers run.
- Motivates the learned codebook over uniform bucketing or histogram binning by arguing that a deterministic, reversible encoder is more information-preserving than scalar quantisation.
- Shows that the same architecture suffices for all three tasks using simple, generic downstream models, supporting the claim that the codebook itself carries most of the transferable signal.
- The generalist VQ-VAE outperforms the concurrent MOMENT model on the limited subset of shared benchmarks.

## Limitations and open critiques
- Accuracy is competitive but rarely dominant: TOTEM "matches or outperforms" SOTAs rather than decisively surpassing them, and the headline MSE improvements are single-digit percentages.
- VQ-VAE codebook collapse and under-utilisation are known pathologies that the paper does not deeply analyse; hyperparameters that prevent collapse in vision may not transfer to noisy TS.
- Sundial argues that all discrete-tokenisation approaches (TOTEM, Chronos, LLMTime) throw away structure that flow-matching on continuous values preserves, and reports substantial point-forecast gains from dropping discretisation.
- The codebook is trained separately from downstream tasks, so the tokenizer cannot adapt its vocabulary to the forecasting or classification objective at hand; Chronos sidesteps this by training the T5 and the vocabulary end-to-end.
- Model sizes and training compute are not benchmarked against recent billion-parameter TS-FMs, so TOTEM's position on the accuracy-efficiency Pareto is unclear.

## Follow-up work and dialogue
TOTEM and [Chronos](./chronos.md) are the two canonical answers to "how to discretise time series": TOTEM uses a learned VQ-VAE codebook on top of raw values, while Chronos uses scalar uniform quantisation (min-max binning) into a fixed vocabulary consumed by a T5. Chronos argues simpler is better when paired with enough data; TOTEM argues the representation should be learned end-to-end. [LLMTime](./llmtime.md) is a third data point — it uses a text LLM's existing BPE over digit strings, neither uniform nor learned. [Sundial](./sundial.md) rejects all three by going continuous via flow-matching. TOTEM also sits adjacent to [UniTS](./units.md) and [MOMENT](./moment.md) in the multi-task/universal cluster.

## Reproducibility
- **Open weights:** yes — the paper states pretrained codebooks and downstream models are released.
- **Code:** public repo stated as `https://github.com/SaberaTalukder/TOTEM`.
- **Training data:** public corpora only (Weather, Traffic, Electricity, ETT variants for forecasting/imputation; SMD, MSL, SMAP, SWaT, PSM for anomaly detection). Zero-shot held-out domains enumerated in the appendix.
- **Compute to retrain:** not disclosed as GPU-hours; the paper emphasises minimal engineering and standard VQ-VAE training.
- **Deployment footprint:** codebook plus small convolutional encoder/decoder; downstream transformers are "simple, generic"; no CPU-inference numbers reported.

## When to cite this paper
Cite TOTEM as the canonical reference for a learned VQ-VAE codebook as the tokeniser of a TS foundation model, for the systematic specialist-versus-generalist comparison at scale, and as the direct point of comparison to Chronos whenever the question is learned versus uniform discretisation.

## In the knowledge graph
- **Cluster:** [Multi-task / universal unified TS models](../foundation-models/taxonomy.md#cluster-6--multi-task--universal-unified-ts-models)
- **Architecture family:** [Masked encoder](../architectures/masked-encoder.md)
- **Related concepts:** [value quantization](../concepts/value-quantization.md), [patch tokenization](../concepts/patch-tokenization.md), [multi-task universal](../concepts/multi-task-universal.md), [revin normalization](../concepts/revin-normalization.md)
- **See also:** [chronos](./chronos.md), [units](./units.md), [moment](./moment.md), [llmtime](./llmtime.md), [sundial](./sundial.md)
