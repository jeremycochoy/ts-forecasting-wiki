# Patch Tokenization

## Intuition

Patch tokenization groups `P` contiguous time-series values into one transformer token via a linear (or MLP) projection. It is to numeric sequences what subword tokenization is to text: it raises the information content of each token, cuts sequence length by a factor of `P`, and therefore turns quadratic attention into something tractable for long histories. Before PatchTST popularized it in 2023, transformers for TS attended over single timestamps and kept being beaten by simple linear baselines; after patching, transformers finally became competitive and then dominant.

## Mechanics

Given a univariate series `x \in R^T`, patching is essentially a strided unfold followed by a linear map:

```
# x: (B, T)     batch of series
# P: patch length, S: stride (S = P for non-overlapping)
# D: hidden size
patches = unfold(x, size=P, stride=S)    # (B, N, P) with N = (T - P)/S + 1
tokens  = patches @ W_in + b_in          # (B, N, D) — now a transformer-ready sequence
z       = Transformer(tokens)            # (B, N, D)
y_patch = z @ W_out + b_out              # (B, N, P_out) each token decodes a patch of predictions
```

For multivariate inputs the standard trick is *channel independence*: each of `C` channels is patched in parallel, forming a `(B*C, N, D)` batch, processed by the same weights, then reshaped back. This means one set of weights handles any number of channels at inference, and it sidesteps the "variate entanglement" problem that plagued earlier multivariate transformers. RevIN is usually applied per-channel on the raw window *before* patching, so patches live in a standardized scale.

On the output side, models emit either one value per position (1-step-ahead) or an entire output patch at once. `[TimesFM](../papers/timesfm.md)` uses an output patch length larger than the input patch — e.g. `P_in = 32`, `P_out = 128` — so rolling out a 512-step horizon takes 4 forward steps instead of 512.

## Why it works

Three forces are at play. First, *sequence compression*: attention cost is `O((T/P)^2 D)` instead of `O(T^2 D)`, which is the only reason a 4096-step context is computationally feasible at all. Second, *signal-to-noise improvement*: a single timestamp is a scalar with very little local structure, whereas a patch of length 16–64 already contains a shape (trend, short seasonality, a local variance) that the attention layer can reason about meaningfully — this is the same reason 2D image patches work for ViTs. Third, *inductive bias toward local averaging*: the linear projection over a patch is structurally similar to a moving-average filter bank, giving the model a free low-pass prior. Together, patching injects the translation-equivariance and locality priors that pure attention lacks, without handcoding convolutions.

There is also an important training-dynamics effect. Short effective sequences are easier to optimize: the attention softmax is not spread over thousands of positions, gradients propagate over fewer steps, and overfitting to spurious single-step correlations is reduced. This is part of why patched models generalize zero-shot where pre-2023 TS transformers did not.

## Trade-offs and failure modes

The cost of patching is *resolution loss*. Events shorter than the patch length are smeared into the patch average; anomaly detection on high-frequency spikes can miss them if `P` is too long. Patch boundaries are arbitrary, so phenomena whose phase does not align with the grid (e.g. a daily cycle on minute data with an offset) can be aliased. Longer output patches trade rollout speed for per-step calibration: the model commits to a block of predictions at once, which can hurt accuracy when the process has sharp regime changes mid-patch.

Choosing `P` is a data-dependent decision. On hourly data, patches of length 24 (one day) align with natural periodicity; on minute data, length 60 does. Mismatches show up as characteristic errors: predictions that visibly "tile" at the patch boundary, or missing short bursts.

## Design choices in the literature

- `[TimesFM](../papers/timesfm.md)` uses non-overlapping `P_in = 32`, `P_out = 128`, explicitly trading multi-step emission for fewer autoregressive steps.
- `[MOIRAI](../papers/moirai.md)` picks *multiple* patch sizes (e.g. 8/16/32/64/128) and keeps one input projection per size, keyed by the series frequency — a hand-engineered form of multi-scale tokenization.
- `[Moirai-MoE](../papers/moirai-moe.md)` argues that the frequency-keyed projections are insufficient and replaces them with a single projection plus sparse MoE routing that discovers specialization from data.
- `[TTM](../papers/ttm.md)` uses *adaptive* patching where the patch length depends on the effective frequency, then adds a resolution prefix-tuning token so the mixer knows which regime it is in.
- `[Timer-XL](../papers/timer-xl.md)` flattens a multivariate panel into a 1D sequence of patches and lets a universal causal attention treat variates and time uniformly.
- `[Time-LLM](../papers/time-llm.md)` reuses the patch-projection step, but the projected tokens are then *reprogrammed* into an LLM's word-embedding space via cross-attention to text prototypes.

