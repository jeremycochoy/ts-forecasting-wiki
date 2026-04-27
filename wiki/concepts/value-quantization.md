# Value Quantization

## Intuition

Value quantization maps continuous real numbers to a finite alphabet of discrete tokens so that forecasting can be cast as next-token classification. Once the continuous axis is discretized, *nothing about language-model architectures needs to change*: the same T5 or GPT codebase, the same cross-entropy loss, the same sampling code, even the same HuggingFace checkpoints and training scripts all apply. It is a deliberate inversion of decades of regression-based forecasting: we give up exact real-valued outputs in exchange for a fully probabilistic, drop-in language-model stack.

## Mechanics

The most widely deployed recipe is Chronos-style mean scaling plus uniform binning:

```
# x: (T,) context window for one series
s      = mean(|x|) + eps              # per-series scale
x_tilde = clip(x / s, -C, C)          # normalize and clip to a fixed range [-C, C]
bin    = floor((x_tilde + C) / (2C / V)) # uniform bucket in {0, ..., V-1}
token  = bin + K_special              # shift to leave room for <pad>/<eos>/...
```

With `V ≈ 4096` the resulting token sequence is fed to an unmodified T5 as if it were text. Training is standard cross-entropy over the vocabulary. At inference the decoder samples `M` trajectories of token ids; each token is dequantized to its bin center (or to a learned continuous representative), rescaled by `s`, and the `M` trajectories form the empirical predictive distribution from which any quantile can be read.

A learned variant uses a VQ-VAE. A small encoder maps a patch `p \in R^P` to a latent `z \in R^d`, which is snapped to its nearest codebook vector `c_k` before being decoded back to `\hat p`. The codebook is trained jointly with a reconstruction loss and a commitment loss:

```
L = || p - decode(c_k) ||^2 + beta * || z - stop_grad(c_k) ||^2
```

The resulting codebook indices `k \in {1,...,K}` are then the discrete "tokens" of the TS transformer. Unlike uniform binning, each token corresponds to a *prototype waveform*, so a single token carries multi-step shape information.

A third extreme: no TS-specific quantizer at all. `[LLMTime](../papers/llmtime.md)` serializes numbers as ASCII digit strings (with explicit sign, decimal, and digit-grouping tokens), feeds them to a vanilla GPT-3 or Llama-2, and decodes generated digit strings back to reals. The LLM's BPE becomes the implicit quantizer; probability mass on digit positions is converted into a continuous density via a discrete-to-continuous transform.

## Why it works

Three things make quantization-as-tokens effective. First, it converts an unbounded regression target into a closed classification target, which stabilizes optimization: softmax-plus-cross-entropy has better-conditioned gradients than MSE over a variable-scale target. Second, the model automatically becomes probabilistic — sampling gives joint trajectories, and autoregressive factorization plus sampling is a principled Monte-Carlo approximation to the predictive posterior, with no need to commit to a parametric family (Student-t, Gaussian, mixture). Third, it is architecture-agnostic: any transformer, SSM, or MoE that can do next-token prediction becomes a TS forecaster for free, unlocking the entire LM toolchain (distributed training, KV cache, beam search, speculative decoding).

It also turns out to be a surprisingly good fit for TS distributional structure. Most real series have multimodal, skewed, heavy-tailed predictive distributions at long horizons, and categorical sampling captures those shapes with no tail assumption.

## Trade-offs and failure modes

The most visible cost is *precision ceiling*. A 4096-bin uniform quantizer over `[-15s, +15s]` has bin width `~0.007s`; any phenomenon finer than that is smeared. If the bins are set too narrow, tails get clipped; too wide, and the model wastes representation capacity on barely-used buckets. Mean scaling is fragile for intermittent series (many zeros) and for series whose mean drifts across the window. The vocabulary must be fixed at pretraining time, so a shift in scale at deployment — say a trailing mean near 0 — collapses the effective range.

