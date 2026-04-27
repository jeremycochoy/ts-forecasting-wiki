# Moirai 2.0: When Less Is More for Time Series Forecasting

> **Short name:** `moirai-2` · **arXiv:** [2511.11698](https://arxiv.org/abs/2511.11698) · **PDF:** [local](../../papers/moirai-2_2511.11698.pdf) · **Date:** 2026-02 · **Venue:** preprint (v3, 2026-02-03)

**Authors:** Chenghao Liu, Taha Aksu, Juncheng Liu, Xu Liu et al. (Salesforce AI Research; corresponding author Junnan Li)

## Abstract
Moirai 2.0 is a decoder-only time-series foundation model trained on a new 36M-series / ~295B-observation corpus. It replaces three defining choices of [MOIRAI](./moirai.md) 1.0 — masked-encoder pretraining, multi-patch input projection, and a mixture-of-distributions head — with a decoder-only backbone, a single patch size, and a pinball-loss quantile head, and adds multi-token prediction. On the [GIFT-Eval](../datasets-benchmarks/gift-eval.md) leaderboard the 11.4M small variant ranks 5th by MASE and 6th by CRPS among pretrained foundation models while being ~30x smaller and ~2x faster than MOIRAI-1-Large; scaling to base (87M) or large (305M) monotonically hurts performance on the same benchmark.

## Key contributions
- Pivot from masked encoder to causal decoder-only backbone, computing `T-1` losses per `T`-patch sequence instead of the ~15% used under MOIRAI-1's masking ratio.
- Replacement of the MOIRAI-1 multi-patch-size projection with a single patch size, which the ablations show improves both efficiency and accuracy.
- Quantile-loss head over nine equidistant levels `{0.1, 0.2, ..., 0.9}`, replacing MOIRAI-1's mixture-of-Student-t NLL head and aligned directly with the CRPS metric of GIFT-Eval.
- Autoregressive multi-quantile decoding: a depth-2 expand-then-collapse procedure that rolls out `m x m` candidates per step and collapses them back to `m` quantiles, avoiding the dimensional mismatch of feeding quantiles back in and the information loss of committing to the median.
- Multi-token prediction head (multiple future patches per output token) combined with 50% patch-level random masking for robustness and long-horizon efficiency.
- A new 36M-series / ~295B-observation corpus combining non-leaking GIFT-Eval Pretrain, GIFT-Eval TrainTest train split, Chronos-Mixup (30M synthetic / 63B points), KernelSynth (1M / 1.02B), and ~2.15M internal Salesforce CloudOps telemetry series / ~1.48B daily observations from January 2024.

## Architecture at a glance
Moirai 2.0 is a decoder-only Transformer with RMSNorm, GLU feed-forwards, RoPE positional encodings, causal multi-head self-attention, and residual-block projections on both input and output sides. Univariate series are patched into non-overlapping patches, each concatenated with a binary missing-value indicator and projected to the hidden dimension. Instance-normalization statistics are computed only from the first 30% of each window to avoid future leakage, and samples whose second-70% segment drifts too far are filtered out. The output head produces `n_token * n_q * p` values — multi-token prediction of `n_q = 9` quantile levels for multiple future patches per position. Small is 11.4M parameters, base 87.1M, large 305M; exact `(d, L, heads, patch)` hyperparameters are not disclosed.

## Why it matters
Moirai 2.0 is an explicit self-critique of the MOIRAI-1 recipe: the same Salesforce team argues that masked-encoder + multi-patch + mixture-of-distributions was over-engineered and that a decoder-only + single-patch + quantile-head stack wins on GIFT-Eval at a fraction of the size. Its failure to scale past 11M parameters despite 295B pretraining observations is a concrete data point against naive parameter scaling for TS-FMs.

## Strengths
- Clean row-by-row ablation from MOIRAI-1-small to Moirai-2.0 isolates each change; quantile loss contributes the single largest jump (MASE 0.85 -> 0.744, CRPS 0.58 -> 0.553) and the final residual-block projection plus the full inference stack brings MASE/CRPS to 0.728 / 0.516.
- Training loss aligned with the CRPS metric that GIFT-Eval actually scores, matching the Chronos-Bolt / TiRex / YingLong trend of pinball-loss training.
- Large efficiency win: ~30x smaller and ~2x faster than MOIRAI-1-Large at better accuracy, with a KV-cache case study showing up to 4x speedup at 10K context / 1K horizon and 17x at 10K / 10K for repeated forecasting.
- Open weights (`Salesforce/moirai-2.0-R-small`) and code in the existing `uni2ts` repo.
- Depth-2 expand-collapse decoding is a principled answer to the "how do you autoregress a quantile vector" problem that prior quantile-head TS-FMs sidestep by predicting the whole horizon in one shot.

## Limitations and open critiques
- Scaling from 11.4M to 87.1M to 305M monotonically worsens MASE (0.728 -> 0.732 -> 0.743) and CRPS (0.516 -> 0.525 -> 0.530) on GIFT-Eval; the base and large checkpoints are documented as strictly worse than the small one the team recommends.
- Rank drops from 4th on short horizons to 6th on medium to 8th on long; long-horizon modeling is explicitly listed as unsolved.
- Multivariate forecasting and covariate support are dropped entirely — multivariate inputs are handled as independent univariate series — sharply narrowing the task surface relative to MOIRAI-1's any-variate attention and to [Chronos-2](./chronos-2.md)'s group attention.
- Nature is a persistent weak domain on GIFT-Eval, attributed by the authors to corpus under-coverage of natural and environmental series.
- The 2.15M-series / 1.48B-observation Salesforce internal CloudOps telemetry component is proprietary with undisclosed mixture weights, so the full 36M-series corpus is not reproducible despite four of its five sources being open.
- Core transformer hyperparameters `(d, L, heads, patch size, context length)` are not tabulated; only parameter counts and a "context of approximately 10K" reference in the KV-cache case study are disclosed.

## Follow-up work and dialogue
Moirai 2.0 is a self-directed successor to [MOIRAI](./moirai.md) and [Moirai-MoE](./moirai-moe.md): the same Salesforce team argues the masked-encoder + multi-patch + mixture-of-Student-t recipe of MOIRAI-1 was over-engineered, and that a decoder-only + single-patch + quantile-head stack plus a larger corpus wins on GIFT-Eval at ~1/30 the parameter count. Its closest contemporary is [Chronos-2](./chronos-2.md), the analogous second-generation successor from Amazon: where Chronos-2 kept the encoder-decoder T5 structure and added group attention for multivariate / covariate support, Moirai 2.0 abandoned encoder-decoder entirely and dropped multivariate support altogether, betting on architectural simplification instead of capability expansion.

It joins the decoder-only lineage of [TimesFM](./timesfm.md), [Timer](./timer.md), [Timer-XL](./timer-xl.md), [Sundial](./sundial.md), and [Timer-S1](./timer-s1.md), which share a patched causal decoder but differ on the output head: TimesFM predicts points, Sundial uses flow matching, Timer-S1 uses quantile-CRPS stacked with Serial-Token Prediction blocks, and Moirai 2.0 uses a plain pinball head combined with multi-token prediction and recursive multi-quantile decoding. The explicit failure to scale past 11M is a data point for [../concepts/scaling-laws.md](../concepts/scaling-laws.md) and stands in direct tension with Kairos, YingLong, and Timer-S1, which report continued gains from parameter scaling.

## Reproducibility
- **Open weights:** yes — `Salesforce/moirai-2.0-R-small` on HuggingFace; base and large exist for the scaling experiment but the paper recommends small.
- **Code:** public at `SalesforceAIResearch/uni2ts` (shared repo with MOIRAI-1); GIFT-Eval replication notebooks at `SalesforceAIResearch/gift-eval`.
- **Training data:** partially public — GIFT-Eval Pretrain / TrainTest train, Chronos-Mixup, and KernelSynth are open; the 2.15M-series / 1.48B-observation Salesforce internal CloudOps telemetry component is proprietary, so the full 36M-series mixture is not reproducible.
- **Compute to retrain:** AdamW, lr `1e-3`, weight decay `1e-1`, `β1=0.9`, `β2=0.98`, 10K linear warmup then cosine annealing, 100K total steps at batch size 256, bf16 mixed precision. GPU count, wall-clock, and FLOPs not disclosed.
- **Deployment footprint:** 11.4M / 87.1M / 305M params; approximate context length 10K in the KV-cache case study; exact `(d, L, heads, patch)` not disclosed. Inference demonstrated on a single H200 GPU; small is framed as CPU / small-GPU friendly given the 30x reduction vs MOIRAI-1-Large.

## When to cite this paper
Cite Moirai 2.0 (a) as the canonical reference for the Salesforce pivot from masked encoder + multi-patch + mixture-distribution to decoder-only + single-patch + quantile-loss, (b) for the autoregressive multi-quantile expand-collapse decoding procedure, and (c) as the cleanest published data point that naive parameter scaling can hurt a TS-FM when pretraining data is held fixed — the 11M -> 87M -> 305M regression on GIFT-Eval.

## In the knowledge graph
- **Cluster:** [Cluster 1 — Decoder-only autoregressive TS-FMs](../foundation-models/taxonomy.md#cluster-1--decoder-only-autoregressive-ts-fms) (deliberate pivot away from MOIRAI-1's Cluster 2 masked-encoder membership)
- **Architecture family:** [decoder-only-autoregressive](../architectures/decoder-only-autoregressive.md)
- **Related concepts:** [patch tokenization](../concepts/patch-tokenization.md), [probabilistic forecasting](../concepts/probabilistic-forecasting.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md), [scaling laws](../concepts/scaling-laws.md)
- **Dataset / corpus:** new 36M-series / ~295B-observation Moirai 2.0 corpus (not [LOTSA](../datasets-benchmarks/lotsa.md); combines non-leaking GIFT-Eval Pretrain, GIFT-Eval TrainTest train, Chronos-Mixup, KernelSynth, and Salesforce internal CloudOps telemetry). Evaluation: [GIFT-Eval](../datasets-benchmarks/gift-eval.md).
- **See also:** [moirai](./moirai.md), [moirai-moe](./moirai-moe.md), [chronos-2](./chronos-2.md), [timesfm](./timesfm.md), [sundial](./sundial.md), [timer-s1](./timer-s1.md)
