# Fat-session mutex — plan (2026-09-11)

## Problem (measured today)
- Pool: ~410k tokens (KVarN, `KV_MEM=7.5GiB`, `MAX_SEQS=4`, `max_model_len 245760`).
- One session ≈ 135k tokens. 1× fast. 2× concurrent: second queues ~70s
  (`waiting=1 capacity`, kv 25→47%), then both decode at 3–8 tok/s with
  draft acceptance collapsing 77%→30–35%. 3× distinct: kv →76%, prefix
  delta hit-rate ~24% vs ~90% lifetime, `preemptions 0` (scheduler serializes
  via waiting instead of evicting).
- Prompts are small; **sessions** grow (each turn resends full history).
  Stale/idle sessions hold 0% resident — only their prefix-cache photocopies
  linger (LRU, evictable).
- Rule of thumb: max 2 fat sessions, expect them slow. Never 3.

## Goal
When `fatA + fatB > ~350k` (margin under 410k pool), run fat turns **one at a time**:
one gets the whole desk (fast), the other waits with a visible reason —
instead of both admitted and crawling together.

## Option A — pi extension (client-side)
Seat: `pi agent → [extension: GO/WAIT] → vLLM:18020`.

- Sees true session + scroll size for free (no inference).
- Before each send: `size = usage.prompt_tokens` (last turn) or `chars/4` estimate.
  - `size < 100k`: always GO (thin traffic never blocks).
  - `size >= 100k`: need exclusive fat-lock (lockfile, e.g. `/tmp/qwen-fat.lock`
    holding `{session, size, since}`); holder GO, others WAIT with message
    "GPU busy with the other long session".
  - Release lock when turn completes (or stale-timeout, e.g. 10 min).
- Also: warn at ~120k (50%), force compact (summarize + restart from summary,
  archive old log) at ~190k (80%). Racer B showed agents already dodge huge
  fills via grep — make that policy in `.agents/agents/qwen38.md`.
- Pros: cheapest, best UX, true sizes, no extra port/latency.
- Cons: guards pi only. Any direct `:18020` traffic (curl, other TUI) bypasses it.
- Build: extension hook on pre-send + lockfile helpers + `scroll-size` status line.
  No server restart.

## Option B — middleman proxy (server-side, all clients)
Seat: `pi / other TUI / curl → [bouncer :18021] → vLLM:18020`. All clients repoint.

- Per request: estimate tokens (`chars/4` or tokenizer), fetch live `/metrics`
  (`kv_cache_usage_perc`, `running`, `waiting`).
  - Thin (`<100k`) + `kv < 0.6` + `running < 2`: forward.
  - Fat (`>=100k`): needs fat-lock; else `429 + Retry-After` + JSON reason.
  - Sessions inferred (stateless HTTP!): by API key, or prefix-hash of prompt head.
- Pros: guards everyone uniformly; single enforcement point; visible 429s instead
  of silent 70s queues.
- Cons: session inference is reconstruction, not knowledge; extra hop/port;
  more moving parts. Natural home per repo notes: alongside `llama-swap`
  (`REQ_METRICS=1` already exposes per-request metrics).
- Build: tiny forwarder (python/node, ~150 lines) + systemd/compose entry +
  client repoint `:18020`→`:18021` + dashboards on `kv/waiting/hit-rate`.

## Decision guide
- All traffic is pi today (racers + stale were all pi) → build **A** now.
- The day a second client appears → add **B**; keep A as the nice WAIT message
  instead of a raw 429.

## Metrics to watch (while slow)
```
curl -s localhost:18020/metrics | grep -E "kv_cache_usage|prefix_cache|preempt|running|waiting"
docker logs qwen38-27b-rtx3090-single-1 --tail 5  # Prefix cache hit rate line
```
`kv~1.0` + `waiting>0` = capacity queue. Delta hit-rate (queries/hits before vs
after) = miss evidence. `preemptions>0` = live eviction (not seen yet).

## Open questions
- Exact fat threshold (100k vs 120k) and pool margin (350k vs 380k) — re-measure
  after any `KV_MEM`/`MAX_LEN` change (note: `MAX_LEN=150000` alone may defuse 2×).
- Lock stale-timeout + crash recovery (dead holder must not block desk forever).
- UX copy for WAIT state (who holds lock, since when, ETA heuristic).
- Whether thin traffic should *always* bypass (yes, proposed) or also yield during fat holds.
