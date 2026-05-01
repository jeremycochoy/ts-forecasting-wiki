# Timer-XL: Long-Context Transformers for Unified Time Series Forecasting

> **Short name:** `timer-xl` · **arXiv:** [2410.04803](https://arxiv.org/abs/2410.04803) · **PDF:** [local](../../papers/timer-xl_2410.04803.pdf) · **Date:** 2024-10 · **Venue:** ICLR 2025

**Authors:** Yong Liu, Guo Qin, Xiangdong Huang, Jianmin Wang, Mingsheng Long (Tsinghua)

## Abstract
Timer-XL extends Timer's univariate generative pretraining to the multivariate setting by generalizing next-token prediction from 1D sequences to 2D time series, and introducing a universal TimeAttention mechanism with causal positional embeddings. The result is a long-context decoder-only transformer that handles univariate, multivariate, and covariate-informed forecasting in one formulation.

## Key contributions
- Multivariate next-token prediction: each patch token is indexed by (variable, time), and the causal mask is a Kronecker product of a temporal causal mask and a learnable variable-adjacency mask.
- TimeAttention: a single self-attention module that captures both intra-series and inter-series dependencies while remaining causal in time and permutation-equivariant in variables.
- Specialized position embeddings that distinguish endogenous and exogenous variables and maintain variable equivalence.
- Context length extended from Timer's ~720 tokens to thousands of patch tokens, enabling long-context pretraining on ERA5-Large (2880-token context).
- Pretraining on UTSD and [LOTSA](../datasets-benchmarks/lotsa.md) (univariate) plus ERA5-Large (multivariate), producing an 84M checkpoint released as `thuml/timer-base-84m`.

## Architecture at a glance
Timer-XL keeps Timer's decoder-only backbone and next-token MSE training objective but flattens multivariate patches into a 2D (N x T) token grid. TimeAttention lets each token attend to all variables at earlier or equal time steps via a Kronecker-structured mask, preserving causality along the time axis while allowing free attention across variates at the same or earlier times. With FlashAttention, the memory footprint is O(N*T) rather than the naive O(N^2 T^2). Input and output patch size is 96; forecasting is autoregressive rollout with the multivariate flattened-token format, using universal TimeAttention with causal positional embeddings (Liu et al. Table 11).

### Sizes (Liu et al. Table 11; FFN hidden dim = 4D per §A.2)

| Variant | Layers (L) | d_model (D) | d_ff | Heads (H) | d_kv | Patch (P) | Params | Context |
|---|---|---|---|---|---|---|---|---|
| Timer-XL pre-train (released `thuml/timer-base-84m`) | 8 | 1024 | 4096 | 8 | 128 | 96 | 84M | up to 3072 |
| Timer-XL univariate fine-tune | 3 | 512 | 2048 | 8 | 64 | 96 | — | depends on dataset |
| Timer-XL multivariate fine-tune (varies) | 1–6 | 512–1024 | 2048–4096 | 8 | 64–128 | 96 | — | up to 720 |

Same recipe at the multi-task pre-training scale: the released model checkpoint corresponds to the L=8 / D=1024 row.

## Why it matters
Timer-XL shows that long-context decoder-only transformers can absorb multivariate and covariate structure without bolt-on components, unifying a broader class of forecasting problems than univariate TS foundation models. It is one of the clearest demonstrations that causal channel-dependent attention can be scaled efficiently.

## Strengths
- Kronecker-factored attention mask is a clean answer to "how do you stay causal across the time axis while freely attending across variables?" and it comes with an explicit theoretical complexity analysis (Table 8) plus practical FlashAttention integration.
- Direct ablation shows encoder-only multivariate forecasters (such as UniTST, iTransformer) degrade at long context while the decoder-only Timer-XL does not. This is one of the few published head-to-head empirical signals on encoder-vs-decoder for long-context TS.
- Establishes new long-context multivariate benchmarks (ERA5-Large with 4920 stations, ~2880-token context) because the paper argues [Monash](../datasets-benchmarks/monash-archive.md)/ETT/Weather are too short to exercise the context-length advantage. This is constructive benchmark-building rather than just running on existing suites.
- Zero-shot Table 7 shows the 84M Timer-XL-Base matches or beats Moirai-Large (311M), [TimesFM](./timesfm.md), and [MOMENT](./moment.md) on most ETT/ECL/Weather horizons, suggesting long-context pretraining is more parameter-efficient than scaling dense parameters.
- Explicitly separates endogenous from exogenous variables via learned position embeddings, which gives Timer-XL native covariate support that vanilla Timer lacks.

## Limitations and open critiques
- Still point-prediction only: no probabilistic head. For [CRPS](../evaluation/metrics.md#21-crps--continuous-ranked-probability-score) / [WQL](../evaluation/metrics.md#23-wql--weighted-quantile-loss) comparisons against [chronos](./chronos.md), [lag-llama](./lag-llama.md), [sundial](./sundial.md), or [chronos-2](./chronos-2.md), Timer-XL has to rely on post-hoc quantile fitting.
- Single released size (84M Base) in the main results; the paper does not produce a parameter-scaling curve for TimeAttention and so does not establish whether long-context pretraining remains beneficial at 300M-1B scale.
- ERA5-Large is a single-domain (weather) long-context corpus; generalization from ERA5 pretraining to non-spatial multivariate panels such as retail or finance is not audited.
- The same team's [timer](./timer.md) pretraining datasets (UTSD) overlap with parts of the downstream evaluation suite; the paper marks pretrained datasets in Table 7 but does not audit the overlap at the sub-series level — see [../research/reproducibility.md](../research/reproducibility.md).
- Kronecker mask assumes a fixed variable adjacency pattern per task, and the paper does not explicitly ablate how sensitive performance is to the choice of variable ordering or to missing variates at inference.
- Memory scaling, while reduced via FlashAttention to O(N*T), still makes very high-dimensional panels (thousands of variates) expensive; [chronos-2](./chronos-2.md) argues the group-attention formulation is more memory-efficient in the O(V) regime.

## Follow-up work and dialogue
Timer-XL is the explicit multivariate successor to [timer](./timer.md), citing channel-independence as its motivation. It argues against the encoder-only multivariate line, represented by [moirai](./moirai.md) (any-variate attention) and UniTST, that flattening plus bidirectional attention degrades at long context. [chronos-2](./chronos-2.md) responds by introducing group attention, which scales as O(V) rather than O(V*T) and adds [in-context learning](../concepts/in-context-learning.md) over related series. On the decoder-only MoE axis, [time-moe](./time-moe.md) and [moirai-moe](./moirai-moe.md) pursue sparsity as a scaling lever instead of long context. [sundial](./sundial.md) reuses a Timer-style backbone but swaps the point head for flow-matching, staking out the generative-distribution alternative. See [../architectures/decoder-only-autoregressive.md](../architectures/decoder-only-autoregressive.md) for where Timer-XL sits.

## Reproducibility
- **Open weights:** yes — `thuml/timer-base-84m` on HuggingFace (quoted directly in the paper).
- **Code:** public at `thuml/Timer-XL`.
- **Training data:** partially public — UTSD and LOTSA are open; ERA5-Large is derived from public ERA5 reanalysis data and the splits (80% stations / 80% time) are described but exact release depends on the repository.
- **Compute to retrain:** not fully disclosed in the main text; pretraining context length is 2880 tokens for the ERA5-Large run versus 1440 for vanilla Timer.
- **Deployment footprint:** 84M parameters for Timer-XL-Base (the main released checkpoint); multivariate inference complexity O(N*T) with FlashAttention; demonstrated on standard GPUs.

## When to cite this paper
Cite Timer-XL as the canonical reference for multivariate next-token prediction via a Kronecker-structured causal TimeAttention, and for the demonstration that decoder-only long-context pretraining is a viable route to unified univariate / multivariate / covariate-informed forecasting. It is also the right citation for the specific claim that encoder-only multivariate transformers degrade at long context while causal decoder-only transformers do not.

## In the knowledge graph
- **Cluster:** [Decoder-only autoregressive TS-FMs](../foundation-models/taxonomy.md#cluster-1--decoder-only-autoregressive-ts-fms)
- **Architecture family:** [Decoder-only autoregressive](../architectures/decoder-only-autoregressive.md)
- **Related concepts:** [patch tokenization](../concepts/patch-tokenization.md), [multi-task universal](../concepts/multi-task-universal.md), [zero-shot forecasting](../concepts/zero-shot-forecasting.md)
- **See also:** [timer](./timer.md), [chronos-2](./chronos-2.md), [sundial](./sundial.md)
