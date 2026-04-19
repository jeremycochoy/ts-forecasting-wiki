# Hyperparameter Transfer Across Scales

When training a family of models at multiple sizes (e.g., Tiny / Small /
Base / Large), the naive approach is to grid-search learning rate, weight
decay, initialization scale, and other hyperparameters independently at
every size. This is prohibitively expensive at larger scales. HP transfer
methods let you tune cheaply at a small size and predict the right HPs
for larger sizes without re-searching.

## Intuition

The core problem: a learning rate that works at `d=512` is too large at
`d=1024`. Wider layers produce larger activation magnitudes and gradient
norms, so the same LR causes divergent updates. Similarly, initialization
variance that is correct for one width produces exploding or vanishing
signals at another. HP transfer methods either (a) reparameterize the
model so that the optimal HPs are width-invariant by construction, or
(b) establish empirical scaling rules that predict how each HP should
change as `d` or `L` grows.

## Methods

### 1. muP (Maximal Update Parameterization)

The gold standard, grounded in the Tensor Programs theory of Greg Yang.
muP reparameterizes a transformer so that optimal LR, init, and other HPs
are provably invariant to width `d`. The key changes vs standard
parameterization (SP):

- **Learning rate:** in SP, optimal LR shrinks as `d` grows. In muP, the
  hidden-layer LR is scaled by `(d_base / d)` automatically via the
  parameterization, so the user-facing LR stays constant across widths.
- **Initialization:** input/output layers and hidden layers use different
  init scales that depend on `(d_base / d)`.
- **Attention logits:** scaled by `1/d` instead of `1/sqrt(d)`, preventing
  entropy collapse at large width.
- **Output layer LR:** decoupled from hidden-layer LR.

**Workflow:** tune all HPs on a small "base shape" (e.g., `d=256, L=6`),
then scale to any target width with zero re-tuning.

**Limitation:** muP formally handles width scaling only. Depth scaling
(`L=6` to `L=24`) is not covered by the theory — in practice, optimal LR
is less sensitive to depth than to width, so the gap is tolerable but not
zero.

**References:**
- Yang et al., "Tensor Programs V: Tuning Large Neural Networks via
  Zero-Shot Hyperparameter Transfer"
  ([arXiv:2203.03466](https://arxiv.org/abs/2203.03466), 2022). The
  foundational paper. Library: `maximal-update-parameterization` on GitHub.
- Bordelon et al., "Depthwise Hyperparameter Transfer in Residual
  Networks" ([arXiv:2309.16620](https://arxiv.org/abs/2309.16620),
  ICLR 2024). Shows muP alone does NOT transfer across depth;
  proposes residual branch scaling of `1/sqrt(depth)` combined with
  muP to enable joint width+depth transfer.
- Blake et al., "u-muP: The Unit-Scaled Maximal Update
  Parametrization" ([arXiv:2407.17465](https://arxiv.org/abs/2407.17465),
  ICLR 2025). Simplified muP that removes base-shape and init-scale
  hyperparameters; shows an LR-only search of 9 runs suffices.

### 2. muP (simple) — the LR scaling rule only

Also informally called "poor man's muP." Skip the full muP library.
Apply only the dominant effect — the per-layer LR scaling rule:

```
lr_target = lr_proxy * (d_proxy / d_target)
```

Equivalently: scale the LR for each linear layer by `base_fan_in /
fan_in`. If your best Tiny LR (`d=512`) is `3e-3`, then Small (`d=768`)
gets `2e-3` and Base (`d=1024`) gets `1.5e-3`.

This variant was named **"muP (simple)"** by Wortsman et al. (2024),
who compared it against full muP (which adds output-head init scaling,
attention scaling change from `1/sqrt(d_h)` to `1/d_h`, and zero-init
of query projection weights) and found:

> "We add additional features from Yang et al. in Appendix Figure E.5
> **without measurable improvement** at the largest scale we test."

Everett et al. (2024) went further, showing that standard
parameterization with per-layer LR scaling actually **outperforms** full
muP — the transfer is "fully encapsulated by the per-layer LR exponents,"
and the other muP changes are unnecessary. A NeurIPS 2025 paper (Choi
et al.) additionally found that independent weight decay is a more
important mechanism than muP for LR transfer in practical LLM training.

