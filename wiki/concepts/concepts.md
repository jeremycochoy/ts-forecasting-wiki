# Concepts

This section collects the recurring design ideas and training recipes that cut across time-series foundation models and the broader representation-learning literature they build on. Most concern how raw numeric series are turned into something a transformer can consume (patch tokenization, value quantization, RevIN), how the model expresses uncertainty (probabilistic forecasting), how it is evaluated without task-specific fine-tuning (zero-shot, in-context learning), how it is scaled (scaling laws, synthetic data, multi-task heads), and how feature representations are pretrained without a forecasting objective (contrastive representation learning).

## Sub-pages

- [Patch tokenization](patch-tokenization.md) — contiguous windows as transformer tokens.
- [Value quantization](value-quantization.md) — mapping real values to discrete bins or VQ codes.
- [Zero-shot forecasting](zero-shot-forecasting.md) — inference on unseen series without gradient updates.
- [Probabilistic forecasting](probabilistic-forecasting.md) — predictive distributions and quantiles, not point estimates.
- [In-context learning](in-context-learning.md) — conditioning on related series or demonstrations at inference time.
- [Scaling laws](scaling-laws.md) — loss-vs-compute/data/params curves for TS.
- [Data normalization](data-normalization.md) — the full normalization landscape: RevIN, mean-scaling, partial-window, raw values, and why nobody differences.
- [RevIN normalization](revin-normalization.md) — deep dive on RevIN mechanics and failure modes.
- [Multi-task universal](multi-task-universal.md) — one model for forecast, classify, impute, anomaly.
- [Synthetic data augmentation](synthetic-data-augmentation.md) — KernelSynth, TSMix, PFN priors, and friends.
- [HP transfer across scales](hp-transfer-across-scales.md) — muP, poor man's muP, cross-shaped proxy search, and architectural stabilizers for tuning HPs once at small size and scaling up.
- [Contrastive representation learning](contrastive-representation-learning.md) — InfoNCE-style pretraining via positive/negative discrimination (CPC, TS2Vec, TNC, T-Loss); the representation-learning track that runs parallel to forecasting-only TS-FMs.
