# Contrastive Representation Learning for Time Series

A self-supervised pretraining family that learns a feature encoder by pulling representations of "related" time-series segments together and pushing "unrelated" ones apart, optimizing a discriminative loss (InfoNCE, triplet, or N-pair) instead of a reconstruction or forecasting objective.

## Intuition

A forecaster needs to predict the next value; a classifier needs a label; an anomaly detector needs to flag outliers. All three want the same thing under the hood: a representation `z_t = f(x_{1:t})` that captures the slow, semantically meaningful structure of the series and discards the local noise. Contrastive objectives bypass the reconstruction problem ("predict every detail of the next sample") in favor of a discrimination problem ("among these N candidates, which one really comes next?"). The discrimination problem is much easier to optimize because the loss is bounded and the model never has to model `p(x_{t+k})` directly — it only has to score it correctly relative to a few negatives. Maximizing this score is equivalent to maximizing a lower bound on the mutual information `I(x_{t+k}; c_t)` between the future sample and the current context, so the encoder is implicitly forced to retain whatever about the past is predictive of the future.

This idea reached time series first via Contrastive Predictive Coding ([CPC, van den Oord et al., 2018](../../papers/cpc_1807.03748.pdf)), which used a strided CNN encoder plus a GRU autoregressive aggregator and the InfoNCE loss to learn audio representations on LibriSpeech that linearly probed close to fully supervised models. The TS-specific follow-ups — T-Loss (Franceschi et al., 2019), TNC (Tonekaboni et al., 2021), TS-TCC (Eldele et al., 2021), and especially [TS2Vec](../../papers/ts2vec_2106.10466.pdf) (Yue et al., 2022) — reworked the positive-pair definition, the encoder backbone, and the contrastive granularity for non-stationary signals.

## Mechanics

The canonical setup has three pieces: an **encoder** `f_θ` that maps a window to a sequence of latent vectors, a **definition of positive pairs** (which two latent slots should be close), and a **contrastive loss** that scores the positive against a batch of negatives.

```
# CPC (van den Oord 2018):
z_t = encoder(x_t)               # strided CNN over raw waveform / series
c_t = GRU(z_{<=t})               # 256-dim hidden state, summarizes the past
# For each future step k, score the true z_{t+k} against negatives:
f_k(x_{t+k}, c_t) = exp(z_{t+k}^T W_k c_t)
L_NCE = -E[log f_k(x_{t+k}, c_t) / sum_j f_k(x_j, c_t)]   # one positive, N-1 negatives
```

```
# TS2Vec (Yue 2022): hierarchical, no autoregressive aggregator
crop two overlapping windows from x_i; mask latents with Bernoulli(0.5)
z, z' = dilated_CNN(crop1), dilated_CNN(crop2)   # 10 residual blocks
# at every timestamp t in the overlap region:
L_temp(i, t) = -log( exp(z_{i,t} . z'_{i,t}) / sum over other timestamps in overlap )
L_inst(i, t) = -log( exp(z_{i,t} . z'_{i,t}) / sum over other instances at the same t )
# repeat after maxpool by 2 along time, recurse to top, average:
L_hier = mean_over_scales(L_temp + L_inst)
```

The two patterns differ on three axes that turn out to matter. First, **what is "future"**: CPC predicts `k` steps ahead in latent space; TS2Vec instead aligns the *same* timestamp under two augmentations, sidestepping the question of how far to predict. Second, **what is "negative"**: CPC's negatives are sampled future windows (other timesteps, other speakers); TS2Vec's negatives are split into instance-wise (other series in the batch at the same `t`) and temporal (other timestamps in the same series), which the paper argues are complementary. Third, **what is the granularity**: CPC produces one context vector per window; TS2Vec produces a representation *per timestamp*, max-pools to coarser scales, and contrasts at every level of a binary hierarchy. The hierarchical version is what lets a single TS2Vec encoder serve forecasting, classification, and anomaly detection without retraining.

