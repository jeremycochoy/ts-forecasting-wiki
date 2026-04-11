# Reproducibility and Buildability

For each of the 20 papers on this wiki, the table below records what a
new researcher can actually get their hands on: open weights, training
code, training corpus, disclosed training cost, and deployment
footprint. Values are taken from the per-paper fact-sheet on this wiki;
where a value was not disclosed by the paper and is not general
knowledge, the cell is "—".

Training cost is notoriously under-reported in TS-FM papers, so most of
that column is "—". Do not read a dash as "cheap"; read it as "not
known".

## Per-paper reproducibility

| Paper | Open weights | Open code | Open training data | Training cost | Deployment footprint |
|---|---|---|---|---|---|
| [TimesFM](../papers/timesfm.md) | Yes (HuggingFace) | Yes (inference) | Partial (corpus is Google Trends + Wiki Pageviews + synthetic; not fully released) | — | ~200M params, GPU-friendly |
| [Chronos](../papers/chronos.md) | Yes (all 20M–710M sizes on HF) | Yes | Partial (TSMix + KernelSynth recipes published; underlying datasets are public) | — | 20M–710M, runs on a single GPU at all sizes |
| [Chronos-2](../papers/chronos-2.md) | Yes (HF) | Yes | Synthetic pretraining pipeline described | — | 120M, single GPU |
| [MOMENT](../papers/moment.md) | Yes (40M / 125M / 385M on HF) | Yes | Yes (Time Series Pile is released) | — | Up to 385M, single GPU |
| [MOIRAI](../papers/moirai.md) | Yes (base / large / small) | Yes | Yes (LOTSA, ~27B observations) | — | Medium, single GPU |
| [Moirai-MoE](../papers/moirai-moe.md) | Yes (assumed, Salesforce) | Yes | Yes (reuses LOTSA) | — | Sparse MoE, activated compute cheaper than dense equivalent |
| [Timer](../papers/timer.md) | Yes | Yes | Partial (S3 format corpus) | — | Mid-size decoder, single GPU |
| [Timer-XL](../papers/timer-xl.md) | Yes | Yes | — | — | Long-context decoder, multi-GPU for training |
| [Lag-Llama](../papers/lag-llama.md) | Yes | Yes | Yes (assembled from public sources) | — | Small decoder, single GPU |
| [TimeGPT-1](../papers/timegpt.md) | **No** (closed API) | No | No (proprietary) | — | API-only; no local deployment |
| [Time-MoE](../papers/time-moe.md) | Yes | Yes | Yes (Time-300B) | Large (billion-param sparse MoE on 300B tokens — not publicly priced) | 2.4B total params, activated subset at inference; still the heaviest deployable TS-FM |
| [Time-LLM](../papers/time-llm.md) | No new weights (reprograms frozen Llama) | Yes | — | — | Llama-sized backbone + small adapter; needs LLM serving stack |
| [GPT4TS / FPT](../papers/gpt4ts.md) | No new weights (reprograms frozen GPT-2 / BERT / BEiT) | Yes | — | — | GPT-2 sized, single GPU |
| [LLMTime](../papers/llmtime.md) | None (zero-shot on GPT-3 / Llama-2) | Yes | None (no training) | None (prompting only) | Whatever LLM you call |
| [TTM](../papers/ttm.md) | Yes (HF) | Yes | Partial | — | 1–5M params, CPU-deployable |
| [UniTS](../papers/units.md) | Yes | Yes | Yes (38-dataset bundle) | — | Single-GPU training reported |
| [TOTEM](../papers/totem.md) | Yes | Yes | — | — | Small VQ-VAE + small transformer |
| [Sundial](../papers/sundial.md) | Yes (assumed, Tsinghua) | Yes | TimeBench ~1T points | — | Flow-matching decoder, single-pass horizon |
| [Mamba4Cast](../papers/mamba4cast.md) | Yes | Yes | Synthetic only (no real corpus dependency) | Low (PFN-style training) | SSM backbone, linear-time inference |
| [Timer-S1](../papers/timer-s1.md) | Partial (release claimed at Tsinghua/ByteDance) | Partial | Yes (TimeBench ~1T pts, curation pipeline documented) | — (multi-billion sparse MoE on ~1T pts, not publicly priced) | 8.3B total / 0.75B active, 11.5K context, single-pass multi-horizon via TimeSTP |

## Reading the table

Three groups fall out:

- **Fully open and deployable.** TimesFM, Chronos, Chronos-2, MOMENT,
  MOIRAI, Moirai-MoE, Timer, Timer-XL, Lag-Llama, TTM, UniTS, TOTEM,
  Sundial, Mamba4Cast — weights on HuggingFace or equivalent, code
  available, and at least a documented pretraining recipe. A researcher
  can fine-tune any of these today.
- **Code-only, bring your own LLM.** Time-LLM, GPT4TS, LLMTime. You
  need a base LLM to run them; their contribution is the protocol, not
  an artifact. Treat them as baselines, not deployment targets.
- **Closed.** TimeGPT-1 is API-only. It is useful as a commercial
  benchmark but cannot be extended.

Training cost is the single most opaque column. Time-MoE is the only
paper that clearly crosses into "big run" territory; everything else is
plausibly reproducible on single-GPU or small multi-GPU budgets, which
is one of the most under-appreciated facts about TS-FMs.

## Good models to build on right now

If you are starting a project this week, these are the checkpoints with
the best ratio of capability to friction. If you are starting a *training
run* rather than a fine-tune, see
[../benchmarks/training-a-small-model.md](../benchmarks/training-a-small-model.md)
for the full recipe — corpus choice, curation pipeline, backbone and
head options, and the parameter-count brackets that define "small."

- **TTM** — for deployment-constrained work, CPU targets, or any
  project where you need to iterate fast. 1-5M parameters means you
  can fine-tune on a laptop. See [../papers/ttm.md](../papers/ttm.md).
- **Chronos (base or large) or Chronos-2** — for probabilistic
  forecasting where you need samples and quantiles out of the box.
  Chronos has clean value quantization; Chronos-2 is the current
  champion when you have related series. See
  [../papers/chronos.md](../papers/chronos.md) and
  [../papers/chronos-2.md](../papers/chronos-2.md).
- **MOIRAI / Moirai-MoE** — for multivariate zero-shot, especially
  when the number of channels varies between datasets. See
  [../papers/moirai.md](../papers/moirai.md) and
  [../papers/moirai-moe.md](../papers/moirai-moe.md).
- **MOMENT** — for multi-task work (forecast + classify + anomaly +
  impute from the same encoder). The released Time Series Pile is an
  independent bonus. See [../papers/moment.md](../papers/moment.md).
- **Time-MoE** — if you specifically want to study billion-parameter
  sparse MoE on TS. It is the only credible open artifact at that
  scale. See [../papers/time-moe.md](../papers/time-moe.md).
- **Mamba4Cast** — if your research question is about non-transformer
  backbones or synthetic-only training. It is the clearest open example
  of both bets. See [../papers/mamba4cast.md](../papers/mamba4cast.md).

## Related wiki pages

- [comparison-matrix.md](comparison-matrix.md)
- [contributing.md](contributing.md)
- [../papers/README.md](../papers/README.md)
