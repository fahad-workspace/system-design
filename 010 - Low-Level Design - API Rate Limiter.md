# Low-Level Design: API Rate Limiter

## Table of Contents
- [Key Design Considerations](#key-design-considerations)
- [Rate Limiting Algorithms](#rate-limiting-algorithms)
- [Java Implementation (Class Design & Code)](#java-implementation-class-design--code)
- [Distributed Rate Limiting Considerations](#distributed-rate-limiting-considerations)
- [Performance and Scaling](#performance-and-scaling)
- [Common Pitfalls and How to Avoid Them](#common-pitfalls-and-how-to-avoid-them)
- [Best Practices for Implementation & Maintenance](#best-practices-for-implementation--maintenance)
- [How to Discuss This in an Interview](#how-to-discuss-this-in-an-interview)

## Key Design Considerations

- **Per-Client Limits**: The rate limiter enforces limits on a per-client basis (e.g. per API key or IP). Each client has its own allowance of requests in a given time frame to ensure one client does not impact others.
- **Throughput (N Requests/Sec)**: Define the maximum number of requests (N) allowed per second or per window. Different limits may apply for different user tiers or APIs.
- **Multi-Threaded Safety**: In a multi-threaded environment, use proper locking or atomic operations to avoid race conditions when checking and updating counters under high concurrency.
- **In-Memory vs Distributed**: On a single server, an in-memory counter or token bucket is sufficient. In a distributed setup with multiple servers, use a centralized store or gateway to keep global counts consistent.
- **Configurability**: Allow easy adjustment of rate limits (through config files or environment variables). Different endpoints or clients can have distinct thresholds.
- **Burst Handling**: Decide whether short bursts are allowed within an average rate or if strict request spacing is required.
- **Ease of Use**: The component should be easy to integrate (e.g., a function call before processing each request). It must be efficient so it adds minimal latency.

## Rate Limiting Algorithms

Common approaches include:

- **Fixed Window Counter**: Tracks requests in a fixed time window (e.g. 1 minute). Easy to implement but may allow bursty traffic around window boundaries.
- **Sliding Window Log**: Uses a rolling time window, keeping timestamps of recent requests. More accurate but can be memory-heavy and more complex to manage.
- **Token Bucket**: A bucket refills tokens at a steady rate, and each request consumes a token. Allows bursts up to the bucket capacity while enforcing an average rate.
- **Leaky Bucket**: Requests enter a bucket drained at a fixed rate. Spikes are smoothed out because overflow is either queued or dropped.

The choice depends on whether bursts are acceptable and how strictly the average rate must be enforced. Token Bucket is often ideal for many APIs, allowing short bursts while controlling overall rate.

## Java Implementation (Class Design & Code)

Below is a simplified Token Bucket rate limiter in Java. It stores a token bucket per client using a thread-safe map. Each bucket tracks current tokens, last refill time, and a synchronized method to consume a token.

```java
import java.util.concurrent.ConcurrentHashMap;

public class TokenBucketRateLimiter {

    private final int capacity;            // max tokens
    private final double refillRatePerSec; // tokens added per second
    private final ConcurrentHashMap<String, TokenBucket> buckets = new ConcurrentHashMap<>();

    public TokenBucketRateLimiter(int capacity, double refillRatePerSec) {
        this.capacity = capacity;
        this.refillRatePerSec = refillRatePerSec;
    }

    private class TokenBucket {
        private double tokens;
        private long lastRefillTimeNano;

        public TokenBucket() {
            this.tokens = capacity;  // start full
            this.lastRefillTimeNano = System.nanoTime();
        }

        public synchronized boolean tryConsume() {
            long now = System.nanoTime();
            double secondsSinceLast = (now - lastRefillTimeNano) / 1e9;
            double tokensToAdd = secondsSinceLast * refillRatePerSec;

            if (tokensToAdd > 0) {
                tokens = Math.min(capacity, tokens + tokensToAdd);
                lastRefillTimeNano = now;
            }

            if (tokens >= 1) {
                tokens -= 1;
                return true;
            } else {
                return false;
            }
        }
    }

    public boolean allowRequest(String clientId) {
        TokenBucket bucket = buckets.computeIfAbsent(clientId, k -> new TokenBucket());
        return bucket.tryConsume();
    }
}
```

Usage example in a Spring Boot filter:

```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {

    @Autowired
    private TokenBucketRateLimiter rateLimiter;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain)
            throws ServletException, IOException {

        String clientKey = request.getHeader("X-API-Key");
        if (clientKey == null) {
            clientKey = request.getRemoteAddr();
        }

        if (!rateLimiter.allowRequest(clientKey)) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value()); // 429
            response.setHeader("Retry-After", "1");
            response.getWriter().write("{\"error\": \"Rate limit exceeded\"}");
            return;
        }

        chain.doFilter(request, response);
    }
}
```

The filter checks the rate limiter for each request and returns `429 Too Many Requests` if the client exceeded its tokens.

## Distributed Rate Limiting Considerations

In a multi-server environment, each instance needs a shared view of usage:

- **Centralized Data Store**: Use a system like Redis to maintain counters or token buckets across nodes. Update and read from Redis atomically so no two servers allow more requests than intended.
- **Sticky Sessions**: Route the same client to the same server. This avoids a shared store but can lead to uneven load and lacks fault tolerance if a server goes down.
- **API Gateway**: Offload rate limiting to an API gateway or reverse proxy that applies the limits before traffic reaches your application servers. The gateway can use an internal store for counters.
- **Failover and Consistency**: Plan how the system behaves if the shared store is unavailable. Some implementations fail-open (allow traffic), others fail-closed (block). Maintain consistent counting with atomic operations or scripts.

## Performance and Scaling

- **In-Memory Overhead**: Checking an in-memory bucket is fast. Just ensure thread safety. For many distinct clients, remove or expire old buckets to prevent memory bloat.
- **Distributed Overhead**: Each request involves a network call to Redis or a gateway. This adds slight latency but typically remains acceptable.
- **Scaling the Store**: Under very high load, Redis or the gateway could become a bottleneck. Use clustering or partitioning if needed.
- **Shedding Excess Load**: Rate limiting can enhance performance by rejecting abusive traffic quickly (HTTP 429). This prevents overload of the rest of the system.

## Common Pitfalls and How to Avoid Them

- **Race Conditions**: Always use atomic increments or synchronization for counters. Avoid separate get/set operations that can lead to exceeding limits.
- **Boundary Burst**: Fixed windows can allow double bursts at window edges. Consider sliding window or token bucket if this is problematic.
- **Single Point of Failure**: A single Redis node or gateway can break everything if it fails. Use replication, clustering, or fallback approaches.
- **Memory Leaks**: Remove stale entries for inactive clients if storing them in a map or external store.
- **Misconfigured Limits**: Setting limits too strict can block legitimate users; too loose may not prevent abuse. Monitor usage patterns and adjust as needed.

## Best Practices for Implementation & Maintenance

- **Algorithm Choice**: Pick token bucket for bursty traffic or leaky bucket for smoothing. Possibly combine strategies (e.g., daily fixed window plus real-time token bucket).
- **Use Existing Libraries**: Many frameworks or libraries provide well-tested rate limiter implementations. Leverage them to reduce bugs and complexity.
- **Monitoring and Logging**: Track how many requests are being throttled. Provide user-facing headers like `X-RateLimit-Remaining` and `Retry-After` to help clients adapt.
- **Testing**: Write unit tests simulating various rates and concurrency. Perform load tests to ensure the rate limiter and any store (e.g., Redis) can handle the demand.
- **Dynamic Updates**: Keep rate limit settings configurable. Consider rolling out new limits gradually or allowing different plans for different clients.

## How to Discuss This in an Interview

- **Start with Requirements**: Clarify the needed rate, client basis, concurrency, and distribution. 
- **Compare Algorithms**: Mention fixed window, sliding window, token bucket, and leaky bucket, highlighting pros and cons.
- **Walk Through an Implementation**: Outline the data structures and concurrency control. Show how to integrate it into a web request flow.
- **Handle Distributed Cases**: Explain how a shared store or gateway can enforce consistent limits across multiple servers.
- **Address Pitfalls**: Note race conditions, boundary bursts, single points of failure, and misconfiguration issues.
- **Demonstrate Trade-Offs**: Be ready to discuss performance overhead, burst tolerance, and how to scale out the solution.
