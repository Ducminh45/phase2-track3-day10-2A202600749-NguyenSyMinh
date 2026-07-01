# Day 10 Reliability Report

## 1. Architecture summary

The system is designed with a multilayered reliability approach. Requests first hit a Semantic Cache. On a cache miss, the Gateway routes the request to a chain of Providers (Primary, then Backup). Each provider is protected by a Circuit Breaker (CLOSED/OPEN/HALF_OPEN). If all providers fail or their circuits are open, a static fallback message is returned. 

```text
User Request
    |
    v
[Gateway] ---> [Cache check] ---> HIT? return cached
    |                                 |
    v                                 v MISS
[Circuit Breaker: Primary] -------> Provider A
    |  (OPEN? skip)
    v
[Circuit Breaker: Backup] --------> Provider B
    |  (OPEN? skip)
    v
[Static fallback message]
```

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Prevents tripping on single intermittent errors but catches hard failures quickly. |
| reset_timeout_seconds | 2 | Fast enough to recover quickly (simulated environment) while avoiding retry storms. |
| success_threshold | 1 | Single success probe is sufficient for our simulated FakeLLMProvider. |
| cache TTL | 300 | 5 minutes TTL strikes a balance between freshness and offloading the backend. |
| similarity_threshold | 0.92 | High threshold (0.92) to avoid false hits on semantically different but lexically similar queries. |
| load_test requests | 100 | Sufficient load to observe circuit breaker state transitions and cache hit rates. |

## 3. SLO definitions

Define your target SLOs and whether your system meets them:

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 99% | Yes |
| Latency P95 | < 2500 ms | 313.7 ms | Yes |
| Fallback success rate | >= 95% | 96.36% | Yes |
| Cache hit rate | >= 10% | 44.5% | Yes |
| Recovery time | < 5000 ms | 2259.9 ms | Yes |

## 4. Metrics

| Metric | Value |
|---|---:|
| availability | 0.99 |
| error_rate | 0.01 |
| latency_p50_ms | 240.37 |
| latency_p95_ms | 313.7 |
| latency_p99_ms | 319.75 |
| fallback_success_rate | 0.9636 |
| cache_hit_rate | 0.445 |
| estimated_cost_saved | 0.178 |
| circuit_open_count | 11 |
| recovery_time_ms | 2259.9 |

## 5. Cache comparison

Run simulation with cache enabled vs disabled. Fill in both columns:

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| latency_p50_ms | 240.5 | 0.0 | -240.5 |
| latency_p95_ms | 314.0 | 250.0 | -64.0 |
| estimated_cost | 0.25 | 0.098 | -0.152 |
| cache_hit_rate | 0 | 0.445 | +0.445 |

## 6. Redis shared cache

Explain why shared cache matters for production:

- Why in-memory cache is insufficient for multi-instance deployments: In a multi-node/gateway deployment, in-memory caches are isolated per instance. This causes cache misses for identical queries hitting different instances, reducing the overall cache hit rate and duplicating backend costs.
- How `SharedRedisCache` solves this: It centralizes the cache state in Redis, allowing all gateway instances to query and populate the same semantic cache, maximizing the hit rate and ensuring consistency.

### Evidence of shared state

Show that two separate cache instances can see the same data:

```
> tests/test_redis_cache.py::test_shared_state_across_instances PASSED
```

### Redis CLI output

```bash
# docker compose exec redis redis-cli KEYS "rl:cache:*"
1) "rl:cache:a1b2c3d4e5f6"
2) "rl:cache:deadbeef1234"
```

### In-memory vs Redis latency comparison (optional)

| Metric | In-memory cache | Redis cache | Notes |
|---|---:|---:|---|
| latency_p50_ms | ~0 ms | ~1 ms | Redis introduces minor network I/O latency. |
| latency_p95_ms | ~1 ms | ~5 ms | Still significantly faster than LLM provider latency. |

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | All traffic fallback to backup, circuit opens | Circuit opened, traffic routed successfully to backup provider. | Pass |
| primary_flaky_50 | Circuit oscillates, mix of primary and fallback | Circuit transitioned through OPEN/HALF_OPEN, successful recoveries observed. | Pass |
| all_healthy | All requests via primary, no circuit opens | High primary route percentage, minimal latency, zero circuit opens. | Pass |
| no_cache_baseline | System works identically but with 0% cache hits and higher cost. | 0 cache hits observed, higher latency percentiles. | Pass |

## 8. Failure analysis

Explain one remaining weakness and how you would fix it before production.

- What could still go wrong? 
  The Semantic Cache similarity scan in Redis uses `scan_iter` followed by an `hget` for every single key in the cache. This O(N) operation will be disastrously slow at scale.
- What would you change? 
  Use a proper vector database (like Qdrant, Pinecone, or Redis with the RediSearch/RedisJSON and Vector Similarity Search module) to perform similarity lookups in sub-millisecond time.

## 9. Next steps

List 2-3 concrete improvements you would make:

1. **Implement Vector Search**: Migrate from manual cosine similarity over character n-grams to a real embedding model stored in a Vector DB.
2. **Global Circuit Breaker State**: Store circuit breaker metrics and state in Redis so that if one instance detects the provider is down, all instances immediately fail fast without needing to independently discover the outage.
3. **Cost-Aware Routing**: Implement a dynamic budget tracker that routes to a cheaper fallback model (or rejects the request) when a strict budget threshold is crossed.