## Why it works

InfoNCE is a tractable lower bound on mutual information: `I(x_{t+k}; c_t) ≥ log N − L_NCE`. Optimizing the loss therefore forces the encoder to keep whatever about the past is informative about the future and discard whatever is not. Because the loss is bounded above by `log N` and below by zero, gradients stay well-behaved even at the start of training — a major practical advantage over reconstruction losses on heavy-tailed time series, where MSE on raw values gives the optimizer almost no useful signal. The contrastive setup also avoids two failure modes specific to TS reconstruction: (a) collapse to predicting the local mean (because that minimizes MSE on stationary regions), and (b) wasted capacity on high-frequency noise (because MSE penalizes mismatches in the noise band as much as in the signal band).

For TS specifically, the choice of *positive pair* encodes the inductive bias. Subseries consistency (T-Loss) assumes a series and its subsample share semantics — broken by level shifts. Temporal consistency (TNC) assumes adjacent windows share semantics — broken by anomalies. Contextual consistency (TS2Vec) assumes the same timestamp under two augmentations should map to the same representation — robust to both, at the cost of needing an augmentation that does not change the series magnitude.

## Trade-offs and failure modes

- **Negative-sample distribution matters.** If all negatives come from the same speaker / series / domain, the encoder over-fits to whatever varies across that group. The CPC ablation (Table 2) shows phone classification accuracy swings from 57.3% to 65.5% depending on whether negatives are mixed-speaker, same-speaker, or restricted to the current sequence.
- **No probabilistic output for free.** Contrastive pretraining gives a representation, not a forecast. To use it for probabilistic forecasting you have to graft a head on top — typically a quantile regression or a Gaussian likelihood — which is a separate training pass.
- **Augmentations are domain-specific and hand-engineered.** Cropping and masking work for many TS, but transformations like jittering, scaling, or permutation (TS-TCC) embed inductive biases that are sometimes harmful (TS2Vec §1 critiques exactly this).
- **Compute for negatives scales with batch size.** InfoNCE wants many negatives per positive; small batches under-train the contrastive head. This is one reason TS2Vec is much faster than TNC despite a larger encoder — the positive-pair scheme does not need a separate discriminator and can amortize across the batch.
- **Doesn't beat TS-FMs at forecasting in 2026.** Pretrained TS-FMs ([Chronos](../papers/chronos.md), [MOIRAI](../papers/moirai.md), [Time-MoE](../papers/time-moe.md)) have eaten the zero-shot forecasting market that contrastive methods used to occupy. Contrastive is still competitive on classification (UCR/UEA) and anomaly detection where the downstream task does not match the TS-FM pretraining objective.

## Design choices in the literature

- **CPC** ([1807.03748](../../papers/cpc_1807.03748.pdf), DeepMind 2018) — strided CNN encoder + GRU aggregator + InfoNCE; 64.6% LibriSpeech phone accuracy from a linear probe vs. 74.6% supervised. Multi-domain (audio, vision, text, RL) but the audio setup is the prototype TS application.
- **T-Loss** (Franceschi, Dieuleveut, Jaggi, NeurIPS 2019) — dilated causal CNN encoder + triplet loss on subseries consistency. TS-specific; the first widely cited unsupervised TS encoder. CNN, not RNN.
- **TNC** (Tonekaboni, Eytan, Goldenberg, ICLR 2021) — bidirectional RNN encoder; defines positive pairs as samples from a *temporal neighborhood* with a stationarity test, uses a debiased contrastive objective. Motivated by clinical signals where the latent state changes slowly.
- **TS-TCC** (Eldele et al., IJCAI 2021) — CNN encoder; instance contrast across two augmentations (strong and weak), plus a temporal contrast inside a transformer. Introduced the augmentation-pair recipe that TS2Vec then critiqued.
- **TS2Vec** ([2106.10466](../../papers/ts2vec_2106.10466.pdf), AAAI 2022) — 10-block dilated CNN encoder, contextual consistency under timestamp masking and random cropping, hierarchical contrast (instance-wise + temporal, recursive max-pool). +2.4% over T-Loss on UCR-125 and +3.0% on UEA-29 with 40× shorter training.
- Inside the TS-FM literature, contrastive pretraining is a minority track. The TS-FMs in this wiki mostly use masked reconstruction ([MOMENT](../papers/moment.md), [MOIRAI](../papers/moirai.md), [TSPulse](../papers/tspulse.md)), tokenized cross-entropy ([Chronos](../papers/chronos.md), [TOTEM](../papers/totem.md)), or autoregressive next-patch loss ([TimesFM](../papers/timesfm.md), [Timer](../papers/timer.md), [Sundial](../papers/sundial.md)). [UniTS](../papers/units.md) is the closest to a contrastive flavor in this wiki — it uses prompt-token alignment across tasks — but it is supervised on labels.

