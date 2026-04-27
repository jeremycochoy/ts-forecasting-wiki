# In-Context Learning

## Intuition

In-context learning (ICL) lets a pretrained model adapt its predictions from examples placed in the prompt — with *no* gradient updates. In language modeling, this arises naturally from left-to-right autoregression: once the model has seen a few "Q: ... A: ..." pairs, it continues the pattern. For time-series foundation models, ICL is the bridge between pure zero-shot and full fine-tuning: whenever related series, covariates, or small panels exist, the model should *use them at inference* instead of requiring a training loop. That ability turns a single checkpoint from a one-series-at-a-time forecaster into a flexible, context-aware predictor.

## Mechanics

There are two recognizable architectural strategies in current TS foundation models.

**Group attention (Chronos-2).** The input is a set of series arranged as a panel: the target series plus `K` related series (other variates, exogenous covariates, historically-similar siblings). Each is patched and embedded separately, then concatenated into one long sequence. A custom attention mask — *group attention* — lets each token attend freely within its own series along the temporal axis (typically causal) and across series at the same time index:

```
# S = number of series in the group, N = patches per series, D = hidden
tokens = patches.reshape(B, S*N, D)    # flatten panel into one sequence
mask   = build_group_mask(S, N)        # within-series causal + cross-series full
z      = Transformer(tokens, mask)
target_out = decoder_head(z[:, :N])    # read out the target series' positions
```

The mask is the whole mechanism: it encodes "tokens at the same timestamp of any series can exchange information, tokens at later times cannot peek at earlier tokens in the causal direction." Chronos-2's 120M encoder-only stack is trained end-to-end with this mask, so it learns to use the panel context when it reduces the target's quantile loss and to ignore it otherwise.

**Universal TimeAttention (Timer-XL).** The decoder-only side realizes ICL differently. A multivariate panel is *flattened* into a single sequence of patch tokens, with position embeddings that encode both time index and variate index. A causal mask that respects the 2D layout (variates at the same time can see each other; no peeking forward in time) lets one transformer handle univariate, multivariate, and covariate-conditioned forecasting uniformly:

```
for v in variates:
    tokens_v = patch_embed(series_v) + posemb(time, v)
seq = concatenate(tokens_v for v in variates)   # (B, V*N, D)
out = CausalTransformer(seq)
```

Because the same architecture sees any variate layout at training time, it generalizes to previously unseen variate counts and roles at inference — a form of ICL at the layout level.

## Why it works

The intellectual core of ICL is that a sufficiently expressive sequence model, trained on heterogeneous data where useful signal often sits in *siblings* rather than just history, learns an implicit "use the sibling" circuit. Attention is the natural implementation: a query from the target series can match keys from sibling series and retrieve their values, exactly as in NLP where a question token retrieves the right answer token from demonstrations. The group-attention mask is not a new mechanism, it is a *structural prior* that bakes the sibling-retrieval pattern into attention.

From a Bayesian perspective, ICL is amortized posterior inference: the pretraining objective teaches the model to output `p(y | history, siblings)` for whatever sibling configuration appears, and at inference the model evaluates that conditional for the user's panel. Unlike fine-tuning, the "update" is instantaneous and reversible because it lives entirely in the forward pass.

The reason this matters for TS specifically is that related panels are everywhere in practice: a retailer's SKUs, a grid's substations, a building's rooms. Fine-tuning a separate model per SKU is infeasible; feeding the SKU plus its peers as context is cheap and preserves the zero-shot deployment story.

## Trade-offs and failure modes

ICL is not free. The group attention cost is `O((SN)^2 D)`, so stacking many related series blows up the context quickly. Practical implementations cap the panel at a few dozen siblings. ICL also *requires* matching sibling distributions: if the context series have genuinely different dynamics from the target, attention can either ignore them (best case) or be misled by spurious correlations (worst case). Training the model to be *robust* to useless context — equivalent to LLMs being asked to ignore distractors — is an open problem.

The training objective matters a lot. If pretraining only sees single series, the model never learns to use a panel; you need pretraining batches that explicitly include related context with a signal the model can exploit, which is why Chronos-2 relies heavily on synthetic augmentation to generate sibling structure at scale.

## Design choices in the literature

- `[Chronos-2](../papers/chronos-2.md)` — 120M encoder-only patch transformer with an explicit group-attention mask; handles univariate, multivariate, covariate, and panel forecasting uniformly through structured masking.
- `[Timer-XL](../papers/timer-xl.md)` — flattens the multivariate panel into one causal sequence and uses universal TimeAttention with 2D positional embeddings, generalizing to unseen variate layouts.
- `[Moirai](../papers/moirai.md)` any-variate attention is a spiritual predecessor: a masked encoder that accepts variable variate counts without architectural changes, but without explicit sibling-context training.
- `[UniTS](../papers/units.md)` — task tokens prepended to the input implement a different flavor of ICL at the *task* axis rather than the *series* axis.
- `[LaT-PFN](../papers/lat-pfn.md)` — the Bayesian-amortization formulation explicitly: the predictor is a PFN transformer that approximates the posterior predictive `p(y|x, D)` over a context dataset `D` of related synthetic series, and the model is trained on heterogeneous synthetic contexts so the in-context circuit emerges by design rather than by accident. Context curation is treated as the test-time analog of LLM prompt engineering.

## Open questions

- **Does a few-shot prompt work like NLP?** TS rarely has natural "demonstration" series the same way NLP has Q/A pairs; whether demonstration-style prompting yields the same emergent behavior is open.
- **How many siblings are useful?** There is no clear scaling law relating panel size to accuracy gain.
- **Context poisoning.** What happens when siblings are adversarial or mismatched? Robust ICL in TS has not been studied rigorously.
- **Channel independence vs ICL.** Channel-independent models (PatchTST, [MOMENT](../papers/moment.md)) cannot do sibling-based ICL at all; is the lost information worth the simplicity?
- **Relationship to retrieval.** Is explicit retrieval of similar historical windows (memory-augmented forecasting) competitive with architectural ICL?

## Papers that exemplify this

- `[Chronos-2](../papers/chronos-2.md)` — the clearest example of architectural ICL in TS: group attention lets multivariate, covariate, and panel context propagate to the target series in a single forward pass.
- `[Timer-XL](../papers/timer-xl.md)` — decoder-only ICL via flattened multivariate sequences and 2D causal position embeddings, with emergent generalization to new variate layouts.
- `[MOIRAI](../papers/moirai.md)` — any-variate attention as a predecessor that supports variable variate counts without dedicated ICL training.
- `[UniTS](../papers/units.md)` — task-token conditioning as a different ICL axis.
- `[LaT-PFN](../papers/lat-pfn.md)` — the Bayesian-amortized formulation, with PFN cross-attention from the target prompt to a learned average-pooled summary of related context series; "context curation" framed as TS-side prompt engineering.

## Related wiki pages

- [Zero-shot forecasting](zero-shot-forecasting.md)
- [Multi-task universal](multi-task-universal.md)
- [Patch tokenization](patch-tokenization.md)
- [Encoder-decoder T5](../architectures/encoder-decoder-t5.md)
- [Decoder-only autoregressive](../architectures/decoder-only-autoregressive.md)
