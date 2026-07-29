# LLM Inference Performance

> Scope: load and capacity testing of LLM inference servers (vLLM, TRT-LLM, SGLang, OpenAI-compatible endpoints, Ray Serve LLM, KServe). Covers metric definitions, workload design, tools, and SLO methodology for generative AI.

LLM inference is **not conventional HTTP load testing**. Responses stream token-by-token over an open connection, output length is unbounded, and the dominant cost is GPU compute, not network. A generic `k6`/`JMeter` run that only measures request latency will badly misreport performance - it ignores the streaming shape entirely. This topic defines the metrics that actually matter, how to design realistic LLM workloads, and which tools to use.

> For general workload theory (Little's Law, open vs closed models), see `workload-design.md`. For analyzing the percentiles and SLO failures this produces, see `results-analysis.md`. For running these tests in CI/CD, see `test-execution.md`.

---

## Why LLM load testing is different

| Aspect | Traditional API | LLM inference |
|---|---|---|
| Response | Single body, fixed size | Streamed tokens, variable length |
| Latency signal | TTFB + total | TTFT + inter-token gaps + E2E |
| Cost driver | CPU/network | GPU memory (KV cache), compute |
| Throughput unit | Requests/sec | Tokens/sec (not requests/sec) |
| Concurrency limit | Threads/sockets | KV cache capacity (`max_num_seqs`) |
| Key risk | Timeouts | Queueing delay, GPU saturation, OOM |

Two phases dominate every request:
- **Prefill** - the model processes the full prompt to build the KV cache. Compute-heavy, determines **TTFT**. Scales with prompt length.
- **Decode** - tokens generated one at a time using the KV cache. Memory-bandwidth-bound, determines **ITL/TPOT**. Scales with output length.

---

## Core Metrics

### TTFT - Time to First Token
Time from request send to first streamed token. Driven by prompt length, queue/self.time, and prefill speed. The primary *perceived responsiveness* metric for chat/coding assistants. High TTFT under load usually means the scheduler is queuing requests (KV cache exhausted) rather than a slow model.

### TPOT / ITL - Generation smoothness
- **TPOT (Time Per Output Token)** - average gap between tokens for a single request: `(E2E - TTFT) / (output_tokens - 1)`. Report the mean of per-request TPOTs.
- **ITL (Inter-Token Latency)** - same gaps, but token-weighted across all requests (mean of every gap). This is the system's steady streaming speed.
- **Which to use**: TPOT compares per-request behavior (each request equal); ITL estimates system-wide streaming feel across mixed traffic. A bursty ITL (high p99 vs mean) makes output appear in clumps - the classic "GPU contention" signature.

### E2E Latency
Total time from prompt to final token. `E2E = TTFT + (output_tokens - 1) * TPOT`. Matters for batch/codegen/summarization where the full response is needed before the next step.

### TPS - Token Throughput
- **System TPS** - total output tokens/sec across all requests. Raw capacity; rises with load until GPU-saturated. `TPS = output_tokens / (T_last - T_first)`.
- **User TPS** - tokens/sec a single user experiences; `≈ 1 / ITL` at long outputs. Drops as concurrency rises because the engine shares the GPU.

### RPS - Request Throughput
Completed requests/sec. Dominant metric for high-volume short-prompt traffic (chatbots, search, API gateways). Shorter prompts + higher `max_num_seqs` raise RPS.

### Goodput (the metric that matters most)
`Goodput = (requests meeting ALL SLOs) / total_requests * 100%`. Unlike TPS/RPS, goodput tells you what fraction of users got an *acceptable* experience. A system can show high TPS while most requests violate latency SLOs. Always define goodput with explicit thresholds, e.g. TTFT < 500ms, TPOT < 15ms, E2E < 2s. As load climbs, goodput falls even as raw throughput keeps rising - that crossover is your real capacity limit.

### Percentiles
Report **p50/p95/p99** for TTFT, ITL, and E2E - never just averages. Averages hide the unlucky 5% whose tokens arrive in bursts. p99 is the near-worst-case; if p99 meets SLO, the system is consistent.

---

## Workload Dimensions

Controlling these is what separates a useful LLM test from a misleading one:

| Dimension | Effect | How to set |
|---|---|---|
| **Prompt length (ISL)** | Longer → higher TTFT, more KV cache | Use real traffic (ShareGPT) or synthetic ranges |
| **Output length (OSL)** | Longer → higher E2E, more decode cost | Real distributions; `--random-output-len` for synthetic |
| **Concurrency** | Bounded by KV cache, not sockets | Set `--max-concurrency` to simulate gateway limits |
| **Request rate** | Open-model arrival (Poisson) | `--request-rate`; `inf` for max throughput |
| **Burstiness** | Gamma-distributed arrivals | `1.0` realistic, `0.1-0.5` stress, `2-5` uniform |
| **Batching (`max_num_seqs`)** | Higher → more RPS, worse per-user latency | Tune on the server, not the client |

**KV cache math**: `max_concurrency ≈ KV_cache_tokens / max_model_len`. vLLM prints this at startup. Set test concurrency to 80-90% of it for capacity planning; use the full value as the SLA limit.

---

## Tools

Purpose-built LLM benchmarkers surface token metrics natively. Prefer them over generic HTTP load testers.

| Tool | Best for | Notes |
|---|---|---|
| **vLLM `bench serve`** | Serving benchmarks against an OpenAI-compatible endpoint | Native TTFT/TPOT/ITL/TPS, ShareGPT + synthetic datasets, ramp-up, goodput via percentiles |
| **GuideLLM** | Production vLLM SLA/capacity studies | Auto reports, live progress, profile-based; recommended by vLLM for production |
| **NVIDIA GenAI-Perf** | Multi-backend (TRT-LLM, Triton, vLLM), concurrency/rate modes | Emits TTFT, ITL, output TPS, goodput; pairs with Perf Analyzer |
| **llmperf / LLM-Perf** | Quick pointwise latency/throughput checks | Lightweight, great for smoke and regression |
| **LLM Locust** | Distributed load on Locust, GenAI metrics | Use when you already run Locust fleets |
| **k6 + custom metrics** | Unified CI with existing k6 stacks | Must instrument streaming manually: capture TTFT from first SSE chunk, ITL between chunks, count tokens. See `../tools/k6.md` for metric primitives |

> **Anti-pattern (per current research):** do not misuse model-server micro-benchmarkers (vLLM `bench`, SGLang bench) as production-level evaluators. They optimize for regression/feature testing, not realistic arrival patterns. Use them for component baselines; use GuideLLM/GenAI-Perf/production-style load for capacity and SLO validation.

---

## Quick Start Commands

```bash
# vLLM bench serve - max throughput probe with ShareGPT data
python -m vllm.entrypoints.openai.api_server --model meta-llama/Llama-3-8B &
python -m vllm bench serve \
  --model meta-llama/Llama-3-8B \
  --endpoint http://localhost:8000/v1/completions \
  --dataset-name sharegpt \
  --request-rate inf \
  --max-concurrency 256 \
  --num-prompts 500

# vLLM bench serve - realistic arrival pattern
python -m vllm bench serve \
  --model meta-llama/Llama-3-8B \
  --endpoint http://localhost:8000/v1/completions \
  --dataset-name sharegpt \
  --request-rate 10 \
  --burstiness 1.0 \
  --num-prompts 200

# GuideLLM - production SLA study
pip install guidedllm
guidedllm evaluate \
  --model meta-llama/Llama-3-8B \
  --base-url http://localhost:8000/v1 \
  --data emulated --rate 10 --duration 120

# NVIDIA GenAI-Perf - multi-backend benchmark
genai-perf profile \
  --model meta-llama/Llama-3-8B \
  --endpoint-type chat \
  --url localhost:8000 \
  --concurrency 64 \
  --request-count 200

# llmperf - quick latency/throughput check
python token_benchmark_ray.py \
  --model meta-llama/Llama-3-8B \
  --mean-input-tokens 512 \
  --mean-output-tokens 128 \
  --num-concurrent-requests 32 \
  --results-dir ./results
```

---

## Test Design & Methodology

1. **Start from real traffic shapes.** Use ShareGPT or captured production traces for prompt/output length distributions. Synthetic `random` datasets are fine for stress but unrealistic for sizing.
2. **Define SLOs as goodput thresholds first.** e.g. "TTFT p95 < 500ms, ITL p95 < 20ms, E2E p95 < 3s" → goodput target 99%.
3. **Run a max-throughput probe** (`--request-rate inf --max-concurrency <limit>`) to find the concurrency ceiling and baseline TPS.
4. **Sweep concurrency / request rate** to find the goodput cliff - the load where SLO compliance drops. That is capacity.
5. **Test ramp-up and spikes** (linear/exponential ramp, bursty arrival) to validate autoscaling and queue behavior.
6. **Always assert on token metrics, not just HTTP 200.** A 200 with a 10s TTFT is a failed request.

### Workload pattern recipes (vLLM bench semantics)
| Goal | `--request-rate` | `--burstiness` | `--max-concurrency` |
|---|---|---|---|
| Max throughput | `inf` | n/a | limited |
| Realistic baseline | 5-20 | 1.0 | inf |
| Stress / resilience | 20-100 | 0.1-0.5 | inf |
| Latency profiling | 1-10 | 2-5 | inf |
| Capacity / SLA | target rate | 1.0 | SLA limit |

---

## Common Mistakes

- **Measuring only HTTP latency** - TTFT and streaming gaps are invisible to `http_req_duration`. You'll report a green test while users stare at a blank cursor.
- **Ignoring output length variance** - fixed `max_tokens` hides the real E2E spread; use realistic OSL distributions.
- **Unbounded concurrency** - without `--max-concurrency`, the client fires until the GPU chokes; you measure collapse, not capacity. The KV cache, not sockets, is the real cap.
- **Confusing TPS with user experience** - high system TPS can coexist with terrible per-user ITL. Track both; report goodput.
- **Averaging instead of percentiles** - a 200ms mean TTFT with 8s p99 is a broken chat UX.
- **Using a micro-benchmarker for production SLOs** - `vllm bench`/SGLang bench validate the engine, not your serving capacity under real arrival patterns.
- **No streaming instrumentation in k6** - a naive `http.get` to a streaming endpoint counts only the final byte; you must parse SSE chunks to get TTFT/ITL.
- **Forgetting prompt caching** - prefix caching dramatically cuts TTFT for repeated prefixes; test with and without it (e.g. RAG, system prompts) to size the win.

---

## Observability

Track these server-side alongside the client metrics above:
- **KV cache utilization** - the true saturation signal; when it pins at 100%, TTFT inflates via queueing.
- **Batch size / `num_running` vs `num_waiting`** - waiting > 0 means you're over concurrency.
- **Prefill vs decode time split** - isolates whether slowness is prompt-side or generation-side.
- **GPU util + memory** - confirms you're compute-bound, not starved.

> For dashboards/APM integration patterns, see `observability.md`. For interpreting the goodput cliff and saturation, see `results-analysis.md`.
