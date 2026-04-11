# Scaling Laws

## Intuition

Scaling laws are empirical power-law relationships between held-out loss and the three resources you can spend on pretraining: parameter count `N`, training-token budget `D`, and total compute `C`. In language modeling they settled the question "should we spend marginal compute on more parameters or more data?" and motivated the move from 100M- to 100B-parameter models. For time-series foundation models, the question is more basic: do the Chinchilla-style curves even *apply* to numeric sequences, and if so, are we on a saturation plateau or still in the exponential part? Empirical scaling results are what justify investing in larger TS corpora instead of architectural tweaks.

## Mechanics

A scaling-law study fits a functional form like

```
L(N, D) ≈ L_inf + A * N^(-alpha) + B * D^(-beta)
```

to held-out loss across a grid of model sizes and token budgets. `L_inf` is the irreducible loss (noise floor); `alpha` and `beta` are power-law exponents that tell you how sensitive loss is to parameters and data. For MoE models, `N` splits into *active* and *total* parameters, and both axes are measured separately — if loss tracks active parameters, MoE gives you nothing beyond a dense baseline at equal FLOPs; if it tracks total parameters, sparsity is genuinely providing extra capacity.

The typical experimental setup is:

```
for N in [30M, 70M, 200M, 1B, ...]:
    for D in [1B, 10B, 100B, ...] tokens:
        train model(N, D)
        log validation NLL L(N, D)
fit power law over the grid → alpha, beta, L_inf
```

Because training models at every gridpoint is expensive, researchers usually train a sparse cross of (N, D) values and interpolate. Reporting a clean curve across two orders of magnitude on both axes is the minimum bar for a credible scaling claim.

## Why it works

Scaling laws are not a theorem; they are an empirical regularity that holds whenever three conditions are met: the data is diverse enough that larger models cannot memorize it, the optimization is well-tuned across sizes (same learning-rate schedule family, consistent batch-size-to-size ratio), and the model class is *expressive enough* that the bottleneck is capacity rather than bias. Under those conditions, loss becomes an approximately additive function of log-parameters and log-data because the model's errors decompose into an approximation term (cured by more parameters) and an estimation term (cured by more data), each shrinking as a power law.

Time series sit at an interesting place in this story. The total volume of *real* public TS data is small compared to web text, so TS scaling laws often require synthetic data to push `D` high enough to see a clean curve. The fact that TS FMs *do* show Chinchilla-style curves — instead of saturating at 10M parameters like the pre-FM generation — is the main evidence that the domain has moved from "expressive enough model on limited data" to "compute-limited regime, keep scaling."

## Trade-offs and failure modes

The obvious failure mode is *measuring on the wrong loss*. Next-patch NLL is not the quantity users care about; forecast accuracy at long horizons is. Power-law fits on NLL sometimes fail to translate into proportional improvements on [Monash](../datasets-benchmarks/monash-archive.md) [MASE](../evaluation/metrics.md#17-mase--mean-absolute-scaled-error) or [GIFT-Eval](../datasets-benchmarks/gift-eval.md) [CRPS](../evaluation/metrics.md#21-crps--continuous-ranked-probability-score), especially because patched autoregressive rollout accumulates error nonlinearly. A second failure mode is *leakage*: if the pretraining corpus touches the test set, larger models overfit to leakage rather than learning, and the curve looks great without transferring.

Sparse MoE scaling is trickier than dense. Load-balancing losses and router choices change the effective `N`, and reporting total-parameter scaling without active-parameter scaling can be misleading. Finally, TS scaling may be bottlenecked by *data diversity* rather than data volume: doubling tokens by repeating the same few domains does less than doubling by adding new ones, which is why synthetic augmentation (KernelSynth, TSMix) matters even more than in NLP.

## Design choices in the literature

- `[Lag-Llama](../papers/lag-llama.md)` was among the first open TS FMs to publish scaling-law curves, framed as empirical evidence that decoder-only TS transformers track LLM-style loss trends.
- `[Time-MoE](../papers/time-moe.md)` pushes scale: 2.4B total parameters (sparse MoE) on the [Time-300B](../datasets-benchmarks/time-300b.md) corpus, reporting clean curves as a function of both active and total parameters — the first billion-parameter TS FM and the cleanest evidence that TS is still compute-limited.
- `[TimesFM](../papers/timesfm.md)` at ~200M on a ~100B-point corpus sits on an intermediate point of the curve and shows that 100M–1B dense is already enough to approach supervised-SOTA zero-shot on Monash.
- `[Timer](../papers/timer.md)` reports emergent few-shot capability — roughly 99% data reduction to match supervised baselines — which can be read as the TS analog of the few-shot emergence observed in LLM scaling.
- `[Sundial](../papers/sundial.md)` extends scale via [TimeBench](../datasets-benchmarks/timebench.md) (~1T points) combined with a flow-matching objective, implying that scaling has to be joint with the training objective, not just with raw tokens.

## Open questions

- **Are the TS scaling exponents the same as NLP?** Lag-Llama and Time-MoE report *qualitatively* similar slopes but the comparison is not clean; `alpha` and `beta` for TS have not been pinned down.
- **Data diversity vs data volume.** Does doubling *domains* matter more than doubling *points* per domain? No systematic ablation exists.
- **Sparse vs dense Pareto frontier.** Time-MoE argues MoE is on a better frontier than dense TimesFM; the comparison lacks matched training budgets.
- **Does scaling help long-horizon forecasting?** Loss-at-next-patch scales cleanly, but MASE-at-horizon-100 may not, because error compounding is not captured by NLL.
- **Scaling with synthetic data.** How far can synthetic-only pretraining ([Mamba4Cast](../papers/mamba4cast.md), KernelSynth) push the curve before saturation?

## Papers that exemplify this

- `[Lag-Llama](../papers/lag-llama.md)` — first published TS scaling-law study on an open decoder-only FM, establishing that power-law behavior transfers from language to numeric sequences.
- `[Time-MoE](../papers/time-moe.md)` — 2.4B sparse MoE on Time-300B, clean scaling on both active and total parameters, the first billion-param TS FM.
- `[TimesFM](../papers/timesfm.md)` — ~200M dense decoder on ~100B points, anchors the middle of the dense curve and reaches near-supervised zero-shot.
- `[Timer](../papers/timer.md)` — emergent few-shot at scale, analogous to LLM few-shot emergence.
- `[Sundial](../papers/sundial.md)` — ~1T-point TimeBench pretraining with a flow-matching objective, probing whether scale plus objective jointly shift the curve.

## Related wiki pages

- [Synthetic data augmentation](synthetic-data-augmentation.md)
- [Zero-shot forecasting](zero-shot-forecasting.md)
- [Multi-task universal](multi-task-universal.md)
- [Decoder-only autoregressive](../architectures/decoder-only-autoregressive.md)
- [Mixture-of-experts](../architectures/mixture-of-experts.md)
- [Flow-matching continuous](../architectures/flow-matching-continuous.md)
- [Training recipes](../research/training-recipes.md) — the disclosed optimizer / LR / step / hardware values behind each scaling-curve point.
