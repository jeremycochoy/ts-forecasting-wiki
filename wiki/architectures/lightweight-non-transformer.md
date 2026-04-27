# Lightweight Non-Transformer

## Intuition

Lightweight non-transformer TS foundation models drop the quadratic self-attention backbone in favor of MLP-Mixer-style channel/time mixing or state-space models (SSMs). The bet is that transformers are *overkill* for many TS workloads: if you patch aggressively, normalize cleanly, and use the right inductive biases, a 1–5M parameter model can match or beat billion-parameter transformers — while running on a CPU, in one forward pass, at millisecond latency. This is a direct philosophical counterweight to the MoE scale-up narrative: where `[Mixture-of-experts](mixture-of-experts.md)` says "more parameters, cheaper compute," lightweight non-transformers say "fewer parameters, correct priors."

## Mechanics

Two representative architectures dominate: TTM's TSMixer and Mamba4Cast's selective SSM.

**TSMixer (used in TTM).** After patching and RevIN, alternate two kinds of MLP blocks — time-mixing and channel-mixing — on a `(C, N, D)` tensor (channels, patches, hidden):

```
# after patch + RevIN: x in R^{B x C x N x D}
for block in 1..L:
    # time-mixing: MLP along the patch axis, shared across channels
    y = x.transpose(N, D)                     # (B, C, D, N)
    y = MLP_time(y)                            # mixes across N
    y = y.transpose(D, N)
    x = x + y
    # channel-mixing: MLP along the channel axis, shared across patches
    y = x.transpose(C, D)                     # (B, D, N, C)
    y = MLP_channel(y)                         # mixes across C
    y = y.transpose(D, C)
    x = x + y
# forecast head: project over the time axis
y_hat = x @ W_out                              # (B, C, L)
```

Alternation gives the model both cross-time and cross-channel information flow without ever running attention. TTM adds three refinements: **adaptive patching** (patch length scales with effective frequency), **resolution prefix tuning** (a learned token embedding conditions the mixer on sampling frequency), and a **multi-level design** that separates channel modeling from exogenous-variable modeling. Total parameters: 1–5M, CPU-deployable.

**Selective SSM (Mamba4Cast).** Replace attention with a Mamba layer. A selective state-space model maintains a fixed-size hidden state `h_t` that is updated by a linear recurrence whose parameters are input-dependent:

```
# continuous-time SSM, discretized and made selective
# A, B, C, Delta are functions of the input (selective)
for t in 1..T:
    A_bar, B_bar = discretize(A, B, Delta(x_t))
    h_t = A_bar * h_{t-1} + B_bar * x_t     # linear recurrence
    y_t = C * h_t                            # output
```

Cost is `O(T D)` in time and memory instead of `O(T^2 D)` for attention. The horizon is emitted in *one forward pass* by unrolling the recurrence through the future positions — no autoregressive rollout. Training uses the PFN (Prior-data Fitted Network) objective: sample a generative process from a prior, generate `(x, y)`, minimize `-log f_theta(y | x)`, repeat. The resulting model is an amortized Bayesian forecaster under its learned prior, trained *entirely on synthetic data*.

## Why it works

Both families lean on the observation that *the bottleneck in TS is usually the input pipeline, not the sequence model*. Once patching, channel independence, RevIN, and a reasonable training objective are in place, the actual shape-modeling work is mild — a few MLPs or an SSM recurrence is sufficient. Self-attention's quadratic cost buys flexibility the model does not need for smooth, locally-structured numeric sequences.

TTM's argument is more specific. MLP-Mixer has a strong inductive bias: separate time-mixing and channel-mixing layers force the model to process the two axes independently. On TS this matches the structure of the data almost exactly — time is one axis, variates are another, and they interact only through the alternation pattern. Attention is a more general mixer that has to *learn* to factor into the same two axes; MLP-Mixer gives it for free. Add adaptive patching and resolution prefix tuning, and the model has everything needed for multi-frequency generalization without ever learning an attention pattern.

Mamba4Cast's argument is orthogonal: linear-time recurrence plus a rich synthetic prior is enough to approximate the posterior predictive for any process in the prior's support. Because the model is trained on fresh prior samples at every step, it never overfits to a finite dataset; because the SSM is linear-time, it handles arbitrary long contexts cheaply; because the PFN interpretation is Bayesian, it is zero-shot by construction.

## Trade-offs and failure modes

The obvious ceiling for lightweight models is *peak accuracy on complex tasks*. For multi-modal distributions, very long horizons, or series whose dynamics genuinely require long-range dependencies, a small MLP-Mixer or SSM cannot match the expressive power of a large transformer. TTM beats many FMs on *typical* datasets but concedes ground on hard long-horizon benchmarks and on datasets with sharp regime shifts.

Mamba4Cast's PFN training means its quality is entirely bounded by the prior: if the target series comes from a regime poorly covered by the prior, the model's calibration degrades gracefully, but it cannot recover from a bad prior by seeing more data. This is a fundamentally different failure mode from real-data FMs, which simply overfit to their training distribution.

