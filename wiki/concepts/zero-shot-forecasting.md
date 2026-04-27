# Zero-Shot Forecasting

## Intuition

Zero-shot forecasting is the ability of a single pretrained checkpoint to produce accurate forecasts on a new dataset, domain, or frequency *without any gradient updates*. It is the capability that separates a "foundation model" from a conventional supervised forecaster: instead of training a model per series, you ship one set of weights and let the user call them on anything. If zero-shot works, it implies the pretraining corpus spans enough of the space of temporal dynamics that most real-world series are close to in-distribution — the same conceptual leap that made GPT a general text interface.

## Mechanics

Zero-shot inference has a simple shape:

```
# User has: history x[1..H], wants forecast for h[1..L]
# Model: pretrained foundation model f_theta (never touched at inference)
x_norm, stats = RevIN_forward(x)        # per-instance standardization
tokens        = patch_and_project(x_norm)
z             = f_theta(tokens)          # forward pass, no backward
y_norm        = decode_head(z, horizon=L)
y_hat         = RevIN_inverse(y_norm, stats)
```

For zero-shot to produce sensible numbers, three things must hold. The input pipeline must accept *variable history length* — a user might have 50 or 5000 past points. The output stage must support *variable horizon length*, typically by rolling out patches autoregressively, by a fixed-length decoder with padding, or by a horizon-conditional head. And the normalization layer must absorb *arbitrary scale and level*, which is exactly what RevIN (or the equivalent mean-scaling used by Chronos) provides. Models that fail on one of these axes cannot be zero-shot, period — they become single-task supervised models requiring fine-tuning.

At evaluation time, zero-shot is measured on benchmarks the model was *never trained on*: the [Monash Archive](../datasets-benchmarks/monash-archive.md), [GIFT-Eval](../datasets-benchmarks/gift-eval.md), and more recently fev-bench are the standard choices. Leakage control is taken seriously — for example, MOIRAI's [LOTSA](../datasets-benchmarks/lotsa.md) corpus explicitly excludes overlap with Monash test splits.

## Why it works

Zero-shot in time series is not magic; it rests on three structural observations. First, *most real series share a small set of motifs* — trend, seasonality at common periods, bursts, decay, regime shifts — so a model that has seen enough diverse series implicitly learns a library of those motifs and composes them to explain new data. Second, *per-instance normalization* removes the nuisance axes (level and scale) that otherwise dominate generalization error, so the pretraining distribution only needs to cover *shapes*, not raw values. Third, *autoregressive factorization* is composable across horizons, so once the model is good at one-step or one-patch prediction from typical contexts, it rolls out arbitrary horizons by repeated application.

This is structurally similar to zero-shot transfer in NLP: a left-to-right model trained on a big text mixture generalizes to new domains because the conditional `p(next token | context)` is the composable primitive. For TS, `p(next patch | history)` plays the same role, and the invariance to scale and level is the analog of whitespace-and-case normalization.

## Trade-offs and failure modes

Zero-shot is not universally strong. It degrades when (a) the target series comes from a *regime not represented* in pretraining — e.g. a totally new physical process, an unusual frequency like 10 kHz, or a genuinely intermittent demand series with lots of zeros; (b) *covariates matter* and the model has no way to ingest them, which rules out most univariate TS FMs on problems where exogenous features drive the target; (c) *long horizons* expose cumulative error from autoregressive rollout, which is why models with direct multi-horizon heads (MOIRAI, [Chronos-2](../papers/chronos-2.md), [Sundial](../papers/sundial.md)) often beat next-step decoders at long horizons; and (d) *heavy-tailed or extreme-event* series, where a uniform quantizer clips or a Student-t head underweights the tails.

Supervised fine-tuning on a target dataset still typically beats zero-shot by 5–20 percent on standard metrics, so the real claim is that zero-shot closes most — not all — of the gap. The practical question is whether the accuracy gap is worth the operational burden of fine-tuning.

## Design choices in the literature