## Open questions

- **Does patch size need to adapt to frequency at inference time?** MOIRAI's multi-projection approach and Moirai-MoE's routed approach disagree on whether specialization should be handcoded or learned; the ablation evidence is not yet decisive.
- **Is non-overlapping the right default?** Overlapping patches give the attention layer multiple phase-shifted views and might recover resolution loss, at the cost of a longer sequence; almost no TS FM has reported a clean ablation.
- **Should patches be conditional on content?** An adaptive "where to cut" policy — analogous to BPE merges in NLP — is untried in TS foundation models.
- **Joint patch-and-channel tokenization.** Channel independence throws away cross-variate information that could be captured if a token spanned multiple channels; Timer-XL's flattening is one attempt but not obviously optimal.
- **How does patching interact with RevIN statistics at long horizons?** RevIN uses window-level statistics; if patches span regime shifts, the assumed stationarity breaks.

## Papers that exemplify this

- `[TimesFM](../papers/timesfm.md)` — input/output patch asymmetry (`P_out > P_in`) for efficient rollout on a decoder-only stack.
- `[MOMENT](../papers/moment.md)` — applies PatchTST-style non-overlapping patching plus RevIN into a masked encoder at multiple sizes (40M/125M/385M).
- `[MOIRAI](../papers/moirai.md)` — introduces per-frequency multi-projection as the input stage of a masked encoder, framed as explicit multi-scale tokenization.
- `[Moirai-MoE](../papers/moirai-moe.md)` — replaces the multi-projection with a single projection and argues routing subsumes frequency specialization.
- `[Timer-XL](../papers/timer-xl.md)` — flattens multivariate panels into one patch sequence for a universal decoder-only attention.
- `[Time-LLM](../papers/time-llm.md)` — projects TS patches and then re-aligns them with an LLM token space via reprogramming.
- `[TTM](../papers/ttm.md)` — adaptive patching and resolution prefix tuning inside a lightweight MLP-Mixer.
- `[Chronos-2](../papers/chronos-2.md)` — group attention operates over patch tokens across related series as its in-context mechanism.
- `[SEMPO](../papers/sempo.md)` — long patches (`L_p = 64`) over a 512-point context; the input is first passed through energy-aware spectral decomposition before patching, so each patch carries an energy-partitioned frequency signature rather than raw samples.
- `[TSPulse](../papers/tspulse.md)` — short patches (`pl = 8`) with `D = 3 · pl = 24` hidden width; hybrid patch-level masking (full and partial patches with a learnable mask token) replaces the block-masking of MOMENT/UniTS, and the same patches feed both time-domain and FFT-domain branches.
- `[TS-JEPA](../papers/ts-jepa.md)` — patches *as input units to JEPA*: 10 non-overlapping patches per series with 70% mask ratio, sin-cos absolute positional encoding, 1D-CNN tokenizer. The patches feed the JEPA latent-space prediction loop rather than a reconstruction or autoregression objective, so patching plays the same input-pipeline role as in masked encoders but the reconstruction target lives in latent space.
- `[LaT-PFN](../papers/lat-pfn.md)` — a counter-example: input is *not* patched. The dilated MobileNet1D embedder produces per-timestep latents directly, and the paper observes that *patch-like* tokens nevertheless emerge at fixed positions in the latent space (paper §5, Figures 9–10), reminiscent of ViT tokenization but learned implicitly. A useful data point on whether patching needs to be hand-engineered into the input pipeline or can be discovered.
- `[TiDE](../papers/tide.md)` (pre-FM) — the explicit counter-example. TiDE deliberately avoids patching, attention, and recurrence, and matches PatchTST on the LTSF benchmarks at 5–10× the speed. Read alongside DLinear: when channel independence and a covariate-aware MLP are in place, patches are not strictly necessary for long-horizon forecasting accuracy on the canonical benchmarks. The TS-FMs in this wiki adopt patches anyway because patches play other roles (token sequence length, multi-scale tokenization, mask granularity) that pure forecasting accuracy does not isolate.

## Related wiki pages

- [RevIN normalization](revin-normalization.md)
- [Value quantization](value-quantization.md)
- [Probabilistic forecasting](probabilistic-forecasting.md)
- [Decoder-only autoregressive](../architectures/decoder-only-autoregressive.md)
- [Masked encoder](../architectures/masked-encoder.md)
- [Lightweight non-transformer](../architectures/lightweight-non-transformer.md)
