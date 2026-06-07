# Capacity Plan — Vision Moderation Service

## 1. Latency Budget Breakdown

The synchronous endpoint carries 90% of traffic with a p95 latency budget of **250 ms**.

| Stage | Budget (ms) | Notes |
|---|---|---|
| Network in | 5 | Half of 10 ms total network round-trip |
| Auth + routing | 3 | JWT validation + reverse proxy hop |
| Payload parse | 5 | Image decode + JSON deserialization |
| Feature lookup (Redis) | 10 | p95 = 8 ms; 10 ms gives 2 ms headroom |
| Pre-processing | 10 | Resize + normalize; part of ~15 ms combined pre/post CPU work |
| Model inference | 22 | GPU (T4): ~22 ms median; fits budget cleanly |
| Post-processing | 5 | Score thresholding + label mapping; remaining part of ~15 ms combined pre/post CPU work |
| Serialization | 3 | JSON response marshal |
| Network out | 5 | Half of 10 ms total network round-trip |
| **Headroom** | **182** | Buffer for p95 tail, GC pauses, queuing jitter |
| **Total** | **250** | ✅ Balances exactly |

> **Note on headroom:** 182 ms of headroom reflects that all component values above are **medians**, not p95 values. Under production load, queuing delay, GC pauses, and Redis tail latency realistically consume an additional 80–120 ms at p95. The remaining ~60–100 ms is a deliberate safety margin before the SLA refund threshold (500 ms) is approached.

---

## 2. CPU vs GPU Decision

**Decision: GPU (T4)**

| Option | Inference time | Throughput/replica (1 worker) | Approx. instance | On-demand price | Monthly cost (1 replica, 730 h) |
|---|---|---|---|---|---|
| CPU (8-core) | ~75 ms | ~13 RPS | `c5.2xlarge` (8 vCPU) | ~$0.34/h (AWS, us-east-1, 2024 estimate) | ~$248 |
| GPU (T4) | ~22 ms | ~45 RPS | `g4dn.xlarge` (4 vCPU + T4) | ~$0.526/h (AWS, us-east-1, 2024 estimate) | ~$384 |

Source: AWS on-demand pricing page, referenced values are approximations within 30%.

With GPU inference at 22 ms, a single replica can sustain ~45 RPS (accounting for 15 ms pre/post and ~5 ms overhead). This is 3.5× higher throughput per replica than CPU, meaning far fewer replicas are needed for the same load. The higher per-replica cost is offset by needing ~3× fewer instances. At scale (300–500 RPS), GPU is both cheaper and safely within the latency budget with meaningful headroom.

---

## 3. Replica Sizing

**Per-replica throughput (GPU):**
- Inference 22 ms + pre/post 15 ms + overhead ~8 ms = ~45 ms per request
- Single-threaded: ~22 RPS; with 2 concurrent workers per GPU: **~44 RPS**
- Apply 30% headroom buffer → effective capacity per replica: **~31 RPS**

| Scenario | Target RPS | Replicas needed (raw) | Replicas with 30% headroom | Monthly cost estimate |
|---|---|---|---|---|
| Sustained peak | 300 RPS | 300 / 44 ≈ 7 | **10 replicas** | 10 × $384 = **$3,840/mo** |
| Spike (5-min) | 500 RPS | 500 / 44 ≈ 12 | **17 replicas** | 17 × $384 = **$6,528/mo** *(spike only)* |

**Budget check:** Sustained 10 replicas = $3,840/mo — within the $4,000/mo compute budget. ✅

**Spike strategy: Autoscaling**
The 500 RPS spike is 5 minutes in duration. We will use **horizontal pod autoscaling (HPA)** triggered on GPU utilization > 70% or RPS > 280. Scale-out target is 17 replicas (500 RPS ÷ 31 effective RPS/replica, rounded up). Warm pool of 2 standby replicas will be maintained to reduce scale-out latency to under 60 seconds. We will not pre-provision 17 replicas at all times (too expensive), and we will not queue-shed for a predictable 5-minute spike — the SLA promise to partners makes shedding risky.

---

## 4. Batching Decision

**Decision: Enable dynamic batching on the GPU endpoint, disabled by default on the synchronous path.**

For the **batch endpoint** (10% of traffic, partner aggregations), dynamic batching is appropriate. Partners are tolerant of slightly higher latency in exchange for throughput. We recommend a **max batch size of 16** and a **max wait window of 20 ms**. At 22 ms median inference per image on a T4, a batch of 16 images processed together takes approximately 35–50 ms on the GPU (sublinear due to parallelism), compared to 16 × 22 ms = 352 ms if processed serially. This is well within the 250 ms synchronous budget and far inside any async partner SLA. The 20 ms wait window will typically be reached before the batch fills, ensuring queued images aren't held too long.

For the **synchronous endpoint** (90% of traffic, p95 ≤ 250 ms), dynamic batching is **not enabled**. The 250 ms budget is tight and any wait window introduces unpredictable tail latency. Each synchronous request will be dispatched to its own GPU worker immediately. If in future load testing shows GPU utilization > 85% with healthy latency headroom, we can revisit enabling batching with a max wait window of ≤ 5 ms on this path.
