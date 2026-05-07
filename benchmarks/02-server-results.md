# 02 — llama-server Load Test Results

Server config: `llama-server -ngl 99 --parallel 4 --cont-batching --metrics --ctx-size 4096`  
Model: `Llama-3.2-3B-Instruct-Q4_K_M.gguf` | Backend: Metal (Apple M1 Pro)

## Load test summary

| Concurrency | Total reqs | RPS | E2E P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 61 | 1.08 | 7,500 | 11,000 | 13,000 | 0 |
| 50 | 56 | 0.95 | 21,000 | 39,000 | 40,000 | 0 |

### 10 users — by type

| Type | P50 (ms) | P95 (ms) | P99 (ms) |
|---|--:|--:|--:|
| short | 7,400 | 9,300 | 9,800 |
| long-rag | 11,000 | 13,000 | 13,000 |

### 50 users — by type

| Type | P50 (ms) | P95 (ms) | P99 (ms) |
|---|--:|--:|--:|
| short | 20,000 | 36,000 | 39,000 |
| long-rag | 28,000 | 40,000 | 40,000 |

## KV cache / slot utilization

Metric `llamacpp:kv_cache_usage_ratio` not exposed in this Homebrew build (llama-server v9020).  
Proxy metric used: `llamacpp:n_busy_slots_per_decode = 3.74` under 10-user load (4 slots configured → **93.5% slot utilization**).

At 50 users, median latency rose from 7.5 s → 21 s — requests queued because all 4 slots were saturated.

## Observations

- With `--parallel 4`, throughput plateaus around 1.0–1.1 req/s regardless of concurrency.
- Increasing from 10 → 50 users does NOT increase RPS (0.95 vs 1.08) — the bottleneck is GPU compute, not network.
- P95 degrades 3.5× (11 s → 39 s) at 50 users due to queuing behind 4 saturated slots.
- long-rag requests consistently 2–3× slower than short — longer prompt = more prefill time.
