# Model Sizing Cheat Sheet

Quick reference for picking a `(hidden_dim, num_layers)` combination that
lands a decoder-only transformer in a specific parameter-count bracket,
and for sanity-checking the size of a new model against the smallest
SOTA time-series foundation models. Companion to
[training-a-small-model.md](training-a-small-model.md), which covers
*where* to train; this page covers *how big* to build.

## The bracket table, enriched with (d, L) recommendations

The bracket rows are the same ones used in
[training-a-small-model.md](training-a-small-model.md), with an added
column of decoder-only `(d, L)` combinations that land inside each
bracket. Bold entries are the cleanest round-number fit. Representative
SOTA param counts come from the paper leaves; `—` means the paper does
not disclose the number.

| Bracket | Range | Representative SOTA | Decoder-only `(d, L)` | Actual model configs | Hardware tier |
|---|---|---|---|---|---|
| Tiny | 1–10M | [TTM](../papers/ttm.md) (1–5M, MLP-Mixer), [Lag-Llama](../papers/lag-llama.md), [Mamba4Cast](../papers/mamba4cast.md) | `d=256, L=4` → 3.1M · `d=256, L=8` → 6.3M · `d=384, L=4` → 7.1M — note: at `d=512` a single layer is already 3.1M, so 2 layers overshoot Tiny's upper end when combined with embeddings | Lag-Llama `d=144, L=8, H=9` (2.45M) · TTM 1–5M (MLP-Mixer, no `d/L`) · Mamba4Cast `d=1024, 2 SSM blocks` (27M, SSM not attention) | Single consumer GPU, CPU inference realistic |
| Small | 10–50M | [Chronos-Mini](../papers/chronos.md) (20M), [MOIRAI-Small](../papers/moirai.md) (14M), [MOMENT-Small](../papers/moment.md) (40M) | **`d=512, L=4`** → 12.6M · `d=512, L=8` → 25.2M · `d=768, L=4` → 28.3M · `d=512, L=12` → 37.7M | MOIRAI-Small `d=384, L=6, H=6` (14M) · TimesFM-17M `d=512, L=10, H=16` · Chronos-Mini `d=384, L=4+4 enc-dec, H=8` (20M) · Sundial-Small `d=512, L=6, H=8` (32M) · MOMENT-Small `d=512, L=6, H=8` (40M) · Chronos-Small `d=512, L=6+6, H=8` (46M) | Single 4090 / A100 40GB |
| **Small ceiling** | **40–50M** | **[Chronos-Small](../papers/chronos.md) (46M), [MOMENT-Small](../papers/moment.md) (40M)** | **`d=768, L=6`** → **42.5M** (cleanest fit) · `d=512, L=16` → 50.3M · avoid `d=1024, L=3` (37.7M, too shallow) | MOMENT-Small `d=512, L=6` (40M, masked encoder) · Chronos-Small `d=512, L=6+6 enc-dec` (46M) · Timer 50M `d=1024, L=6, H=8` | Single A100 40GB |
| Base | 50–200M | [Chronos-2](../papers/chronos-2.md) (120M), [MOMENT-Base](../papers/moment.md) (125M), [MOIRAI-Base](../papers/moirai.md) (91M), [Timer](../papers/timer.md) | `d=768, L=8` → 56.6M · `d=768, L=12` → 84.9M · **`d=1024, L=8`** → **100.7M** · `d=768, L=16` → 113.2M · `d=1024, L=12` → 151.0M | TimesFM-70M `d=1024, L=10, H=16` · Timer-XL-Base `d=1024, L=8, H=8` (84M) · MOIRAI-Base `d=768, L=12, H=12` (91M) · Sundial-Base `d=768, L=12, H=12` (128M) · MOMENT-Base `d=768, L=12, H=12` (125M) · Chronos-2 `d=—, L=—` (120M, not tabulated) | Single H100 / A100 80GB |
| **Base ceiling** | **~200M** | **[TimesFM](../papers/timesfm.md) (~200M), [Chronos-Base](../papers/chronos.md) (200M)** | **`d=1024, L=16`** → **201.3M** (matches TimesFM almost exactly) | TimesFM-200M `d=1280, L=20, H=16, d_ff=1280` (FFN=d, not 4d) · Chronos-Base `d=768, L=12+12 enc-dec, H=12` (200M) | Single H100 |
| Large | 200M–1B | [Chronos-Large](../papers/chronos.md) (710M), [MOMENT-Large](../papers/moment.md) (385M), [MOIRAI-Large](../papers/moirai.md) (311M), [Timer-XL](../papers/timer-xl.md) | `d=1024, L=24` → 302M · `d=1280, L=24` → 472M · `d=1536, L=24` → 680M | MOIRAI-Large `d=1024, L=24, H=16` (311M) · MOMENT-Large `d=1024, L=24, H=16` (385M) · Sundial-Large `d=1024, L=24, H=16` (444M) · Chronos-Large `d=1024, L=24+24 enc-dec, H=16` (710M) · Time-MoE-Large `d=768, L=12, H=12` (200M active / 453M total) | 4–8× datacenter GPUs |
| Huge | 1B+ | [Time-MoE](../papers/time-moe.md) (2.4B), [Timer-S1](../papers/timer-s1.md) (8.3B) | At this scale, a sparse MoE (activated subset of experts per token) beats a dense transformer of equal total size; dense equivalents would need `d=2048, L=24+`. | Time-MoE-Ultra `d=1024, L=36, H=16` (1.1B active / 2.4B total, 2/8 experts) · Timer-S1 `d=1024, L=24 TimeMoE + 16 TimeSTP, H=16` (0.75B active / 8.3B total, 2/32 experts) | Cluster |