## Open questions

- Is contrastive pretraining complementary to or substitutable for masked reconstruction in a billion-scale TS-FM? No paper in the current 23-leaf cohort runs a clean head-to-head ablation.
- Can the InfoNCE bound be tightened with a learned negative-sampling distribution that adapts per-domain? CPC's ablation hints at large headroom but no follow-up has been published in the TS literature.
- For multivariate panels, should the contrastive loss operate per channel (instance contrast) or jointly across channels (cross-variate contrast)? TS2Vec defaults to per-instance; the channel-independence trade-off is open.
- What is the minimum batch size at which InfoNCE-style losses still give competitive representations on long-context (`T > 4096`) TS? Audio CPC uses 8 GPUs × 8 examples per minibatch; TS analogues are smaller, and the floor is uncharted.
- Do contrastive representations transfer zero-shot across *frequencies* the way TS-FMs do, or only across *datasets at one frequency*?

## Papers that exemplify this

- **[CPC](../../papers/cpc_1807.03748.pdf)** — the foundational sequential-contrastive recipe; explicit GRU aggregator + InfoNCE, the architecture the user description "GRU + contrastive loss" most directly maps to.
- **[TS2Vec](../../papers/ts2vec_2106.10466.pdf)** — top-cited TS-specific contrastive paper, hierarchical and timestamp-level; the standard reference for representation-learning baselines in TS classification leaderboards (UCR/UEA).
- TNC, T-Loss, TS-TCC — the three TS-specific predecessors that TS2Vec critiques and outperforms; cited here for completeness, not yet ingested as paper leaves.
- [TOTEM](../papers/totem.md) — uses a VQ-VAE codebook rather than a contrastive loss, but addresses the same underlying problem (one universal encoder for many downstream tasks). Useful contrast point.

## Related wiki pages

- [zero-shot forecasting](zero-shot-forecasting.md) — what large TS-FMs use *instead* of contrastive pretraining for the same generalization goal.
- [multi-task universal](multi-task-universal.md) — the downstream-task setup that contrastive representations naturally support.
- [value quantization](value-quantization.md) — alternative pretraining target that addresses the same "what should the model predict" question.
- [patch tokenization](patch-tokenization.md) — tokenization choice; CPC uses strided-CNN downsampling, TS2Vec uses an input projection + dilated CNN, neither uses patches in the PatchTST sense.
- [synthetic data augmentation](synthetic-data-augmentation.md) — the augmentation question contrastive learning depends on most directly.
- [masked encoder](../architectures/masked-encoder.md) — the reconstruction-based alternative to contrastive pretraining (MOMENT, MOIRAI).
- [lightweight non-transformer](../architectures/lightweight-non-transformer.md) — same architectural bracket as TS2Vec's dilated-CNN encoder.
- [foundations/deep-learning-era](../foundations/deep-learning-era.md) — broader context on the pre-TS-FM neural era.