Both families also lack a clean path to [in-context learning](../concepts/in-context-learning.md) across siblings. Channel mixing in TTM is limited to *aligned* variates; the "panel of related series" setting that [Chronos-2](../papers/chronos-2.md) and [Timer-XL](../papers/timer-xl.md) handle is not a natural fit for either TSMixer or Mamba at this point. Covariate support in TTM is handled by a dedicated exogenous head, which is a pragmatic patch rather than a unified mechanism.

## Siblings and design space

Compared to `[Decoder-only autoregressive](decoder-only-autoregressive.md)`, lightweight non-transformers trade expressive ceiling for latency and size. Compared to `[Masked encoder](masked-encoder.md)`, they trade bidirectional context and multi-task heads for architectural simplicity. Compared to `[Flow-matching continuous](flow-matching-continuous.md)`, Mamba4Cast shares the "one-shot horizon" philosophy but uses a recurrence instead of an ODE integration.

## Design choices in the literature

- `[TTM (Tiny Time Mixer)](../papers/ttm.md)` — 1–5M parameters, TSMixer MLP-Mixer backbone on patched inputs, adaptive patching, resolution prefix tuning, and a multi-level design that separates channel modeling from exogenous-variable modeling. Pretrained with an exogenous head for covariate support. CPU-deployable, beats many larger transformer FMs in zero-shot and few-shot on standard benchmarks. NeurIPS 2024.
- `[Mamba4Cast](../papers/mamba4cast.md)` — Mamba selective SSM backbone, trained entirely on synthetic data under the Prior-data Fitted Network (PFN) paradigm. Single-pass horizon generation via recurrence unrolling, linear-time inference in history length. Zero-shot by construction since it never sees real series during pretraining.
- `[TSPulse](../papers/tspulse.md)` — IBM Granite's 1.06M-parameter TSMixer-based multi-task model. Eight TSMixer blocks with softmax-gated attention, hidden `D=24`, patch 8, context 512; pretrained with dual-space (time + FFT) masked reconstruction on the same Monash + LibCity corpus as [TTM](../papers/ttm.md). Explicitly disentangles TimeE / FFTE / register-token semantic embeddings and targets classification, imputation, anomaly detection and similarity search rather than forecasting.
- `[SEMPO](../papers/sempo.md)` — 6.5M encoder-decoder *transformer* that the paper self-positions in the lightweight tier. Despite the attention backbone, the sub-10M parameter count and 83M-point pretraining corpus put it next to [TTM](../papers/ttm.md) and Mamba4Cast on the "small model, strong zero-shot" axis. Contribution lives in two plug-ins rather than the backbone: energy-aware spectral decomposition (EASD) replaces random patch masking with FFT-domain masking, and a 128-prompt mixture-of-prompts (MoP) layer injects prefix conditioning into self-attention K/V — routing a prefix, not an FFN, so it is distinct from MoE.
- `[LaT-PFN](../papers/lat-pfn.md)` — uses an 8-layer dilated MobileNet1D as its encoder embedder, explicitly noting in §3.3 that they "found convolutions to be superior to attention over the sequence dimension." This is a quiet but useful confirmation of the lightweight-non-transformer thesis from inside the [JEPA cluster](../foundation-models/taxonomy.md#cluster-8--jepa--latent-space-prediction): when the prediction objective lives in latent space, a CNN embedder is enough — attention over sequence positions is not needed.
- `[TiDE](../papers/tide.md)` (pre-FM) — the closest design ancestor, in lineage if not in cluster. Pure MLP encoder-decoder, channel-independent, covariate highway, linear residual; the Google team behind TiDE went on to build [TimesFM](../papers/timesfm.md). The lightweight TS-FM family (TTM, SEMPO, TSPulse) generalizes TiDE's "MLPs are enough" finding to the foundation-model setting.

## Open questions

- **Priors vs scale.** TTM and Mamba4Cast argue strong priors make scale unnecessary; [Time-MoE](../papers/time-moe.md) and [TimesFM](../papers/timesfm.md) argue scale wins eventually. The Pareto frontier at matched wall-clock inference cost is not well mapped.
- **Cross-variate structure in MLP-Mixer.** TTM's channel mixing is aligned-variate only; extending to panel siblings and arbitrary related series is open.
- **SSM scaling for TS.** Mamba models scale in language; whether TS FMs can ride the same curve to billion-parameter SSMs is untested.
- **Hybrid architectures.** Mamba + attention, or MLP-Mixer + sparse attention, might give the best of both worlds; TS FMs have not seriously explored hybrids.
- **Synthetic-only pretraining.** Mamba4Cast is the only large synthetic-only TS FM; whether the recipe generalizes to larger models and richer priors is wide open.

## Related wiki pages

- [Patch tokenization](../concepts/patch-tokenization.md)
- [Synthetic data augmentation](../concepts/synthetic-data-augmentation.md)
- [Zero-shot forecasting](../concepts/zero-shot-forecasting.md)
- [RevIN normalization](../concepts/revin-normalization.md)
- [Decoder-only autoregressive](decoder-only-autoregressive.md)
- [Masked encoder](masked-encoder.md)
- [Flow-matching continuous](flow-matching-continuous.md)