## Formula

For a decoder-only transformer with hidden dim `d`, `L` layers, standard
multi-head attention, and 4× FFN expansion:

- Attention block (Q, K, V, O projections): `4·d²`
- FFN block (two matrices `d → 4d → d`): `8·d²`
- **Per layer: `12·d²`**
- **For `L` layers: `12·L·d²`**

Small additive overhead:

- LayerNorms + biases: ~2–5% of core, negligible
- Patch projection (`patch_size · d`): a few thousand params
- Positional embeddings: 0 with RoPE, ~`(max_seq · d)` with learned
  absolute (≈ 0.5M at `d=1024, max_seq=512`)
- Quantized value vocabulary ([Chronos](../papers/chronos.md)-style, `V ≈ 4096`):
  add ~`2·V·d` for input embed + output logits, often tied. At
  `d=1024` that is ~8M; at `d=512` it is ~4M.

For a pure patch-in / continuous-regression-out TS-FM the "core = total"
approximation from `12·L·d²` is tight to within a few percent. Add the
overhead numbers above only if you're using a discrete vocabulary or
learned absolute positional embeddings.

## Parameter grid

Exact core-only params for the three most common TS-FM hidden sizes:

| L \ d | d=512 | d=768 | d=1024 |
|---|---|---|---|
| 2 | 6.3M | 14.2M | 25.2M |
| 3 | 9.4M | 21.2M | 37.7M |
| 4 | 12.6M | 28.3M | 50.3M |
| 6 | 18.9M | 42.5M | 75.5M |
| 8 | 25.2M | 56.6M | 100.7M |
| 10 | 31.5M | 70.8M | 125.8M |
| 12 | 37.7M | 84.9M | 151.0M |
| 16 | 50.3M | 113.2M | 201.3M |
| 20 | 62.9M | 141.6M | 251.7M |
| 24 | 75.5M | 169.9M | 302.0M |
| 32 | 100.7M | 226.5M | 402.7M |

## Actual configurations used in the literature

Below are the `(d, L)` configurations that published TS foundation
models actually use, sourced from each paper's method section or
appendix. Where a paper ships multiple sizes, every size is listed.
Values marked "—" are not disclosed in the paper. `L` is the number
of transformer blocks (decoder-only or encoder-only); for T5-style
encoder-decoder models it is reported as `enc + dec`. `d_ff` is the
FFN inner dimension (usually 4·d). For Chronos the paper does not
print `(d, d_ff, heads, L)` inline — the values below come from the
HuggingFace `google/t5-efficient-{mini,small,base,large}` configs
that Chronos loads.

### Decoder-only transformers

