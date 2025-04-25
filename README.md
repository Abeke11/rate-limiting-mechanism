## Redis-backed Token-Bucket Rate Limiter

### 1. Overview  
All inbound HTTP requests pass through a lightweight middleware that decrements a **token-bucket** counter stored in Redis.  
If the bucket for a client key is empty, the request is immediately rejected with **HTTP 429**.  
Buckets refill linearly over time, smoothing bursts and preventing overload.

### 2. Why token-bucket?  
| Requirement | Reason token-bucket fits |
|-------------|-------------------------|
| Allow short spikes without harming steady flow | Excess tokens (= burst capacity) absorb brief surges |
| Simple, O(1) check per request | atomic `DECR` in Redis |
| Works identically on N ≥ 1 service instances | Redis is the single source of truth |
| Fine-grained tuning per client / API-key | bucket size & refill rate per key |

### 3. Middleware logic (pseudo)  
```pseudo
key = rateLimitKey(req)           // IP, API-key, or user-id
tokens = redis.decr(key)          // atomic
if tokens < 0:
    return 429 Too Many Requests
else
    continue → handler()
