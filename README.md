# gx10-bench — measuring DeepSeek-V4-Flash honestly on 4× DGX Spark

Four small tools and one result, from tuning DeepSeek-V4-Flash-0731 at TP=4 on GB10.

The headline is **123.13 tok/s** single-stream decode against **104.17** published for the same
TP=4 topology — but the number is the least interesting part. What's here is the *method*, and
three corrections to how this gets measured that cost me days to learn.

```bash
export DSFV4_HOST=<your-server-ip>
python3 accept_live.py 60          # zero-load: acceptance off whatever traffic is already flowing
python3 bst_parity.py              # the full parity benchmark, ~6 min
```

Pure stdlib. No install, no dependencies.

---

## Three things that will bite you

### 1. Aggregate acceptance is a trap. Compare tok/step.

A 79.6% acceptance figure over 6 draft positions **loses** to 71.7% over 7:

```
        mine   reference
pos 0   95.0%     96.2%
pos 3   91.0%     76.3%
pos 5   86.0%     62.3%
pos 6   79.4%       —      a 7th position the other config never attempts
------------------------------------------
tok/step 6.019     5.777
```

Deeper drafting only pays if the curve *holds*. Aggregate acceptance across different `k` is
not a comparison. `accept_live.py` prints the per-position ladder for this reason.

### 2. Warmup depth moved my result by 11 tok/s

Same server, same config, same day: **111.86** tok/s with 1 warmup, **123.13** with 8
(2 full-length + 6 discarded). That is larger than most tuning changes. A published throughput
number that doesn't state its warmup isn't comparable — including my own earlier one.
`bst_parity.py` hard-codes the full warmup and won't let you shorten it silently.

### 3. `--max-num-batched-tokens` is not your prefill budget

With spec decode on, vLLM subtracts draft slots:

```
max_num_scheduled_tokens = max_num_batched_tokens − (k−1) × max_num_seqs
```

and warns below 8192. Every published config I checked sets `8192` raw and lands *under* the
threshold — at k=6/seqs=16 that's 8112. If you want an effective 8192, set
`8192 + (k−1) × max_num_seqs`.

---

## The tools

| file | what it does |
|---|---|
| **`accept_live.py`** | **Sends zero requests.** Samples Prometheus counters over a window and reports acceptance, tok/step, effective k, and the per-position ladder. Safe to run against production mid-traffic. |
| **`accept.py`** | Acceptance on a controlled fixture. Defaults to **omitting** `temperature` (what real clients send) rather than pinning it. `--sweep 0,0.3,0.7,1.0` plots the curve. |
| **`bst_parity.py`** | Full parity benchmark. Fixed prompt, 2 full warmups + 6 discarded + 10 measured, server-side decode-only metric. |
| **`cross_model_audit.py`** | Flags knobs where *your own* configs disagree with no recorded reason. |
| **`RESULTS.md`** | The measurement, the config, the protocol, and what it doesn't claim. |

**Metric used throughout** — decode-only, excludes prefill:

```
Δ vllm:generation_tokens_total / Δ vllm:request_decode_time_seconds_sum
```

Not tokens-over-wall-time. That naive metric once made my server look like a 40 tok/s machine.

### Two guards, because I got burned by their absence

- **`accept.py` aborts on foreign traffic** (exit 4). These are *global* counters — a sweep run
  against a live server was once **95% someone else's requests** and came back non-monotonic
  (temp 0.7 scoring above temp 0.0). That is what contamination looks like.
- **`cross_model_audit.py`** exists because `draft_sample_method` sat at `greedy` in one config
  and `probabilistic` in another of mine for weeks. Both values were written down; neither note
  said what the field *does*, so nothing fired when I wrote the second launcher.
  **`SOURCES` at the top is mine — edit it for your fleet.** It needs ≥2 configs, and it's most
  useful across *different models on the same hardware*, where a knob diverges unnoticed.

---

## What the number does not claim

- **Code generation, single stream, concurrency 1.** On my real agentic traffic — ~132K-token
  prompts, heavy tool use — the same server measures **~31% acceptance and ~3.2 tok/step**.
  Both numbers are true. Prompt shape drives draft acceptance more than any config knob.
- **Not a 1M-context claim.** I measured the KV pool flat at **54–56 GiB from 327,680 through
  1,048,576** — raising the ceiling buys token capacity (sparse attention makes long contexts
  cheaper per token), not memory. Advertised context and measured throughput are different
  numbers and rarely come from the same run.
- **Deeper drafting is not free.** Measured: k=7 → 122.3 tok/s, k=8 → 115.2, k=10 → 102.6.
  Acceptance degrades at *every* position as k grows, not just the new ones — so you cannot
  extrapolate deeper drafting from a shallower measurement. I tried; the model was off by 19%.

## Environment

`DSFV4_HOST` · `DSFV4_MODEL` · `DSFV4_SSH_HOST` · `DSFV4_CONTAINER` — nothing site-specific is
baked into these files.

Corrections welcome, especially to the numbers. Every claim here is reproducible with the
scripts in this repo; if yours disagree, I'd rather know.