| Model | Size | d | L | Heads | d_ff | Patch in | Context | Params | Source |
|---|---|---|---|---|---|---|---|---|---|
| [TimesFM](../papers/timesfm.md) | 17M | 512 | 10 | 16 | 512 | 32 | — | 17M | Das et al. Table 6 (App. A.6) |
| [TimesFM](../papers/timesfm.md) | 70M | 1024 | 10 | 16 | 1024 | 32 | — | 70M | Das et al. Table 6 (App. A.6) |
| **[TimesFM](../papers/timesfm.md)** | **200M** | **1280** | **20** | **16** | **1280** | **32** | — | **200M** | **Das et al. Table 6 (App. A.6)** |
| [Timer](../papers/timer.md) | 29M | 768 | 6 | 8 | 1536 | 96 | ≤1440 | 29M | Liu et al. Table 14 (App.) |
| [Timer](../papers/timer.md) | 50M | 1024 | 6 | 8 | 2048 | 96 | ≤1440 | 50M | Liu et al. Table 14 (App.) |
| [Timer](../papers/timer.md) | 67M | 1024 | 8 | 8 | 2048 | 96 | ≤1440 | 67M | Liu et al. Table 14 (App.) |
| [Timer-XL](../papers/timer-xl.md) | Base 84M | 1024 | 8 | 8 | 4096 | 96 | ≤2880 | 84M (`timer-base-84m`) | Liu et al. Table 11 |
| [Lag-Llama](../papers/lag-llama.md) | — | 144 | 8 | 9 | — | n/a (lag-feature) | 32 | 2.45M | Rasul et al. Table 5 (App. D) |
| [Sundial](../papers/sundial.md) | Small | 512 | 6 | 8 | 2048 | 16 | 2880 | 32M | Liu et al. Table 5 |
| [Sundial](../papers/sundial.md) | Base | 768 | 12 | 12 | 3072 | 16 | 2880 | 128M | Liu et al. Table 5 |
| [Sundial](../papers/sundial.md) | Large | 1024 | 24 | 16 | 4096 | 16 | 2880 | 444M | Liu et al. Table 5 |

Checks against the `12·L·d²` core:

- TimesFM 200M: `12 × 20 × 1280² ≈ 393M` core, but TimesFM sets the
  FFN hidden dim equal to `d` (not `4d`) per Appendix A.6, so per-layer
  cost is `≈ 6·d²` not `12·d²`, giving `≈ 197M` — matches the 200M
  headline.
- Timer-XL 84M: `12 × 8 × 1024² ≈ 100M` core — within the published
  84M once the non-4× details are counted.
- Sundial-Base 128M: `12 × 12 × 768² ≈ 85M` core plus flow-matching
  head and embeddings — consistent with 128M.

### Encoder-decoder (T5-family)

For these rows, `L` is reported as `encoder + decoder` and `d_ff` is
the T5 inner dim (often not `4·d`). The Chronos paper does not list
`(d, L, d_ff, heads)` per size inline; the values below are the
official T5-Efficient configs (`google/t5-efficient-{tiny,mini,small,
base,large}` on HuggingFace) that the Chronos paper says it uses.

| Model | Size | d | L (enc+dec) | Heads | d_ff | Patch/token | Context | Params | Source |
|---|---|---|---|---|---|---|---|---|---|
| [Chronos](../papers/chronos.md) | Mini | 384 | 4 + 4 | 8 | 1536 | scalar token (V≈4096) | 512 | 20M | Ansari et al. §5.2; HF `t5-efficient-mini` |
| [Chronos](../papers/chronos.md) | Small | 512 | 6 + 6 | 8 | 2048 | scalar token | 512 | 46M | Ansari et al. §5.2; HF `t5-efficient-small` |
| [Chronos](../papers/chronos.md) | Base | 768 | 12 + 12 | 12 | 3072 | scalar token | 512 | 200M | Ansari et al. §5.2; HF `t5-efficient-base` |
| [Chronos](../papers/chronos.md) | Large | 1024 | 24 + 24 | 16 | 4096 | scalar token | 512 | 710M | Ansari et al. §5.2; HF `t5-efficient-large` |
| [Chronos](../papers/chronos.md) | (GPT-2 variant) | 768 | 12 | 12 | 3072 | scalar token | 512 | ≈90M | Ansari et al. §5.2 |

The Chronos paper names its models Mini 20M / Small 46M / Base 200M
/ Large 710M — **there is no "Chronos-Tiny"** despite the label in
some third-party tables. The sub-20M t5-efficient-tiny config
(`d=256, L=4+4, d_ff=1024, heads=4`, ~16M) is mentioned in the
Chronos paper as a reference point but no Chronos model is trained
at that size.

### Masked encoder / encoder-only

| Model | Size | d | L | Heads | d_ff | Patch in | Context | Params | Source |
|---|---|---|---|---|---|---|---|---|---|
| [MOMENT](../papers/moment.md) | Small | 512 | 6 | 8 | 2048 | 8 | 512 | 40M | Goswami et al. §5 (Pre-training Setup) |
| [MOMENT](../papers/moment.md) | Base | 768 | 12 | 12 | 3072 | 8 | 512 | 125M | Goswami et al. §5 |
| [MOMENT](../papers/moment.md) | Large | 1024 | 24 | 16 | 4096 | 8 | 512 | 385M | Goswami et al. §5 |
| [MOIRAI](../papers/moirai.md) | Small | 384 | 6 | 6 | 1536 | variable (freq-dependent) | ≤5000 | 14M | Woo et al. Table 4 |
| [MOIRAI](../papers/moirai.md) | Base | 768 | 12 | 12 | 3072 | variable | ≤5000 | 91M | Woo et al. Table 4 |
| [MOIRAI](../papers/moirai.md) | Large | 1024 | 24 | 16 | 4096 | variable | ≤5000 | 311M | Woo et al. Table 4 |
| [Chronos-2](../papers/chronos-2.md) | (main) | — | — | — | — | patch (P not inline) | — | 120M | Shchur et al. §3.2 (only "120M" and "28M ablation" disclosed; d/L/heads not tabulated in the report) |

