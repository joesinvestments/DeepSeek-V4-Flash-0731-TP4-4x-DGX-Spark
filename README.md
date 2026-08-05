# DeepSeek-V4-Flash-0731 · TP=4 · 4× DGX Spark

Tuning DeepSeek-V4-Flash-0731 on 4× DGX Spark, one variable at a time.

**[→ THE RECIPE](RECIPE.md)** — the config, and every knob I tested to get there, including the
values that lost.

**123.13 tok/s** single-stream decode at TP=4, against **104.17** published for the same topology.
Same prompt, same protocol, n=10 each, distributions non-overlapping — my slowest run beat their
fastest.

```
                    mine     reference
mean              123.13        104.17
median            123.14        104.31
sd                  1.94          3.51
min               120.73         98.84
max               127.61        110.23
```

---

## Why this repo exists

Every DGX Spark repo publishes a working config. Almost none publish **what they tried that lost,
and by how much** — which is the only thing that lets you reason about your own hardware instead
of copying mine.

So [RECIPE.md](RECIPE.md) has the losing values next to the winning ones:

| knob | I run | I also measured |
|---|---|---|
| `draft_sample_method` | `probabilistic` | `greedy` — 26.5% acceptance vs 34.3% |
| `num_speculative_tokens` | 7 | k=8 → 115.2 tok/s, k=10 → 102.6 |
| `max_cudagraph_capture_size` | 96 | 36 — truncates to 32, drops to eager above 4 seqs |
| `gpu_memory_utilization` | 0.85 | 0.80 → −6.9 GiB · 0.90 → won't boot |
| `max_num_batched_tokens` | 8264 | 8192 — lands *under* vLLM's own warning threshold |
| `max_model_len` | 393,216 | 327K…1M — KV pool flat the whole way |

And a **[what I got wrong](RECIPE.md#what-i-got-wrong)** section, because two of the configs I
copied from other operators were wrong for my hardware, and four of my own hypotheses died under
measurement.

## Three things that will bite you

**1. Aggregate acceptance is a trap.** 79.6% over 6 draft positions loses to 71.7% over 7. Compare
`tok/step`, or compare per-position.

**2. Warmup depth moved my result 11 tok/s.** 111.86 under-warmed, 123.13 properly warmed. Same
server, same hour. A throughput number without a stated warmup isn't comparable.

**3. `--max-num-batched-tokens` isn't your prefill budget.** With spec decode on, vLLM subtracts
`(k−1) × max_num_seqs` and warns below 8192. Most published configs set 8192 raw and land under it.

## The tools

```bash
export DSFV4_HOST=<your-server-ip>
python3 accept_live.py 60      # acceptance off live traffic — SENDS ZERO REQUESTS
python3 bst_parity.py          # full parity benchmark, ~6 min
```

Pure stdlib, no install.

| | |
|---|---|
| **`accept_live.py`** | Reads acceptance, tok/step, and the per-position ladder off Prometheus counters. Sends nothing — safe against production mid-traffic. |
| **`accept.py`** | Controlled fixture. Defaults to *omitting* `temperature` (what real clients send). Aborts on foreign traffic — exit 4. |
| **`bst_parity.py`** | Parity benchmark: fixed prompt, 2 full warmups + 6 discarded + 10 measured, decode-only metric. |
| **`cross_model_audit.py`** | Flags knobs where your own configs silently disagree. Edit `SOURCES` for your fleet. |

Metric used throughout, decode-only:

```
Δ vllm:generation_tokens_total / Δ vllm:request_decode_time_seconds_sum
```

## What none of this claims

This is a **code-generation, single-stream** number. My real agentic traffic — ~132K-token
prompts, heavy tool use — measures **~31% acceptance and ~3.2 tok/step** on the exact same config.
Both are true. Prompt shape drives draft acceptance more than any knob in this repo.

It is also not a 1M-context throughput claim. I measured the KV pool flat from 327,680 to
1,048,576 — the ceiling buys token capacity, not memory — but nobody has benchmarked *at* a
million-token prompt, me included.

---

Corrections welcome, especially to the numbers. Everything here is reproducible with these
scripts. If yours disagree, I'd rather know.

MIT.