**References:**
- Wortsman et al., "Small-scale proxies for large-scale Transformer
  training instabilities"
  ([arXiv:2309.14322](https://arxiv.org/abs/2309.14322), ICLR 2024
  Oral). Defines "muP (simple)" and shows the additional muP features
  provide no measurable improvement. The canonical reference for this
  technique.
- Everett et al., "Scaling Exponents Across Parameterizations and
  Optimizers"
  ([arXiv:2407.05872](https://arxiv.org/abs/2407.05872), 2024). Shows
  SP + per-layer LR scaling outperforms full muP.
- Choi et al., "Weight Decay may matter more than muP for Learning
  Rate Transfer in Practice"
  ([arXiv:2510.19093](https://arxiv.org/abs/2510.19093), NeurIPS 2025).
- See also the "How To Scale" guide at
  [howtoscalenn.github.io](https://howtoscalenn.github.io/) which
  documents muP (simple) as a recommended practice.

### 3. Progressive transfer (previous-size warm-start)

Tune HPs exhaustively on the smallest model (cheap, hours on a consumer
GPU). For each subsequent size, take the previous best LR and weight
decay and run a narrow 3–5 point grid around them. This is what most
TS-FM papers actually do in practice — they just do not report it
explicitly.

**Pros:** simple, no implementation overhead, works for arbitrary
architecture changes (not just width scaling).

**Cons:** LR drift accumulates across steps. By the fourth or fifth size,
the grid center may have drifted far from the true optimum. Mitigate by
combining with the `lr ∝ 1/d` rule (method 2) to re-center the grid at
each step.

**Reference:** standard practice; the empirical basis is in Kaplan et al.,
"Scaling Laws for Neural Language Models"
([arXiv:2001.08361](https://arxiv.org/abs/2001.08361), 2020) and Hoffmann
et al., "Training Compute-Optimal Large Language Models"
([arXiv:2203.15556](https://arxiv.org/abs/2203.15556), 2022), which
established the power-law relationship between optimal LR and model size.

### 4. Cross-shaped proxy search

For a target config `(d_target, L_target)`, train two cheap proxies:

- **Proxy A** — same depth, smaller width: `(d_small, L_target)`.
  Tests width-independent HP sensitivity.
- **Proxy B** — same width, smaller depth: `(d_target, L_small)`.
  Tests the HPs at the target width with fewer layers (cheaper).

Grid-search LR on both. If they agree, high confidence for the full
model. If they disagree, you know which axis (width vs depth) is driving
the sensitivity — itself useful diagnostic information.

**Why it works:** optimal LR is much more sensitive to width than to
depth. Proxy B (same width, fewer layers) is therefore the more reliable
predictor of the target LR. Proxy A tells you the depth-independent
component and gives you a consistency check.

**Example for Base `(d=1024, L=12, 151M)`:**

```
Proxy A: d=512,  L=12 →  37.7M  (fits on a 4090)
Proxy B: d=1024, L=6  →  75.5M  (fits on a 4090 with bf16)
```

Both proxies are trainable on consumer hardware; together they bracket
the target along both axes.

**Reference:** no single paper describes this exact two-proxy strategy.
The technique synthesizes insights from three published results:

- Bordelon et al. ([arXiv:2309.16620](https://arxiv.org/abs/2309.16620),
  ICLR 2024) — established that width and depth are fundamentally
  different axes for HP transfer (muP handles width; depth requires
  separate `1/sqrt(L)` residual-branch scaling). This justifies probing
  each axis independently.
- Li, Zheng et al., "Predictable Scale: Part I, Step Law"
  ([arXiv:2503.04715](https://arxiv.org/abs/2503.04715), 2025) — found
  that optimal LR depends on total parameter count `N` and dataset size
  `D` via `eta(N,D) = 1.79 * N^(-0.713) * D^(0.307)`, and that this
  law is **invariant to model shape** (width-to-depth ratio at fixed N).
  If true, two proxies at different shapes but similar N should agree
  on LR — providing a cross-validation mechanism.
- Mlodozeniec et al., "Completed Hyperparameter Transfer across
  Modules, Width, Depth, Batch and Duration"
  ([arXiv:2512.22382](https://arxiv.org/abs/2512.22382), Apple, 2025) —
  uses Kronecker factorization of per-layer LR across width and depth,
  which is the formal version of the intuition behind the cross-shaped
  decomposition. However, this paper uses a single small proxy, not two.
- Standard muP practice (EleutherAI, Cerebras guides) recommends a
  proxy at the same depth as the target but reduced width — the
  width-only half of the cross-shaped idea.

### 5. Architectural stabilizers (orthogonal, combine with any method above)

These do not transfer HPs directly but reduce the sensitivity of optimal
LR to scale, making all transfer methods more reliable:

- **QK-Norm (Query-Key Normalization):** normalizes Q and K before the
  dot product, preventing attention logit blow-up at large `d`. Used by
  [Timer-S1](../papers/timer-s1.md). Achieves a similar effect to muP's
  `1/d` attention scaling, without the full reparameterization.
  Reference: Dehghani et al., "Scaling Vision Transformers to 22 Billion
  Parameters" ([arXiv:2302.05442](https://arxiv.org/abs/2302.05442),
  2023); Henry et al., "Query-Key Normalization for Transformers"
  ([arXiv:2010.04245](https://arxiv.org/abs/2010.04245), 2020).
- **Pre-RMSNorm:** applying RMSNorm before each sub-layer (attention,
  FFN) rather than after. Stabilizes residual-stream magnitudes across
  depths. Standard in modern transformers (LLaMA, Mistral) and in
  [SEMPO](../papers/sempo.md), [Moirai 2.0](../papers/moirai-2.md).
  Reference: Zhang and Sennrich, "Root Mean Square Layer Normalization"
  ([arXiv:1910.07467](https://arxiv.org/abs/1910.07467), 2019).
- **Spectral reparameterization (sigma-reparam):** normalizes weight
  matrices by their spectral norm during training, bounding the
  Lipschitz constant of each layer. Reference: Zhai et al., "Stabilizing
  Transformer Training by Preventing Attention Entropy Collapse"
  ([arXiv:2303.06296](https://arxiv.org/abs/2303.06296), 2023).

**Practical recommendation:** use QK-Norm + Pre-RMSNorm in all sizes of
your model family. These are cheap (nearly zero overhead), widely adopted,
and they reduce the LR-sensitivity gap between sizes — making the poor
man's muP rule and the cross-shaped proxy search both more reliable.

## Recommended workflow for a consumer-GPU budget

1. Use QK-Norm + Pre-RMSNorm in every size.
2. Full grid search on Tiny (`d=512, L=6`, hours on a 4090).
3. Apply `lr_small = lr_tiny × (512 / 768)` for Small. Validate with a
   quick 3-point check.
4. For Base: run Proxy B (`d=1024, L=6`) to validate the LR at the target
   width. If it agrees with the `lr × (512 / 1024)` rule, proceed.
5. Base+ and Large inherit from Base — same `d`, just deeper, LR barely
   changes with depth.

## Open questions

- Does muP's width-transfer theory extend cleanly to patched time-series
  inputs, where the effective "vocabulary" is continuous and the sequence
  length depends on `patch_size`? No TS-FM paper has tested this.
- Is the `lr ∝ 1/d` rule sufficient for mixture-of-experts TS-FMs, where
  only a subset of parameters are active per token?
- The cross-shaped proxy assumes width and depth effects on LR are
  approximately separable. Under what conditions does this break down?
- Can architectural stabilizers (QK-Norm, sigma-reparam) fully replace
  muP at TS-FM scales, or do they only reduce the problem?

## Papers that use or are relevant to this concept

- [Timer-S1](../papers/timer-s1.md) — uses QK-Norm + Pre-RMSNorm for
  training stability across its 8.3B-param MoE architecture.
- [Moirai 2.0](../papers/moirai-2.md) — uses RMSNorm, GLU FFN, RoPE;
  trains three sizes but does not report HP transfer methodology.
- [SEMPO](../papers/sempo.md) — uses RMSNorm + SwiGLU at 6.5M params.
- [Lag-Llama](../papers/lag-llama.md) — the most explicit TS-FM
  scaling-law study; grid-searched HPs per size (no transfer method).
- [Time-MoE](../papers/time-moe.md) — trains 50M / 200M / 2.4B sizes;
  HP methodology not disclosed.

## Related wiki pages

- [Scaling laws](scaling-laws.md) — the empirical evidence on how loss
  scales with params, data, and compute.
- [Data normalization](data-normalization.md) — the preprocessing
  decision that interacts with the RevIN / RMSNorm placement choices
  this page discusses.
- [Patch tokenization](patch-tokenization.md) — patch size P affects
  effective sequence length and therefore HP tuning at scale.
- [Model sizing cheat sheet](../benchmarks/model-sizing-cheatsheet.md) —
  the `(d, L)` configs this page helps you tune HPs for.
- [Training a small model](../benchmarks/training-a-small-model.md) —
  where to train and what size to target.
- [Training recipes](../research/training-recipes.md) — the optimizer and
  schedule choices each published model made.
- [Decoder-only autoregressive](../architectures/decoder-only-autoregressive.md) — the dominant architecture family where HP transfer has been studied most.
- [Mixture-of-experts](../architectures/mixture-of-experts.md) — sparse-MoE adds routing / aux-loss HPs that compound the transfer problem.
