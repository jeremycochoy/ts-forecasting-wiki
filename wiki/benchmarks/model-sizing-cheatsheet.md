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

| Bracket | Range | Representative SOTA | Decoder-only `(d, L)` | Hardware tier |
|---|---|---|---|---|
| Tiny | 1–10M | [TTM](../papers/ttm.md) (1–5M, MLP-Mixer), [Lag-Llama](../papers/lag-llama.md), [Mamba4Cast](../papers/mamba4cast.md) | `d=256, L=4` → 3.1M · `d=256, L=8` → 6.3M · `d=384, L=4` → 7.1M — note: at `d=512` a single layer is already 3.1M, so 2 layers overshoot Tiny's upper end when combined with embeddings | Single consumer GPU, CPU inference realistic |
| Small | 10–50M | [Chronos-Tiny](../papers/chronos.md) (20M), [MOIRAI-Small](../papers/moirai.md) (14M), [MOMENT-Small](../papers/moment.md) (40M) | **`d=512, L=4`** → 12.6M · `d=512, L=8` → 25.2M · `d=768, L=4` → 28.3M · `d=512, L=12` → 37.7M | Single 4090 / A100 40GB |
| **Small ceiling** | **40–50M** | **[Chronos-Mini](../papers/chronos.md) (46M), [MOMENT-Small](../papers/moment.md) (40M)** | **`d=768, L=6`** → **42.5M** (cleanest fit) · `d=512, L=16` → 50.3M · avoid `d=1024, L=3` (37.7M, too shallow) | Single A100 40GB |
| Base | 50–200M | [Chronos-2](../papers/chronos-2.md) (120M), [MOMENT-Base](../papers/moment.md) (125M), [MOIRAI-Base](../papers/moirai.md) (91M), [Timer](../papers/timer.md) | `d=768, L=8` → 56.6M · `d=768, L=12` → 84.9M · **`d=1024, L=8`** → **100.7M** · `d=768, L=16` → 113.2M · `d=1024, L=12` → 151.0M | Single H100 / A100 80GB |
| **Base ceiling** | **~200M** | **[TimesFM](../papers/timesfm.md) (~200M), [Chronos-Base](../papers/chronos.md) (200M)** | **`d=1024, L=16`** → **201.3M** (matches TimesFM almost exactly) | Single H100 |
| Large | 200M–1B | [Chronos-Large](../papers/chronos.md) (710M), [MOMENT-Large](../papers/moment.md) (385M), [MOIRAI-Large](../papers/moirai.md) (311M), [Timer-XL](../papers/timer-xl.md) | `d=1024, L=24` → 302M · `d=1280, L=24` → 472M · `d=1536, L=24` → 680M | 4–8× datacenter GPUs |
| Huge | 1B+ | [Time-MoE](../papers/time-moe.md) (2.4B), [Timer-S1](../papers/timer-s1.md) (8.3B) | At this scale, a sparse MoE (activated subset of experts per token) beats a dense transformer of equal total size; dense equivalents would need `d=2048, L=24+`. | Cluster |

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

- **Match [Chronos-Mini](../papers/chronos.md) 46M** →
  `d=768, L=6` (42.5M) for a decoder-only equivalent. Note Chronos itself
  is a T5 enc-dec, so for an apples-to-apples T5 replica you would use
  `d=512` with 6 encoder + 6 decoder layers.
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
