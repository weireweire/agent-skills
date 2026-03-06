---
name: disagg-perf-analysis
description: Analyze performance of SGLang disaggregated (prefill-decode separation) inference jobs. Use when the user asks about PD disagg performance, throughput issues, latency analysis, or bottleneck diagnosis for disaggregated serving jobs.
---

# Disaggregated Inference Performance Analysis

Analyze SGLang prefill-decode disaggregation job logs to identify bottlenecks. Reference ~/sglang source code for implementation details when needed.

## Log Locations

Logs are in `outputs/<job_id>/logs/`:
- `*_prefill_w*.out` — prefill worker logs
- `*_decode_w*.out` — decode worker logs
- `benchmark.out` — benchmark results
- `config.yaml` in `outputs/<job_id>/` — job config

## Step 1: Read Benchmark Results

Check `benchmark.out` for key metrics:
- **TTFT** (Time to First Token): high TTFT = prefill or bootstrap bottleneck
- **TPOT** (Time per Output Token): high TPOT = decode compute bottleneck
- **ITL** (Inter-token Latency): ITL >> TPOT = decode scheduling/batching issue
- **Output throughput (tok/s)**: overall decode generation speed
- **Request throughput (req/s)**: end-to-end throughput

## Step 2: Analyze Prefill Logs

Prefill log line format:
```
Prefill batch, #new-seq: N, #new-token: N, #cached-token: N, token usage: X.XX,
#running-req: N, #queue-req: N, #prealloc-req: N, #inflight-req: N,
input throughput (token/s): X.XX
```

### Field meanings (source: `sglang/srt/observability/scheduler_metrics_mixin.py`)

| Field | Source | Meaning |
|-------|--------|---------|
| `#new-seq` | `len(can_run_list)` | Number of new sequences scheduled for prefill in this batch |
| `#new-token` | `adder.log_input_tokens` | Tokens actually computed (not cached) |
| `#cached-token` | `adder.log_hit_tokens` | Tokens hit in prefix/radix cache |
| `token usage` | `(max_total - available - evictable) / max_total` | KV cache pool utilization (weights + KV). For prefill, stays low since KV is transferred out |
| `#running-req` | `len(running_batch.reqs)` | Requests in running batch (usually 0 for disagg prefill) |
| `#queue-req` | `len(waiting_queue)` | Requests waiting to be scheduled for prefill |
| `#prealloc-req` | `len(disagg_prefill_bootstrap_queue.queue)` | **Requests in bootstrap handshake with decode** |
| `#inflight-req` | `len(disagg_prefill_inflight_queue)` | Requests whose KV cache is being transferred to decode |

### Prefill request lifecycle (source: `sglang/srt/disaggregation/prefill.py`)
```
New request arrives
    |
Bootstrap Queue (#prealloc-req)  <-- handshake with decode worker (KVSender created, poll Bootstrapping -> WaitingForInput)
    |
Waiting Queue (#queue-req)       <-- waiting to be scheduled into prefill batch
    |
Prefill Forward (#new-seq/token) <-- actual prefill computation
    |
Inflight Queue (#inflight-req)   <-- KV cache transfer to decode via RDMA/nixl
    |
Done, release prefill-side memory
```

### Prefill bottleneck signals
- **High `#prealloc-req`**: Requests stuck in bootstrap. Root cause is usually decode side can't pre-allocate KV memory fast enough (see decode `_allocatable_tokens`)
- **High `#queue-req`**: Prefill compute is the bottleneck, requests waiting for GPU
- **High `#inflight-req`**: KV transfer is slow (network/RDMA issue)
- **Low `input throughput`**: Prefill compute is slow or starved of requests

## Step 3: Analyze Decode Logs

Decode log line format:
```
Decode batch, #running-req: N, #token: N, token usage: X.XX,
pre-allocated usage: X.XX, #prealloc-req: N, #transfer-req: N,
#retracted-req: N, cuda graph: True, gen throughput (token/s): X.XX, #queue-req: N
```

### Field meanings (source: `sglang/srt/observability/scheduler_metrics_mixin.py`)

| Field | Source | Meaning |
|-------|--------|---------|
| `#running-req` | `len(running_batch.reqs)` | Requests actively generating tokens |
| `#token` | from `_get_token_info` | Total tokens in KV cache (used, not available) |
| `token usage` | `num_used / max_total_num_tokens` | **Total** KV pool utilization. Includes running + pre-allocated + transfer. `pre-allocated usage` is a SUBSET of this |
| `pre-allocated usage` | `sum(fill_ids for transfer_queue) / max_total` | KV memory pre-allocated for requests in transfer queue. This is INCLUDED in `token usage` |
| `#prealloc-req` | `len(decode_prealloc_queue.queue)` | Requests doing handshake + waiting for KV memory allocation on decode side |
| `#transfer-req` | `len(decode_transfer_queue.queue)` | Requests that have KV memory allocated, waiting for KV data transfer from prefill |
| `#retracted-req` | `len(decode_prealloc_queue.retracted_queue)` | Requests retracted to CPU due to memory pressure |
| `gen throughput` | tokens generated per second | Decode generation speed |

