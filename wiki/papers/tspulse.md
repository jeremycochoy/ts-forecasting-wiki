# TSPulse: Tiny Pre-Trained Models with Disentangled Representations for Rapid Time-Series Analysis

> **Short name:** `tspulse` · **arXiv:** [2505.13033](https://arxiv.org/abs/2505.13033) · **PDF:** [local](../../papers/tspulse_2505.13033.pdf) · **Date:** 2026-03 · **Venue:** ICLR 2026

**Authors:** Vijay Ekambaram, Subodh Kumar, Arindam Jati et al. (IBM Research — Granite team)

## Abstract
TSPulse is a family of ultra-light (~1M parameter) TS pre-trained models built on a TSMixer backbone that learns jointly in time and frequency domains and explicitly disentangles its embedding into temporal, spectral and semantic views. It targets four diagnostic tasks — classification, imputation, anomaly detection, similarity search — rather than forecasting. Across TSB-AD, UEA, LTSF imputation and a custom retrieval benchmark the paper reports +20% (anomaly), +5–16% (classification), +50% (imputation) and +25% (similarity) over MOMENT, UniTS, VQShape, Chronos and TimesFM while being 10–100× smaller.

## Key contributions
- **Disentangled dual-space masked reconstruction.** A single pre-training objective produces three embedding segments — `TimeE`, `FFTE`, and register-token `RegE` — each routed through a different head: time-MSE, frequency-MSE, and a softmax cross-entropy "semantic signature" over the log-magnitude spectrum (plus an optional next-point MSE).
- **Hybrid masking for realistic missingness.** Unlike MOMENT/UniTS which use fixed block masking in the embedding space, TSPulse masks at the raw patch level with a learnable token supporting both full and partial patches with randomized span length and ratio — a direct response to the block-mask bias the paper documents in prior work.
- **Register tokens as semantic embeddings.** Following Darcet et al.'s ViT-registers, `R` learnable tokens are concatenated to the time and FFT patch sequences. Sensitivity analysis (Table 2) shows they are highly robust to missingness, noise and phase shift (12% phase distortion vs. 130% for `TimeE`), justifying their use as the retrieval view.
- **TSMixer backbone, not a transformer.** The backbone is 8 stacked TSMixer blocks with softmax-gated attention, explicitly motivated by TSMixer/TTM efficiency. A mini-decoder at 10–20% of backbone size mirrors the backbone and is where channel mixing is re-enabled at fine-tune time, identity-initialized for stable gradients.
- **Post-hoc fusers.** `TSLens` for classification (attends over disentangled views rather than pooling) and `Multi-Head Triangulation` for anomaly detection (fuses `Headtime`, `Headfft`, `Headpred` and their ensemble, picking the best head on a small tuning set).
- **Task-reweighted pre-training.** Each task gets a specialized checkpoint by reweighting reconstruction losses; Appendix A.15 shows a single unified checkpoint is still top-two on every task.

## Architecture at a glance
Input `X ∈ R^{S×C}` with `S=512`, patch `pl=8`, hidden `D=3·pl=24`. The input is RevIN-normalized and hybrid-masked, then an `rfft` produces a masked frequency tensor with the mask propagated through the transform (so there is no separate frequency mask). Time and FFT patches are linearly projected and concatenated with `R` learnable register tokens, giving `C × (2N+R) × D`. The backbone is an 8-layer TSMixer stack with gated attention, mixing within-patch / across-patch / across-channel; a 2-layer mini-decoder follows. Multi-output heads split the decoder output into three segments and optimise `L_time1 + L_time2 + L_fft + L_sign (+ L_pred)`. The model totals ~1.06M parameters (Table 3). Pretraining is channel-independent; channel mixing is re-introduced only at fine-tune time, identity-initialized. Input patch size is 8 (with S=512, giving 64 patches); TSPulse is a dual-space masked reconstruction model and is NOT a forecaster by design — the `Headpred` head is limited to a few points for embedding cues only (Ekambaram et al. Section 2).

Two design rationales beyond the inventory are explicit in paper §2. (a) **Hybrid patch+point masking** is a direct response to the *block-mask bias* the paper documents in MOMENT and UniTS, which apply fixed block masking in embedding space — a setup that over-fits to large contiguous gaps that don't reflect real missingness; TSPulse instead masks at the raw patch level with a learnable token and randomized span / ratio, evaluated under Rubin's MAR / MNAR taxonomy in Appendix A.16. (b) **Register tokens as the semantic-embedding view** follows ViT-Registers (Darcet et al.); paper Table 2 sensitivity analysis shows register embeddings are highly robust to missingness, noise, and phase shift (12% phase distortion vs. 130% for `TimeE`), which is what makes them the stable substrate for retrieval and similarity search rather than a temporary architectural curiosity.

## Why it matters
TSPulse is IBM Research's entry into multi-task TS analysis, combining sub-1M parameter count, a dual-space time+frequency masked objective, and explicit abstraction-level disentanglement. It is the analysis-side complement of [TTM](./ttm.md) — same lab, same TSMixer backbone, same Monash+LibCity corpus — and argues, against [MOMENT](./moment.md) and [UniTS](./units.md), that the right move for multi-task TS is not a larger encoder but better structured representations at the 1M scale.

## Strengths
- Consistent head-to-head wins: Figure 4 on TSB-AD (uni + multi), Figure 5 on 29 UEA datasets, Figure 6 on six LTSF imputation datasets, and Figure 7 on similarity search all place TSPulse above MOMENT, UniTS, VQShape, Chronos and TimesFM at 1M vs. 40–700M parameters.
- Table 3 quantifies the efficiency gap in wall-clock terms: 0.39 ms GPU / 60 ms CPU per batch vs. MOMENT-large at 405 ms / 22 s and Chronos-large at 898 ms / 1755 s — CPU-friendliness with an explicit order-of-magnitude breakdown.
- Disentanglement is empirically demonstrated via §6 / Appendix A.3 synthetic perturbation experiments, not just asserted: `TimeE`, `FFTE` and register embeddings respond differently to masking, noise and phase shift in the way the design predicts.
- Ablations (Table 1) attribute gains to specific components: removing dual-space learning costs 7% on classification and 8% on imputation MSE; removing hybrid masking collapses imputation under hybrid-mask evaluation by 79%.
- Pretraining is cheap: 1B samples in ~1 day on 8×A100 — within academic reach, unlike [Timer-S1](./timer-s1.md) or [time-moe](./time-moe.md).
- Robustness is evaluated under Rubin's MAR / MNAR missingness taxonomy (Appendix A.16), not just uniform block/point masks, which is rare in the TS-FM literature.

## Limitations and open critiques
- TSPulse is explicitly not a forecaster. The `Headpred` head is limited to a few points and only injects temporal cues into the semantic embedding; the paper reports no MASE / CRPS on GIFT-Eval or TSLib and the Limitations section flags forecasting as future work.
- Context length is hard-coded at `S=512` with `pl=8`, identical to MOMENT-small. Longer contexts, variable patch sizes, and multi-resolution support (as in [moirai](./moirai.md) or [timer-xl](./timer-xl.md)) are not addressed.
- Each downstream task has its own specialized checkpoint obtained by reweighting pre-training losses; TSPulse is multi-task in the sense of a shared architecture, not a single weights file serving all four tasks the way [units](./units.md) advertises.
- Channel mixing is deferred to fine-tuning, so strict multivariate zero-shot is limited beyond univariate-stacked inputs — the same critique TTM earned.
- Classification is reported as mean accuracy across 29 UEA datasets; per-dataset wins are pushed to the appendix.
- Similarity search is evaluated on a custom augmented benchmark built by the authors over UCR plus synthetic data; no independent retrieval benchmark is reported.
- No pre-training loss / scaling curve is published: parameter count is a design choice, not an empirical optimum.

## Follow-up work and dialogue
TSPulse sits at the intersection of two lineages. Against [TTM](./ttm.md) it is the same IBM Granite team with the same TSMixer backbone and the same Monash+LibCity ~1B-sample corpus, but with diagnostic heads instead of a forecasting head — effectively "TTM for analysis". Against [MOMENT](./moment.md) it is the 1M-parameter answer to a 40–385M T5-encoder: both target multi-task analysis with masked reconstruction, but MOMENT uses a transformer with a single entangled embedding while TSPulse uses TSMixer with explicit disentanglement; MOMENT is the intellectual predecessor TSPulse most directly rebuts on the "larger encoder is not the right axis" claim. Against [UniTS](./units.md), UniTS uses task tokens to condition a shared transformer backbone while TSPulse uses disentangled heads per representation type — two different architectures of "one backbone, many tasks". Against [TOTEM](./totem.md) it is the continuous-reconstruction counterpart: TOTEM relies on VQ-VAE discrete tokens, TSPulse on masked time+frequency reconstruction in continuous space. Together with Mamba4Cast and TTM in the lightweight / non-transformer cluster, TSPulse extends the "1M-parameter TSMixer is enough" argument from forecasting to classification, anomaly detection, imputation and retrieval.

## Reproducibility
- **Open weights:** yes — `ibm-granite/granite-timeseries-tspulse-r1` on Hugging Face.
- **Code:** public as part of IBM's `granite-tsfm` project (the same repository that hosts TTM).
- **Training data:** fully public — the ~1B-sample subset of [Monash](../datasets-benchmarks/monash-archive.md) and LibCity inherited from TTM, with Diverse Resolution Sampling applied (Table 10), disjoint from the evaluation sets.
- **Compute to retrain:** ~1 day on 8×A100 GPUs, 20 epochs. Not reported as GPU-hours / FLOPs.
- **Deployment footprint:** 1.06M parameters (Table 3), `S=512`, `pl=8`, `D=24`, 8 backbone layers, 2 decoder layers; 0.06 s CPU per batch, 0.39 GB peak memory — explicitly targets GPU-free deployment.

## When to cite this paper
Cite TSPulse as the canonical reference for a sub-1M-parameter multi-task TS analysis model, for explicit disentanglement of temporal, spectral and semantic views under a dual-space masked reconstruction objective, for hybrid patch+point masking as a fix to block-mask bias in imputation pretraining, and as the obligatory comparison point whenever someone argues a 40M+ transformer encoder is required for multi-task TS analysis. It is the IBM Granite team's analysis-side companion to [ttm](./ttm.md).

## In the knowledge graph
- **Cluster:** [Cluster 6 — Multi-task / universal unified TS models](../foundation-models/taxonomy.md#cluster-6--multi-task--universal-unified-ts-models) (primary), with strong secondary membership in [Cluster 5 — Lightweight / non-transformer FMs](../foundation-models/taxonomy.md#cluster-5--lightweight--non-transformer-fms)
- **Architecture family:** [Lightweight non-transformer](../architectures/lightweight-non-transformer.md) — the backbone is a stack of 8 TSMixer blocks with gated attention, not a transformer
- **Related concepts:** [multi-task universal](../concepts/multi-task-universal.md), [patch tokenization](../concepts/patch-tokenization.md), [revin normalization](../concepts/revin-normalization.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md), [synthetic data augmentation](../concepts/synthetic-data-augmentation.md)
- **Dataset / corpus:** [Monash Archive](../datasets-benchmarks/monash-archive.md) (subset) plus LibCity; ~1B samples, same mixture as TTM
- **See also:** [ttm](./ttm.md), [moment](./moment.md), [units](./units.md), [totem](./totem.md)
