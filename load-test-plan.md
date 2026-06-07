# Load Test Plan — Vision Moderation Service

## 1. Tool Choice

**Tool: [k6](https://k6.io/)**

k6 is JavaScript-scriptable, outputs structured JSON metrics natively, and integrates directly with Grafana/Prometheus for real-time observation alongside production dashboards. It handles both constant-rate and ramping VU models cleanly, which maps directly to our warmup → ramp → soak profile. Unlike wrk2 (no scripting) or Locust (Python overhead), k6 gives us per-endpoint scenario splitting (sync vs batch) without test harness complexity.

---

## 2. Test Phases

| Phase | Duration | RPS Target | Description |
|---|---|---|---|
| **Warmup** | 5 min | 30 RPS | Low traffic to prime JIT, fill Redis cache, and confirm baseline health. No pass/fail applied. |
| **Ramp** | 10 min | 30 → 300 RPS | Linear ramp over 10 minutes. Observe autoscaler response and replica provisioning. |
| **Sustained peak** | 30 min | 300 RPS | Steady-state at production peak. Primary measurement window for all pass/fail criteria. |
| **Spike** | 5 min | 300 → 500 RPS | Simulate spike traffic matching the stated operational ceiling. Validate autoscaler scale-out. |
| **Recovery** | 5 min | 500 → 300 RPS | Ramp back down. Confirm error rate and latency normalize after spike. |
| **Soak** | 60 min | 200 RPS | Sustained moderate load to catch memory leaks, connection pool exhaustion, and slow resource drain. |

**Total test duration: ~115 minutes**

---

## 3. Traffic Shape

**Endpoint split:**
- 90% of VUs target `POST /v1/moderate` (synchronous)
- 10% of VUs target `POST /v1/moderate/batch` (batch)

This is implemented in k6 as two named scenarios with weighted executor shares.

**Payload size distribution (synchronous):**

| Payload | Weight | Description |
|---|---|---|
| 100 KB image | 60% | Typical mobile upload |
| 400 KB image | 30% | High-res photo |
| 1.2 MB image | 10% | Near-max acceptable size |

Images are pre-generated synthetic test files (no real user data). The same files are reused from a local fixture pool to avoid test-client I/O becoming a bottleneck.

**Payload size distribution (batch):**

Batches of 8–16 images drawn from the same pool. Batch size sampled uniformly from `[8, 12, 16]`.

**Concurrency model:**

k6 `constant-arrival-rate` executor is used (not VU-based). This correctly models RPS load regardless of server latency, so backpressure is visible as queue buildup, not invisible as "test slowed down with the server."

---

## 4. Pass/Fail Criteria

All criteria below must be satisfied during the **sustained peak phase (300 RPS, 30 min)** and the **spike phase (500 RPS, 5 min)** unless otherwise noted.

| Metric | Threshold | Phase |
|---|---|---|
| p95 latency `/v1/moderate` | ≤ 250 ms | Sustained + Spike |
| p99 latency `/v1/moderate` | ≤ 400 ms | Sustained + Spike |
| p95 latency `/v1/moderate/batch` | ≤ 400 ms | Sustained + Spike |
| HTTP 5xx error rate (all endpoints) | ≤ 0.5% | Sustained + Spike |
| HTTP 5xx error rate (all endpoints) | ≤ 0.1% | Soak (60 min @ 200 RPS) |
| Successful request throughput | ≥ 295 RPS | Sustained peak |
| Successful request throughput | ≥ 490 RPS | Spike |
| p95 latency after spike recovery | ≤ 250 ms within 60 s of ramp-down | Recovery |
| Autoscaler scale-out completion | ≤ 90 s after spike begins | Spike |
| Memory per replica (heap) | No growth > 20% over 60-min soak at 200 RPS | Soak |

**Automatic test abort condition:** if 5xx error rate exceeds 5% for any 60-second window during ramp or sustained peak, the test terminates early and is marked a hard failure.

---

## 5. Bottleneck Checklist

During each test phase, the following will be actively monitored per replica in Grafana:

**GPU replicas (`g4dn.xlarge`):**
- [ ] GPU utilization % — target < 80% at sustained peak; watch for saturation during spike
- [ ] GPU memory usage MB — should remain stable; growth indicates a memory leak in inference pipeline
- [ ] Inference queue depth — should be near 0 at steady state; spike > 5 indicates batching or worker bottleneck
- [ ] CPU utilization % — pre/post-processing is CPU-bound; watch for saturation if > 85%
- [ ] Pod memory (RSS) — flat or very slowly growing; >20% growth over soak = leak

**Redis:**
- [ ] Redis p95 GET latency — should remain ≤ 10 ms; spikes indicate connection pool exhaustion or eviction
- [ ] Redis connection count — monitor for connection leak from inference workers
- [ ] Redis keyspace hit rate — should be > 90%; miss rate spike = cold cache or TTL misconfiguration

**Load balancer / ingress:**
- [ ] Active connections — look for connection queue buildup during spike
- [ ] 502/503 rate — upstream failures indicate replicas not ready or autoscaler lagging

**Application-level:**
- [ ] Request queue time (time between load balancer receipt and first worker pickup) — should be < 10 ms at sustained; < 30 ms at spike
- [ ] Python GC pause frequency and duration (if applicable) — long pauses inflate p99 tail
- [ ] Log error rate — any `ERROR` level log lines correlated with latency spikes