- `[TimesFM](../papers/timesfm.md)` trains a ~200M decoder-only patched transformer on Google Trends, Wikipedia pageviews, and synthetic data and reports near-supervised accuracy zero-shot on Monash; its long output patch makes long-horizon rollout more stable.
- `[Chronos](../papers/chronos.md)` relies on [value quantization](./value-quantization.md) plus an unmodified T5 stack; zero-shot generalization is measured across 42 datasets drawn from diverse domains.
- `[MOIRAI](../papers/moirai.md)` pretrains on LOTSA (~27B observations) and reports GIFT-Eval and Monash results as the primary zero-shot benchmark, with multi-patch-size projections to handle arbitrary frequency.
- `[TimeGPT-1](../papers/timegpt.md)` is commercially built around zero-shot as the user interface — the API accepts a new series and returns a forecast with no per-series training step.
- `[Lag-Llama](../papers/lag-llama.md)` was the first open decoder-only probabilistic FM to demonstrate zero-shot at scale, using lag features as architectural priors.
- `[TTM](../papers/ttm.md)` shows that a sub-5M MLP-Mixer can beat many larger transformer FMs zero-shot, arguing the inductive biases matter more than raw parameter count.
- `[Mamba4Cast](../papers/mamba4cast.md)` is zero-shot by construction — trained on synthetic priors only, it has never seen any real series.
- `[LLMTime](../papers/llmtime.md)` is the surprising data point that a vanilla GPT-3/Llama-2 with numbers-as-text tokenization reaches competitive zero-shot without any TS-specific pretraining.

## Open questions

- **How large a pretraining corpus is actually needed?** TTM and Mamba4Cast suggest data quality and priors beat raw scale; [Time-MoE](../papers/time-moe.md) and Sundial suggest the opposite. The Pareto frontier is not well mapped.
- **Covariate-aware zero-shot.** Chronos-2 and [Timer-XL](../papers/timer-xl.md) start to address this but most univariate FMs cannot ingest covariates at all.
- **What fails — distribution shift or representation?** When zero-shot fails, is it because the inductive biases are wrong or because the corpus missed the regime? The ablations to answer this cleanly have not been published.
- **Is instance normalization hiding the real problem?** RevIN assumes window-level stationarity; non-stationary dynamics are mostly swept under the rug by normalization.
- **Leakage in benchmarks.** As pretraining corpora grow, the boundary with Monash and GIFT-Eval becomes porous; there is active work on leak-audited benchmarks (fev-bench).

## Papers that exemplify this

- `[TimesFM](../papers/timesfm.md)` — near-supervised zero-shot accuracy on Monash from a ~200M decoder-only stack.
- `[Chronos](../papers/chronos.md)` — 42-dataset zero-shot evaluation with T5 plus value quantization.
- `[TimeGPT-1](../papers/timegpt.md)` — commercial zero-shot API paired with conformal uncertainty.
- `[Lag-Llama](../papers/lag-llama.md)` — first open decoder-only probabilistic zero-shot FM, with lag-feature priors.
- `[MOIRAI](../papers/moirai.md)` — LOTSA-pretrained masked encoder with strong Monash and GIFT-Eval numbers.
- `[TTM](../papers/ttm.md)` — tiny MLP-Mixer outperforming larger FMs, arguing priors dominate scale.
- `[Mamba4Cast](../papers/mamba4cast.md)` — synthetic-only pretraining, zero-shot as its only mode of operation.
- `[LLMTime](../papers/llmtime.md)` — vanilla LLM as a zero-shot forecaster via ASCII tokenization.
- `[Moirai 2.0](../papers/moirai-2.md)` — 11.4M decoder-only small variant, evaluated zero-shot on [GIFT-Eval](../datasets-benchmarks/gift-eval.md) only, ranks 5th by MASE and 6th by CRPS among pretrained foundation models; multivariate/covariate inputs are dropped entirely.
- `[SEMPO](../papers/sempo.md)` — 6.5M transformer, 23.1% MSE / 10.8% MAE reduction on TSLib zero-shot versus billion-parameter baselines at 10x-100x fewer params and 10x-3000x fewer pretraining points (Table 1).
- `[LaT-PFN](../papers/lat-pfn.md)` — JEPA + PFN trained on synthetic data only, with related-series in-context examples curating zero-shot inference; beats ARIMA / FBProphet / ForecastPFN on 5 LTSF datasets (Table 1) without ever seeing real data during training. The "synthetic-only zero-shot" branch alongside Mamba4Cast.

## Related wiki pages

- [RevIN normalization](revin-normalization.md)
- [Patch tokenization](patch-tokenization.md)
- [In-context learning](in-context-learning.md)
- [Synthetic data augmentation](synthetic-data-augmentation.md)
- [Decoder-only autoregressive](../architectures/decoder-only-autoregressive.md)
- [Masked encoder](../architectures/masked-encoder.md)