### Decode request lifecycle (source: `sglang/srt/disaggregation/decode.py`)
```
Request arrives at decode
    |
PreallocQueue (#prealloc-req)    <-- handshake with prefill + wait for KV memory allocation (_pre_alloc)
    |                                 Gated by _allocatable_tokens()
    |
TransferQueue (#transfer-req)    <-- KV memory allocated, receiving KV data from prefill via RDMA
    |                                 pre-allocated usage tracks memory used here
    |
Running Batch (#running-req)     <-- actively generating tokens
    |
Done or Retracted
```

### Key: `_allocatable_tokens()` formula (source: `decode.py:691-739`)
```python
allocatable_tokens = available_size - max(
    num_reserved_decode_tokens * (running + transfer + waiting),  # reserve space for future decode
    need_space_for_single_req,  # ensure at least one req can finish
)
```
- `num_reserved_decode_tokens`: default 512, configurable via `--num-reserved-decode-tokens`
- Each new request needs `ISL + num_reserved_decode_tokens` tokens to pre-allocate
- This is the most common bottleneck gate

### Decode bottleneck signals
- **`token usage` near 1.0**: KV cache full, can't accept more requests
- **High `#transfer-req`**: Many requests waiting for KV data from prefill (prefill is slow or network is slow)
- **High `#prealloc-req`**: Decode can't allocate KV memory (allocatable_tokens exhausted)
- **High `#retracted-req`**: Memory pressure caused retraction to CPU
- **Low `gen throughput`**: Decode compute bottleneck or too few running requests

### Relationship between token usage and pre-allocated usage
- `pre-allocated usage` is a **SUBSET** of `token usage` (included, not additive)
- `token usage = running requests' KV + transfer queue's pre-allocated KV`
- If `token usage` is NOT near 1.0, decode KV capacity is not the direct bottleneck
- The real gate is `_allocatable_tokens()` which applies conservative reservations

## Step 4: Common Bottleneck Patterns

### Pattern A: Prefill `#prealloc-req` high, decode `token usage` < 1.0
**Diagnosis**: Decode `_allocatable_tokens` is too conservative.
**Tuning**:
- Decrease `num-reserved-decode-tokens` (default 512, try 256)
- Increase `mem-fraction-static` (note: this is weights + KV combined, not just KV)

### Pattern B: Prefill `#queue-req` high
**Diagnosis**: Prefill compute is the bottleneck.
**Tuning**:
- Increase prefill nodes/workers
- Increase `max-prefill-tokens` / `chunked-prefill-size`
- Increase `data-parallel-size` on prefill

### Pattern C: Decode `token usage` near 1.0, `#retracted-req` > 0
**Diagnosis**: Decode KV cache genuinely full.
**Tuning**:
- Increase decode nodes (more KV capacity)
- Increase `mem-fraction-static` on decode
- Reduce concurrency in benchmark

### Pattern D: High `#inflight-req` (prefill) or high `#transfer-req` (decode)
**Diagnosis**: KV transfer is slow.
**Tuning**:
- Check RDMA/nixl configuration
- Verify `MC_FORCE_MNNVL`, `NCCL_MNNVL_ENABLE` etc.
- Check network bandwidth between prefill and decode nodes

### Pattern E: High TTFT but low `#prealloc-req` and low `#queue-req`
**Diagnosis**: Bootstrap handshake itself is slow.
**Tuning**:
- Check `SGLANG_DISAGGREGATION_BOOTSTRAP_TIMEOUT`
- Check `SGLANG_DECODE_BOOTSTRAP_TIMEOUT`

## Step 5: Relevant Config Parameters

### NOT related to disagg prealloc bottleneck
- `--schedule-conservativeness`: Only affects `new_token_ratio` in `PrefillAdder` scheduling (non-disagg code path). Does NOT affect `DecodePreallocQueue._allocatable_tokens()`.

### Related parameters
| Parameter | Default | Effect |
|-----------|---------|--------|
| `num-reserved-decode-tokens` | 512 | Tokens reserved per request for future decode in `_allocatable_tokens()` |
| `mem-fraction-static` | auto | (weights + KV cache) / GPU memory. Higher = more KV capacity |
| `max-running-requests` | varies | Max concurrent requests in running batch |
| `context-length` | model default | Max sequence length, affects per-request KV allocation |
| `data-parallel-size` | 1 | DP shards; more DP = each shard handles fewer requests |