VQ-VAE codebooks face the classic problem of *codebook collapse*: a few codes dominate, the rest are unused, and the effective vocabulary shrinks. Commitment-loss tuning and code reinitialization are the usual remedies but add complexity. LLMTime's digit tokenization is expensive in token count (each number costs several tokens) and depends heavily on the base LLM's idiosyncratic BPE.

## Design choices in the literature

- `[Chronos](../papers/chronos.md)` uses mean-scaling plus uniform binning into a fixed vocabulary (a few thousand bins), trains an unmodified T5 at 20M–710M sizes with CE loss, and samples for probabilistic output.
- `[Chronos-2](../papers/chronos-2.md)` drops the vocabulary entirely in favor of a quantile decoder, arguing that direct quantile regression removes the precision ceiling while keeping the probabilistic interface.
- `[TOTEM](../papers/totem.md)` learns a *cross-domain* VQ-VAE codebook and compares "specialist" (one codebook per domain) vs "generalist" (one shared codebook) settings — directly probing whether tokenization should be domain-conditional.
- `[LLMTime](../papers/llmtime.md)` uses the base LLM's BPE as the implicit quantizer and converts the token-level distribution over digit strings back to a continuous density for calibrated intervals.
- `[MTS-JEPA](../papers/mts-jepa.md)` introduces a **soft codebook** that maps continuous encoder features to a probability distribution over `K` learnable prototypes via temperature-scaled cosine similarity. Unlike Chronos / TOTEM, the codebook is not used as the *output* token vocabulary — it is an internal bottleneck that turns latent prediction into KL alignment between simplex distributions, and the bounded convex-hull geometry doubles as an analytical non-collapse certificate for JEPA self-distillation. Useful contrast: a codebook can be a *latent geometric constraint* rather than a tokenizer.

## Open questions

- **Does the quantizer need to be learned, or is uniform binning sufficient?** Chronos's uniform recipe is strong; Chronos-2 argues quantile regression is better; TOTEM argues codebooks help cross-domain transfer. Not resolved.
- **How fine should the vocabulary be?** There is no systematic sweep pinning down the precision-vs-sample-efficiency tradeoff in the 1k–100k range.
- **Scale-invariant tokenization.** All current recipes rely on per-window scaling that breaks for non-stationary or intermittent series. A scale-invariant discrete representation is open.
- **Joint quantization across channels.** Current tokenizers are per-channel; whether a joint codebook across variates could capture cross-series structure is untried.
- **Can the quantizer be trained end-to-end with the transformer?** Stop-gradient tricks (straight-through, Gumbel) exist but have not been convincingly applied to foundation-scale TS.

## Papers that exemplify this

- `[Chronos](../papers/chronos.md)` — mean-scaling + uniform binning + unmodified T5 CE training, probabilistic via categorical sampling and dequantization.
- `[Chronos-2](../papers/chronos-2.md)` — abandons vocabulary tokens in favor of direct quantile decoding, a refined position in the same design space.
- `[TOTEM](../papers/totem.md)` — VQ-VAE cross-domain codebook, explicit specialist-vs-generalist comparison.
- `[LLMTime](../papers/llmtime.md)` — numbers-as-text tokenization with discrete-to-continuous density, zero-shot via vanilla LLMs.
- `[MTS-JEPA](../papers/mts-jepa.md)` — soft-codebook bottleneck inside a JEPA loop; discrete geometry doubles as a non-collapse certificate (paper Appendix A.3). Codebook as *constraint*, not as output vocabulary.

## Related wiki pages

- [Patch tokenization](patch-tokenization.md)
- [Probabilistic forecasting](probabilistic-forecasting.md)
- [Zero-shot forecasting](zero-shot-forecasting.md)
- [Encoder-decoder T5](../architectures/encoder-decoder-t5.md)
- [LLM reprogramming](../architectures/llm-reprogramming.md)