MOMENT is built on T5 encoder weights; the configs match
`google/t5-efficient-{small,base,large}` scaled to encoder-only
(hence ~half of Chronos's params at the same size name). MOIRAI
uses the same `(d, L)` grid as T5 but with `d_kv=64` and a
mixture-of-Student-t distribution head.

### Mixture-of-experts

For MoE rows, `params` is split into `activated / total`. The
"equivalent dense" column gives the `12·L·d²` core computed with
`d_ff = 4·d`, i.e. the dense cost of one replica of the expert FFN,
for comparison against the bracket table above.

| Model | Size | d | L | Heads | d_ff (expert) | Experts (active/total) | Activated / Total params | Equivalent dense core | Source |
|---|---|---|---|---|---|---|---|---|---|
| [Time-MoE](../papers/time-moe.md) | Base | 384 | 12 | 12 | 1536 (192/expert) | 2 / 8 | 50M / 113M | 21M | Shi et al. Table 2 |
| [Time-MoE](../papers/time-moe.md) | Large | 768 | 12 | 12 | 3072 (384/expert) | 2 / 8 | 200M / 453M | 85M | Shi et al. Table 2 |
| [Time-MoE](../papers/time-moe.md) | Ultra | 1024 | 36 | 16 | 4096 (512/expert) | 2 / 8 | 1.1B / 2.4B | 453M | Shi et al. Table 2 |
| [Moirai-MoE](../papers/moirai-moe.md) | Small | 384 | 6 | — | 512 (per expert) | 2 / 32 | 11M / 117M | — | Liu et al. Table 1 |
| [Moirai-MoE](../papers/moirai-moe.md) | Base | 768 | 12 | — | 1024 (per expert) | 2 / 32 | 86M / 935M | — | Liu et al. Table 1 |
| [Timer-S1](../papers/timer-s1.md) | (only size) | 1024 | 24 TimeMoE + 16 TimeSTP | 16 | — | 2 / 32 | 0.75B / 8.3B | — | Liu et al. §4.2 |

Time-MoE Base runs with 50M activated params but its dense-equivalent
core is only ~21M — the "50M activated" number includes the shared
expert and routing overhead. This is why MoE configs look oversized
relative to a dense `(d, L)` plug into `12·L·d²`.

### Non-transformer and LLM-reprogramming

The following models do not fit the `12·L·d²` formula. Listed for
completeness with the nearest-equivalent architectural metric.

- [TTM](../papers/ttm.md): TSMixer MLP-Mixer backbone, not
  self-attention. Three sizes: `TTMB` 1M (context 512, patch 64),
  `TTME` 4M (context 1024, patch 128), `TTMA` 5M (context 1536, patch
  128). The paper does not tabulate `(d, L)` for the mixer blocks in
  the main text. `12·d²` does not apply.
- [Mamba4Cast](../papers/mamba4cast.md): Mamba2 state-space model,
  not a transformer. `d_model = 1024`, state expansion factor
  `N = 128`, 2 stacked Mamba2 encoder blocks (Figure 1), ~27M total.
  The `d²` term in the cost is SSM state × d, not attention.
- [Time-LLM](../papers/time-llm.md): frozen **Llama-7B** backbone
  (`d=4096, L=32, 32 heads`) by default; only a reprogramming layer
  on top of the LLM is trained. None of the backbone params are
  updated.
- [GPT4TS / "One-Fits-All"](../papers/gpt4ts.md): frozen **GPT-2
  base** first 6 layers (`d=768, L=6, heads=12`); only positional
  embeddings and LayerNorm are fine-tuned. The paper denotes this
  configuration as "GPT2(6) FPT."
- [LLMTime](../papers/llmtime.md): frozen **GPT-3** or **Llama-2**
  used via prompt-level numerical tokenization; no training on either
  backbone, so `(d, L)` is the backbone's publicly documented
  configuration (GPT-3 davinci: `d=12288, L=96`; Llama-2-7B:
  `d=4096, L=32`).
- [UniTS](../papers/units.md): custom transformer-like block with
  dynamic FFN. Embedding dim `d = 64` (UNITS-SUP) or `d = 128`
  (UNITS-PMT), 3 blocks, 3.41M params in the supervised variant.
  Configs disclosed in §D of the paper.
- [TOTEM](../papers/totem.md): VQ-VAE tokenizer (codebook K=256,
  compression F=4, code dim D=64) plus a small downstream
  transformer forecaster with `d=64, L=4, heads=4, d_ff=256`.
  Forecaster config in Appendix A.11.

## Wider or deeper?

If you have parameters to spend, **widen `d` before adding layers**.
Attention and FFN both scale as `d²` while depth mostly helps with
compositional reasoning, which plateaus beyond roughly 16 layers at
time-series scales. `d=768, L=12` (85M) usually beats `d=512, L=24`
(75M) at matched parameter count. Most published TS foundation models
sit in the sweet-spot box `d ∈ {512, 768, 1024}` with
`L ∈ {6, 8, 12, 16}` — the bracket table above reflects that.

Exceptions:

- **CPU inference under ~10M params**: you are probably not using a
  standard transformer. [TTM](../papers/ttm.md) reaches 1–5M via a
  TSMixer backbone, not self-attention, and
  [Mamba4Cast](../papers/mamba4cast.md) uses a state-space model. The
  `12·d²` formula does not apply — see
  [lightweight-non-transformer](../architectures/lightweight-non-transformer.md).
- **Encoder-decoder (T5) architectures** like
  [Chronos](../papers/chronos.md) effectively double the per-layer cost
  because you count encoder and decoder blocks together. At `d=1024`
  that is roughly `24·d² ≈ 25M` per encoder+decoder pair. T5-Base
  (`d=768`, 12 encoder + 12 decoder) is ~220M;
  [Chronos-Base](../papers/chronos.md) inherits that size directly.
- **Mixture-of-Experts** at the Huge bracket: the 12·d² rule counts
  dense params. For MoE you separate *total* params (what the formula
  gives, multiplied by the number of experts on the FFN) from
  *activated* params per token. [Time-MoE](../papers/time-moe.md) has
  2.4B total / 1.1B active; [Timer-S1](../papers/timer-s1.md) has 8.3B
  total / 0.75B active. Report both numbers.

## Sanity check against GPT-2

GPT-2 medium is `d=1024, L=24, vocab=50257`. The formula gives
`12 × 1024² × 24 = 302M` core; the 50257-token vocabulary adds
`2 × 50257 × 1024 ≈ 103M` of which ~51M is independent under weight
tying; the total is **~353M**. Published GPT-2 medium is **355M**. The
`12·d²·L` rule is accurate to ~1%.

For a TS foundation model with negligible vocabulary, the formula is
essentially the full parameter count — no NLP-vocab correction needed.
This is why TS-FMs at comparable `(d, L)` to GPT-2 configs end up
noticeably smaller: they skip the token-embedding tax.

## Shortlist: copy a specific wiki anchor

- **Match [Chronos-Small](../papers/chronos.md) 46M** (paper's
  smallest-but-one T5 variant; note Chronos's size names are
  Mini 20M / Small 46M / Base 200M / Large 710M) →
  `d=768, L=6` (42.5M) for a decoder-only equivalent. Note Chronos
  itself is a T5 enc-dec, so for an apples-to-apples T5 replica you
  would use `d=512` with 6 encoder + 6 decoder layers (the official
  `google/t5-efficient-small` config).
- **Match [MOMENT-Base](../papers/moment.md) 125M** →
  `d=1024, L=10` (125.8M) or `d=768, L=18` (127.4M).
- **Match [TimesFM](../papers/timesfm.md) ~200M** →
  `d=1024, L=16` (201.3M). This is the cleanest configuration in the
  whole table — it lands within 1% of a published anchor with exactly
  round-number `d` and `L`.

## Related wiki pages

- [training-a-small-model.md](training-a-small-model.md) — where to
  train and what corpus to use
- [univariate-benchmarking.md](univariate-benchmarking.md) — where to
  evaluate so your numbers are comparable
- [efficiency-and-cost.md](efficiency-and-cost.md) — parameter counts,
  inference latency, and CPU deployability across existing TS-FMs
- [decoder-only autoregressive architecture](../architectures/decoder-only-autoregressive.md) — the assumed architecture for this cheat sheet
- [lightweight non-transformer](../architectures/lightweight-non-transformer.md) — how sub-10M models escape the 12·d² floor
- [mixture of experts](../architectures/mixture-of-experts.md) — how MoE total vs activated counts work at Huge scale
