# Representation Learning with Contrastive Predictive Coding

> **Short name:** `cpc` · **arXiv:** [1807.03748](https://arxiv.org/abs/1807.03748) · **PDF:** [local](../../papers/cpc_1807.03748.pdf) · **Date:** 2018-07 · **Venue:** preprint (NeurIPS rejected; widely cited)

**Authors:** Aaron van den Oord, Yazhe Li, Oriol Vinyals (DeepMind)

## Abstract
Contrastive Predictive Coding (CPC) proposes a single recipe for unsupervised representation learning that works across speech, vision, text, and reinforcement learning. The key move is to predict the future *in latent space* using a powerful autoregressive context, while training the encoder and the autoregressor jointly with a probabilistic contrastive loss (InfoNCE) that classifies the true future sample against negatives drawn from a proposal distribution. Optimizing this loss is equivalent to maximizing a lower bound on the mutual information `I(x_{t+k}; c_t)`, so the encoder is forced to retain whatever about the past is predictive of the future and discard local noise.

## Key contributions
- **InfoNCE loss.** A categorical-cross-entropy formulation of noise-contrastive estimation in which the model classifies the positive sample among `N-1` negatives; the optimum recovers the density ratio `p(x_{t+k}|c_t) / p(x_{t+k})` and the loss is a tractable lower bound on mutual information (Section 2.3, `I(x_{t+k}, c_t) ≥ log(N) − L_N`).
- **Predict-in-latent-space framing.** Rather than reconstruct the high-dimensional future `x_{t+k}` (which wastes capacity on per-sample detail), the model only has to score an *encoded* future `z_{t+k}` against negatives. This sidesteps the unimodal-loss limitations of MSE / cross-entropy on raw signals.
- **Encoder + aggregator separation.** A non-linear encoder `g_enc` produces frame-level latents `z_t`; a separate autoregressor `g_ar` summarizes `z_{≤t}` into a context `c_t`. Either can serve downstream tasks, with `c_t` preferred when long context matters and `z_t` preferred when local features suffice.
- **Multi-domain validation.** A single recipe transfers to four modalities — LibriSpeech audio, ImageNet image patches, BookCorpus text, and DeepMind Lab RL — using essentially the same loss with domain-specific encoders.
- **Strong audio probe.** On LibriSpeech, a linear classifier on top of `c_t` reaches 64.6% phone accuracy and 97.4% speaker accuracy versus 74.6% / 98.5% supervised (Table 1).

## Architecture at a glance
The audio prototype uses a five-layer strided 1D CNN encoder over the raw 16 kHz PCM waveform, with strides `[5, 4, 2, 2, 2]`, kernels `[10, 8, 4, 4, 4]`, 512 ReLU units, and a total downsampling of 160 — one latent every 10 ms (Section 3.1). The autoregressor is a single GRU with 256-dim hidden state. For each step `k ∈ {1, …, 12}`, the score is the log-bilinear `f_k(x_{t+k}, c_t) = exp(z_{t+k}^T W_k c_t)` with a separate `W_k` per offset. Training is end-to-end on InfoNCE with negatives drawn from other timesteps in the same minibatch.

## Why it matters
CPC is the first general-purpose contrastive *sequence* model: the InfoNCE loss it introduces became the de facto standard for contrastive self-supervision in vision (SimCLR, MoCo), audio (wav2vec 2.0), and language (sentence embeddings). The "GRU + InfoNCE" combination is the textbook reference for contrastive representation learning with a recurrent aggregator, and it is the canonical citation for the broader idea of predicting the future in latent space rather than reconstructing it.

## Strengths
- **Linear-probe parity on audio.** LibriSpeech 100h phone classification reaches 64.6% with a linear classifier on `c_t` versus 74.6% fully supervised (Table 1), and a single hidden layer closes the gap to 72.5% (Section 3.1). Speaker classification on the same features hits 97.4% / 98.5%, evidence that one representation captures both phonetic content and speaker identity.
- **Ablation evidence for multi-step prediction.** Predicting 12 steps ahead beats 2 / 4 / 8 / 16 steps (Table 2: 64.6% vs 28.5% / 57.6% / 63.6% / 63.8%), showing that long-horizon contrastive prediction is what forces the encoder to capture slow features.
- **Negative-sample structure matters and is testable.** Mixed-speaker, same-speaker, current-sequence-only, and excluded-current-sequence variants give 64.6% / 65.5% / 65.2% / 57.3% respectively (Table 2), giving a calibrated signal of how the negative distribution shapes the learned representation.
- **Cross-domain transfer.** ImageNet linear classification on CPC features improves over the prior unsupervised state of the art by 9% absolute top-1 and 4% absolute top-5 (Section 3.2), with a ResNet-v2-101 encoder and a PixelCNN autoregressor over a 7×7 grid of 64×64 patches.

## Limitations and open critiques
- **Audio benchmark is narrow.** The TS-relevant evaluation is the 100-hour LibriSpeech subset with phone / speaker probes — speech, not arbitrary time series — and there is no native handling of non-stationarity, level shifts, or heterogeneous frequencies that real TS pretraining must absorb.
- **Aggregator is modest by 2026 standards.** A 256-dim GRU was strong in 2018; current TS encoders are dilated CNNs ([TS2Vec](./ts2vec.md)) or transformers, and the GRU's vanishing-gradient behavior on long context is now well understood as a bottleneck.
- **Hand-engineered negatives.** The loss accuracy depends sharply on whether negatives come from the same speaker, mixed speakers, or the same sequence (Table 2); the paper does not propose a learned or adaptive negative-sampling scheme, leaving a real axis of variance to follow-ups.
- **No probabilistic forecast head.** CPC produces a representation, not a distribution over future values; downstream forecasting requires grafting a regression / quantile head — see the [contrastive representation learning](../concepts/contrastive-representation-learning.md) trade-offs section.
- **Multi-domain story is impressive but the audio model alone is what TS readers cite; the paper does not ship a TS checkpoint or a TS-specific evaluation.**

## Follow-up work and dialogue
CPC seeded the entire contrastive-SSL line in TS. [TS2Vec](./ts2vec.md) explicitly critiques CPC's "predict the future in latent space" framing as insufficient for non-stationary TS — a single context vector cannot represent multiple semantic scales — and replaces the GRU aggregator and per-step prediction with a hierarchical, timestamp-level contextual-consistency scheme that contrasts within and across instances at every pooling depth. T-Loss (Franceschi 2019), TNC (Tonekaboni 2021), and TS-TCC (Eldele 2021) are intermediate TS-specific descendants surveyed in [contrastive representation learning](../concepts/contrastive-representation-learning.md). On a different axis, the JEPA family — and specifically [TS-JEPA](./ts-jepa.md) — keeps CPC's predict-in-latent-space idea but drops InfoNCE in favor of an L1 latent loss against an EMA target, removing the need for negative samples; [LaT-PFN](./lat-pfn.md) is a PFN+JEPA descendant that inherits the same lineage. SimCLR, MoCo, and wav2vec 2.0 inherit InfoNCE itself.

## Reproducibility
- **Open artifacts:** —
- **Code:** community implementations only (e.g. `Spijkervet/contrastive-predictive-coding`); DeepMind did not release a reference repo.
- **Data:** LibriSpeech 100h subset (public; force-aligned phone labels and the train/test split released by the authors via Google Drive, see Section 3.1 footnote 2), ImageNet ILSVRC (public), BookCorpus (no longer publicly distributed), DeepMind Lab (public).
- **Compute:** audio model trained on 8 GPUs, minibatch 8 per GPU, Adam at `2e-4`, ~300K updates to convergence (Section 3.1). ImageNet model trained on 32 GPUs with batch 16 (Section 3.2).
- **Deployment footprint:** small (encoder 512-dim, GRU 256-dim for audio), but no shipped TS checkpoint — the recipe is the deliverable, not a model.

## When to cite this paper
Cite CPC as the canonical reference for **InfoNCE** and for the broader idea of **predicting the future in latent space with a probabilistic contrastive loss**. It is also the right citation for "GRU aggregator + contrastive loss for sequential representation learning" and as the foundational SSL paper that mutual-information-based pretraining inherits from. For TS-specific contrastive work prefer [TS2Vec](./ts2vec.md); for the JEPA descendant prefer [TS-JEPA](./ts-jepa.md).

## In the knowledge graph
- **Cluster:** Pre-FM representation learning (multi-domain foundational; not part of the eight-cluster TS-FM taxonomy)
- **Concept hub:** [contrastive representation learning](../concepts/contrastive-representation-learning.md)
- **Related concepts:** [contrastive representation learning](../concepts/contrastive-representation-learning.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md) (failure-mode counterexample — CPC produces representations, not zero-shot forecasts), [joint-embedding predictive architecture](../concepts/joint-embedding-predictive-architecture.md) (close cousin in non-reconstructive SSL), [synthetic data augmentation](../concepts/synthetic-data-augmentation.md) (related role for positive-pair construction)
- **See also:** [TS2Vec](./ts2vec.md), [TS-JEPA](./ts-jepa.md) (cites CPC), [LaT-PFN](./lat-pfn.md) (PFN+JEPA descendant)

## Related wiki pages
- [contrastive representation learning](../concepts/contrastive-representation-learning.md)
- [foundations/deep-learning-era](../foundations/deep-learning-era.md)
