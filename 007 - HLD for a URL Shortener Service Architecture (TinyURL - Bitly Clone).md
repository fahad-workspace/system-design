# High-Level Design: URL Shortener Service Architecture (TinyURL/Bitly Clone)

## Table of Contents
- [Overview & Core Requirements](#overview--core-requirements)
- [Major Microservices & Responsibilities](#major-microservices--responsibilities)
  - [URL Shortening Service](#url-shortening-service)
  - [Redirection Service](#redirection-service)
  - [User Service](#user-service)
  - [Analytics Service](#analytics-service)
  - [Rate Limiting Service](#rate-limiting-service)
  - [URL Validation Service](#url-validation-service)
- [High-Level Architecture & Data Flow](#high-level-architecture--data-flow)
- [Key System Design Topics & Deep Dives](#key-system-design-topics--deep-dives)
  - [Short Code Generation Strategies](#short-code-generation-strategies)
  - [Database Design & Storage Considerations](#database-design--storage-considerations)
  - [Caching Architecture](#caching-architecture)
  - [Distribution & Partitioning](#distribution--partitioning)
  - [Security Fundamentals](#security-fundamentals)
  - [Analytics & Tracking](#analytics--tracking)
  - [Redirection Techniques](#redirection-techniques)
  - [Custom Short URLs](#custom-short-urls)
- [Performance & Scalability Considerations](#performance--scalability-considerations)
- [Common Pitfalls & How to Avoid Them](#common-pitfalls--how-to-avoid-them)
- [Best Practices & Maintenance](#best-practices--maintenance)
- [How to Discuss in a Principal Engineer Interview](#how-to-discuss-in-a-principal-engineer-interview)
- [Conclusion](#conclusion)

## Overview & Core Requirements

We want to build a **URL shortening service** where users can:

1. **Convert long URLs** into much shorter, easily shareable URLs.
2. **Redirect** visitors from short URLs to the original long URLs.
3. **Create custom short URLs** (vanity URLs) for branding purposes.
4. Access **click analytics** for their shortened links.
5. Set **expiration dates** for temporary links.
6. Create **bulk short URLs** for marketing campaigns.
7. Support **high-volume redirection** with minimal latency.
8. Provide **link management** for registered users.

### Additional Requirements

- **High availability** with 99.99%+ uptime (users expect links to work every time).
- **Extremely low latency** for redirects (< 100ms end-to-end).
- **Scalable** to billions of URLs and millions of redirects per second.
- **URL safety** checking to prevent malicious links.
- **Analytics** for click tracking, referrers, geographic data, etc.
- **API access** for enterprise integration.
- **Global distribution** to reduce latency worldwide.
- **Persistent storage** with strong consistency for URL mappings.
- **Prevention of short URL guessing/enumeration**.

## Major Microservices & Responsibilities

### URL Shortening Service
- **Responsibilities**: Handles creating new short URLs, storing mappings between short and long URLs, and managing URL metadata.
- **Data Store**: Typically a distributed NoSQL database (e.g., Cassandra, DynamoDB) for horizontal scaling.
- **Notes**: 
  - Generates unique short codes using various strategies.
  - Validates URLs before shortening them.
  - Handles custom/vanity URL requests.
  - Manages URL expiration.
  - May maintain counters for generating sequential IDs.

### Redirection Service
- **Responsibilities**: Efficiently redirects users from short URLs to original long URLs.
- **Data Store**: 
  - Heavily relies on distributed caching (Redis/Memcached) for performance.
  - Reads from primary storage if cache misses occur.
- **Notes**: 
  - Must be optimized for extremely low latency.
  - Handles most of the system's query traffic.
  - Publishes click events to an event stream.
  - Handles missing/expired URL logic.

### User Service
- **Responsibilities**: Manages user accounts, authentication, and authorization.
- **Data Store**: Relational database (e.g., PostgreSQL) for ACID compliance on user data.
- **Notes**:
  - Handles user registration and login.
  - Manages API keys for API access.
  - Controls user quotas and permissions.
  - Integrates with OAuth providers for social login.

### Analytics Service
- **Responsibilities**: Processes click data to provide insights on URL performance.
- **Data Store**:
  - Time-series database (e.g., InfluxDB, Timescale) for metrics.
  - Data warehouse (e.g., Snowflake, BigQuery) for long-term analytics.
- **Notes**:
  - Consumes click events from Kafka/Kinesis.
  - Aggregates data for dashboards and reports.
  - Handles real-time analytics in addition to historical data.
  - Processes geographic, device, and referrer information.

### Rate Limiting Service
- **Responsibilities**: Prevents abuse by limiting API and redirection requests.
- **Data Store**: Redis for distributed rate counters.
- **Notes**:
  - Implements token bucket or leaky bucket algorithms.
  - Differentiates between authenticated and anonymous users.
  - Provides different limits for different operations.
  - Communicates with API gateway for enforcement.

### URL Validation Service
- **Responsibilities**: Checks URLs for safety, reachability, and compliance.
- **Data Store**: May use databases of known malicious sites/patterns.
- **Notes**:
  - Validates that URLs are well-formed.
  - Checks against phishing/malware databases.
  - May perform safety scoring.
  - Possibly maintains a blacklist of domains.

## High-Level Architecture & Data Flow

1. **URL Shortening Flow**
   - User submits long URL via **UI/API Gateway**.
   - **URL Validation Service** checks for safety and validity.
   - **Rate Limiting Service** ensures user hasn't exceeded quota.
   - **URL Shortening Service** generates a unique short code.
   - Mapping is stored in the database and pre-warmed in cache.
   - Short URL is returned to the user.

2. **Redirection Flow**
   - User clicks a short URL and reaches the **Redirection Service**.
   - Service checks the **Redis Cache** for the short code → long URL mapping.
   - If found, returns HTTP 301/302 redirect to the long URL.
   - If not in cache, queries the database and updates cache.
   - **Click event** is published to Kafka/Kinesis (asynchronously).
   - **Analytics Service** processes the click event.

3. **Analytics Flow**
   - **Analytics Service** consumes click events from Kafka.
   - Real-time metrics are updated in the time-series database.
   - Aggregations are computed for dashboards.
   - Historical data is archived to data warehouse.
   - Users can query analytics through the **API Gateway**.

4. **User Management Flow**
   - Users register/login through the **User Service**.
   - Authentication tokens/sessions are managed.
   - User requests are authorized based on roles/permissions.
   - User dashboard provides link management capabilities.

5. **Distribution & Caching Strategy**
   - Short URL → Long URL mappings are distributed globally.
   - CDN edge locations serve common redirects for minimal latency.
   - Regional caches reduce database load.
   - Write-through caching ensures consistency.

## Key System Design Topics & Deep Dives

### Short Code Generation Strategies

**Concept & Principles**
- Short codes must be unique, compact, and difficult to guess.
- Various approaches offer different trade-offs in terms of length, predictability, and collision risk.
- The character set typically includes [0-9a-zA-Z], yielding 62 possible characters (Base62 encoding).

**Implementation Strategies**

1. **Counter-Based Approach**:
   - Maintain an auto-incrementing counter.
   - Convert decimal ID to Base62 for shorter representation.
   - Advantages: Simple, compact, guaranteed unique.
   - Disadvantages: Sequential, potentially predictable, requires central counter.

```java
public class Base62Encoder {
    private static final String CHARACTERS = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    private static final int BASE = CHARACTERS.length();
    
    public static String encode(long number) {
        if (number == 0) {
            return String.valueOf(CHARACTERS.charAt(0));
        }
        
        StringBuilder encoded = new StringBuilder();
        while (number > 0) {
            encoded.append(CHARACTERS.charAt((int) (number % BASE)));
            number /= BASE;
        }
        
        return encoded.reverse().toString();
    }
    
    public static long decode(String encoded) {
        long number = 0;
        for (char c : encoded.toCharArray()) {
            number = number * BASE + CHARACTERS.indexOf(c);
        }
        return number;
    }
}

@Service
public class SequentialUrlShortener {
    private final AtomicLong counter;
    private final UrlMappingRepository repository;
    
    public SequentialUrlShortener(UrlMappingRepository repository) {
        this.repository = repository;
        // Initialize counter to max ID in database + 1
        this.counter = new AtomicLong(repository.findMaxId() + 1);
    }
    
    public String shortenUrl(String longUrl) {
        long id = counter.getAndIncrement();
        String shortCode = Base62Encoder.encode(id);
        
        UrlMapping mapping = new UrlMapping(id, shortCode, longUrl);
        repository.save(mapping);
        
        return shortCode;
    }
}
```

2. **Random Generation**:
   - Generate random strings of fixed length (typically 6-8 characters).
   - Check for collisions and regenerate if necessary.
   - Advantages: Unpredictable, distributed generation.
   - Disadvantages: Collision probability increases with scale, requires uniqueness check.

```java
@Service
public class RandomUrlShortener {
    private static final String CHARACTERS = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    private static final int CODE_LENGTH = 7;
    private final Random random = new SecureRandom();
    private final UrlMappingRepository repository;
    
    public RandomUrlShortener(UrlMappingRepository repository) {
        this.repository = repository;
    }
    
    public String shortenUrl(String longUrl) {
        String shortCode;
        do {
            shortCode = generateRandomCode();
        } while (repository.existsByShortCode(shortCode));
        
        UrlMapping mapping = new UrlMapping(null, shortCode, longUrl);
        repository.save(mapping);
        
        return shortCode;
    }
    
    private String generateRandomCode() {
        StringBuilder sb = new StringBuilder(CODE_LENGTH);
        for (int i = 0; i < CODE_LENGTH; i++) {
            int randomIndex = random.nextInt(CHARACTERS.length());
            sb.append(CHARACTERS.charAt(randomIndex));
        }
        return sb.toString();
    }
}
```

3. **MD5/SHA-1 Hashing**:
   - Hash the long URL and take first few characters.
   - Same URL always produces the same short code.
   - Advantages: Deterministic, no counter needed.
   - Disadvantages: Collision possibility, reveals URL patterns.

```java
@Service
public class HashBasedUrlShortener {
    private final UrlMappingRepository repository;
    private static final int CODE_LENGTH = 7;
    
    public HashBasedUrlShortener(UrlMappingRepository repository) {
        this.repository = repository;
    }
    
    public String shortenUrl(String longUrl) {
        String hash = generateHash(longUrl);
        String shortCode = hash.substring(0, CODE_LENGTH);
        
        // Handle collisions by appending characters if needed
        int attempt = 0;
        String candidateCode = shortCode;
        while (repository.existsByShortCode(candidateCode)) {
            attempt++;
            int startPos = (CODE_LENGTH * attempt) % (hash.length() - CODE_LENGTH);
            candidateCode = hash.substring(startPos, startPos + CODE_LENGTH);
        }
        
        UrlMapping mapping = new UrlMapping(null, candidateCode, longUrl);
        repository.save(mapping);
        
        return candidateCode;
    }
    
    private String generateHash(String input) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] hashBytes = md.digest(input.getBytes(StandardCharsets.UTF_8));
            
            // Convert to Base62
            BigInteger number = new BigInteger(1, hashBytes);
            StringBuilder hashStr = new StringBuilder();
            
            String characters = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
            BigInteger base = BigInteger.valueOf(characters.length());
            
            while (number.compareTo(BigInteger.ZERO) > 0) {
                BigInteger[] divmod = number.divideAndRemainder(base);
                hashStr.append(characters.charAt(divmod[1].intValue()));
                number = divmod[0];
            }
            
            return hashStr.toString();
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException("Hashing algorithm not found", e);
        }
    }
}
```

4. **Distributed ID Generation**:
   - Use algorithms like Snowflake (from Twitter) or UUID.
   - Enables distributed, uncoordinated generation across many servers.
   - Advantages: No central coordination, high throughput.
   - Disadvantages: Longer IDs (must be encoded), some complexity.

```java
@Service
public class SnowflakeUrlShortener {
    private final SnowflakeIdGenerator idGenerator;
    private final UrlMappingRepository repository;
    
    public SnowflakeUrlShortener(UrlMappingRepository repository, 
                               @Value("${snowflake.node-id}") long nodeId) {
        this.repository = repository;
        this.idGenerator = new SnowflakeIdGenerator(nodeId);
    }
    
    public String shortenUrl(String longUrl) {
        long id = idGenerator.nextId();
        String shortCode = Base62Encoder.encode(id);
        
        UrlMapping mapping = new UrlMapping(id, shortCode, longUrl);
        repository.save(mapping);
        
        return shortCode;
    }
    
    // Simplified Snowflake implementation
    private static class SnowflakeIdGenerator {
        private static final long EPOCH = 1609459200000L; // Custom epoch (2021-01-01)
        private static final long NODE_ID_BITS = 10L;
        private static final long SEQUENCE_BITS = 12L;
        
        private final long nodeId;
        private long lastTimestamp = -1L;
        private long sequence = 0L;
        
        public SnowflakeIdGenerator(long nodeId) {
            this.nodeId = nodeId;
        }
        
        public synchronized long nextId() {
            long timestamp = System.currentTimeMillis();
            
            if (timestamp < lastTimestamp) {
                throw new RuntimeException("Clock moved backwards");
            }
            
            if (timestamp == lastTimestamp) {
                sequence = (sequence + 1) & ((1 << SEQUENCE_BITS) - 1);
                if (sequence == 0) {
                    // Sequence exhausted, wait for next millisecond
                    timestamp = waitNextMillis(lastTimestamp);
                }
            } else {
                sequence = 0;
            }
            
            lastTimestamp = timestamp;
            
            return ((timestamp - EPOCH) << (NODE_ID_BITS + SEQUENCE_BITS)) |
                   (nodeId << SEQUENCE_BITS) |
                   sequence;
        }
        
        private long waitNextMillis(long lastTimestamp) {
            long timestamp = System.currentTimeMillis();
            while (timestamp <= lastTimestamp) {
                timestamp = System.currentTimeMillis();
            }
            return timestamp;
        }
    }
}
```

5. **Key Generation Service (KGS)**:
   - Pre-generate short codes and store them.
   - Fetch available codes as needed.
   - Advantages: No collisions, fast allocation, flexible.
   - Disadvantages: Additional service to maintain, requires pre-allocation.

```java
@Service
public class KeyGenerationService {
    private final Queue<String> availableKeys;
    private final KeyRepository keyRepository;
    private final Object lock = new Object();
    private static final int BUFFER_SIZE = 1000;
    private static final int REFILL_THRESHOLD = 100;
    
    @Autowired
    public KeyGenerationService(KeyRepository keyRepository) {
        this.keyRepository = keyRepository;
        this.availableKeys = new ConcurrentLinkedQueue<>();
        refillKeys();
    }
    
    public String getNextKey() {
        String key = availableKeys.poll();
        
        if (key == null) {
            synchronized (lock) {
                // Double-check after acquiring lock
                key = availableKeys.poll();
                if (key == null) {
                    refillKeys();
                    key = availableKeys.poll();
                }
            }
        }
        
        // Trigger async refill if getting low
        if (availableKeys.size() < REFILL_THRESHOLD) {
            CompletableFuture.runAsync(this::refillKeys);
        }
        
        keyRepository.markAsUsed(key);
        return key;
    }
    
    private void refillKeys() {
        synchronized (lock) {
            if (availableKeys.size() >= BUFFER_SIZE) {
                return; // Already refilled by another thread
            }
            
            List<String> keys = keyRepository.getUnusedKeys(BUFFER_SIZE - availableKeys.size());
            availableKeys.addAll(keys);
            
            // If we couldn't get enough keys, generate more
            if (availableKeys.size() < BUFFER_SIZE) {
                generateNewKeys(BUFFER_SIZE - availableKeys.size());
            }
        }
    }
    
    private void generateNewKeys(int count) {
        List<String> newKeys = new ArrayList<>(count);
        
        for (int i = 0; i < count; i++) {
            String key;
            do {
                key = generateRandomKey();
            } while (keyRepository.exists(key));
            
            newKeys.add(key);
        }
        
        keyRepository.saveAll(newKeys);
        availableKeys.addAll(newKeys);
    }
    
    private String generateRandomKey() {
        // Implementation similar to RandomUrlShortener
        // ...
    }
}
```

**Performance & Scaling**
- Random generation scales well horizontally but collision checks become costly at scale.
- Counter-based approaches need distributed counter services (e.g., Zookeeper).
- Hash-based needs collision detection but offers URL deduplication.
- KGS approach enables efficient pre-generation and bulk allocation.

**Pitfalls & Best Practices**
- **Sequential ID Exposure**: Can reveal usage patterns or total URL count.
- **Collision Handling**: Critical for hash or random-based approaches.
- **URL Entropy**: Short codes should be unpredictable to prevent scanning.
- **ID Exhaustion**: Plan for sufficient ID space (7+ characters is recommended).
- Consider hybrid approaches (e.g., hash plus counter) for optimal systems.
- Pre-reserve special short codes for premium users or marketing campaigns.

### Database Design & Storage Considerations

**Concept & Principles**
- URL shorteners require a mapping from short code to long URL.
- Read-to-write ratio is heavily skewed toward reads (often 100:1 or higher).
- Data must be persistent and consistent across global deployments.
- Sizing considerations: URLs can be long (2KB+), but average around 100 bytes.

**Database Schema Design**

```sql
-- SQL Schema (PostgreSQL example)
CREATE TABLE url_mappings (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    user_id BIGINT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP,
    is_custom BOOLEAN NOT NULL DEFAULT FALSE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX idx_short_code ON url_mappings(short_code);
CREATE INDEX idx_user_id ON url_mappings(user_id);
CREATE INDEX idx_created_at ON url_mappings(created_at);
CREATE INDEX idx_expires_at ON url_mappings(expires_at) WHERE expires_at IS NOT NULL;

-- Click tracking table
CREATE TABLE click_events (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) NOT NULL,
    clicked_at TIMESTAMP NOT NULL DEFAULT NOW(),
    referrer TEXT,
    user_agent TEXT,
    ip_address VARCHAR(45),
    country_code VARCHAR(2),
    
    FOREIGN KEY (short_code) REFERENCES url_mappings(short_code)
);

-- Partition by date to optimize performance
CREATE TABLE click_events_y2025m03 PARTITION OF click_events
    FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');
```

**NoSQL Schema (DynamoDB/Cassandra Example)**

For distributed NoSQL databases:

```
// URL Mappings Table
{
    "short_code": "abc123", // Partition key
    "long_url": "https://example.com/very/long/path...",
    "user_id": "user123",
    "created_at": "2025-03-07T12:00:00Z",
    "expires_at": "2025-12-31T00:00:00Z",
    "is_custom": false,
    "is_active": true
}

// Click Events Table
{
    "short_code": "abc123", // Partition key
    "timestamp": "2025-03-07T12:05:23Z", // Sort key
    "referrer": "https://twitter.com",
    "user_agent": "Mozilla/5.0...",
    "ip_address": "192.168.1.1",
    "country_code": "US"
}
```

**Database Selection Considerations**

1. **SQL Databases (PostgreSQL/MySQL)**:
   - Advantages: ACID compliance, mature ecosystem, complex queries.
   - Disadvantages: Limited horizontal scaling, potential bottlenecks at high scale.
   - Best for: Smaller deployments, systems requiring strong consistency.

2. **NoSQL Databases (Cassandra/DynamoDB)**:
   - Advantages: Horizontal scalability, high write throughput, flexible schema.
   - Disadvantages: Limited query capabilities, eventual consistency model.
   - Best for: Large-scale systems with simple access patterns, global distribution.

3. **Key-Value Stores (Redis/Aerospike)**:
   - Advantages: Extremely fast lookups, simple interface.
   - Disadvantages: Limited persistence guarantees (Redis), data size constraints.
   - Best for: Cache layer, not typically the primary storage.

4. **Hybrid Approaches**:
   - Use SQL for user management, NoSQL for URL mappings.
   - Maintain SQL master record with NoSQL read replicas.
   - Cache hot mappings in Redis with persistent backing store.

**Performance & Scaling**
- **Sharding**: Distribute data by short code or user ID hash.
- **Replication**: Maintain multiple replicas for read scaling and availability.
- **Indexing**: Proper indexing of short_code field is critical for performance.
- **Data Growth**: Billions of URLs can consume terabytes of storage.

**Pitfalls & Best Practices**
- **Write Amplification**: Analytics data can grow much faster than URL mappings.
- **URL Length**: Some URLs can be extremely long; ensure sufficient storage.
- **Time-Series Data**: Click events grow rapidly; consider partitioning by time.
- **Hot Spots**: Popular URLs can cause read hot spots; address with caching.
- Consider archiving old/expired URLs to cold storage.
- Use time-to-live (TTL) features for expiring URLs where supported.
- Implement soft deletes to maintain referential integrity.

### Caching Architecture

**Concept & Principles**
- Caching is crucial for URL shorteners due to read-heavy workload.
- URL lookups should be satisfied from cache for most traffic.
- Cache hit rates of 95%+ are typical for well-tuned systems.
- Multi-level caching can optimize for both latency and cost.

**Caching Layers**

1. **Application-Level Cache**:
   - In-memory caches within application instances.
   - Limited capacity but extremely fast.
   - Useful for handling repeated requests to same instance.

2. **Distributed Cache (Redis/Memcached)**:
   - Primary caching layer for URL mappings.
   - Shared across all application instances.
   - Typically stores millions of frequently accessed mappings.

3. **CDN Edge Caching**:
   - For extremely popular URLs, CDN can cache the redirection response.
   - Reduces load on application servers entirely.
   - Requires careful cache control headers.

**Java Implementation Example (with Redis)**

```java
@Service
public class RedirectionService {
    private final RedisTemplate<String, String> redisTemplate;
    private final UrlMappingRepository repository;
    private final ClickEventPublisher eventPublisher;
    
    private static final Duration CACHE_TTL = Duration.ofDays(7);
    private static final String CACHE_KEY_PREFIX = "url:";
    
    @Autowired
    public RedirectionService(RedisTemplate<String, String> redisTemplate,
                              UrlMappingRepository repository,
                              ClickEventPublisher eventPublisher) {
        this.redisTemplate = redisTemplate;
        this.repository = repository;
        this.eventPublisher = eventPublisher;
    }
    
    public Optional<String> getLongUrl(String shortCode) {
        // Check cache first
        String cacheKey = CACHE_KEY_PREFIX + shortCode;
        String cachedUrl = redisTemplate.opsForValue().get(cacheKey);
        
        if (cachedUrl != null) {
            // Cache hit
            if (cachedUrl.isEmpty()) {
                // We cached a non-existent URL with empty string
                return Optional.empty();
            }
            
            // Publish click event asynchronously (don't block redirection)
            eventPublisher.publishClickEvent(shortCode);
            return Optional.of(cachedUrl);
        }
        
        // Cache miss, check database
        Optional<UrlMapping> mapping = repository.findByShortCode(shortCode);
        
        if (mapping.isPresent()) {
            UrlMapping urlMapping = mapping.get();
            
            // Check if URL is expired
            if (urlMapping.getExpiresAt() != null && 
                urlMapping.getExpiresAt().isBefore(Instant.now())) {
                // Expired URL - cache negative result
                redisTemplate.opsForValue().set(cacheKey, "", CACHE_TTL);
                return Optional.empty();
            }
            
            // Valid URL - cache the mapping
            String longUrl = urlMapping.getLongUrl();
            redisTemplate.opsForValue().set(cacheKey, longUrl, CACHE_TTL);
            
            // Publish click event asynchronously
            eventPublisher.publishClickEvent(shortCode);
            return Optional.of(longUrl);
        } else {
            // URL not found - cache negative result with empty string
            redisTemplate.opsForValue().set(cacheKey, "", CACHE_TTL);
            return Optional.empty();
        }
    }
    
    // Method to invalidate cache when URL is updated or deleted
    public void invalidateCache(String shortCode) {
        String cacheKey = CACHE_KEY_PREFIX + shortCode;
        redisTemplate.delete(cacheKey);
    }
}
```

**Cache Configuration with Redis**

```java
@Configuration
public class RedisCacheConfig {
    
    @Bean
    public RedisConnectionFactory redisConnectionFactory(
            @Value("${redis.host}") String host,
            @Value("${redis.port}") int port) {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration(host, port);
        return new LettuceConnectionFactory(config);
    }
    
    @Bean
    public RedisTemplate<String, String> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, String> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new StringRedisSerializer());
        return template;
    }
    
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofDays(7))
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .build();
    }
}
```

**Caching Strategies**

1. **Cache-Aside (Lazy Loading)**:
   - Application first checks cache, then database if cache miss.
   - Updates cache with database result.
   - Implemented in the example above.

2. **Write-Through**:
   - Cache is updated synchronously when database is updated.
   - Ensures consistency but adds write latency.

```java
@Service
public class UrlShorteningService {
    private final UrlMappingRepository repository;
    private final RedisTemplate<String, String> redisTemplate;
    private static final String CACHE_KEY_PREFIX = "url:";
    private static final Duration CACHE_TTL = Duration.ofDays(7);
    
    // Constructor injection...
    
    @Transactional
    public String createShortUrl(String longUrl, String customCode) {
        // Create mapping in database
        UrlMapping mapping = new UrlMapping();
        mapping.setLongUrl(longUrl);
        mapping.setShortCode(customCode);
        mapping.setCreatedAt(Instant.now());
        repository.save(mapping);
        
        // Write-through to cache
        String cacheKey = CACHE_KEY_PREFIX + mapping.getShortCode();
        redisTemplate.opsForValue().set(cacheKey, longUrl, CACHE_TTL);
        
        return mapping.getShortCode();
    }
}
```

3. **Cache Prewarming**:
   - Proactively populate cache with expected hot URLs.
   - Especially useful after deployments or cache failures.

```java
@Component
public class CacheWarmer {
    private final UrlMappingRepository repository;
    private final RedisTemplate<String, String> redisTemplate;
    private static final String CACHE_KEY_PREFIX = "url:";
    private static final Duration CACHE_TTL = Duration.ofDays(7);
    
    // Constructor injection...
    
    @PostConstruct
    @Scheduled(fixedRate = 24 * 60 * 60 * 1000) // Daily
    public void warmCache() {
        log.info("Starting cache warming process");
        
        // Get the most frequently accessed URLs (e.g., top 10,000)
        List<UrlMapping> popularUrls = repository.findMostAccessedUrls(10_000);
        
        int count = 0;
        for (UrlMapping mapping : popularUrls) {
            String cacheKey = CACHE_KEY_PREFIX + mapping.getShortCode();
            redisTemplate.opsForValue().set(cacheKey, mapping.getLongUrl(), CACHE_TTL);
            count++;
            
            if (count % 1000 == 0) {
                log.info("Warmed {} URLs in cache", count);
            }
        }
        
        log.info("Completed cache warming with {} URLs", count);
    }
}
```

**Cache Invalidation Strategies**

1. **Time-Based (TTL)**:
   - Set expiration time on cache entries.
   - Simplest approach but can lead to stale data.

2. **Event-Based**:
   - Invalidate specific cache entries when the underlying data changes.
   - More precise but requires coordination across services.

3. **Version-Based**:
   - Include a version identifier in cache keys.
   - Update version when data changes, effectively invalidating old entries.

**Performance & Scaling**
- Redis clusters can handle millions of operations per second.
- Distributed caching eliminates single points of failure.
- Cache hit ratios above 95% dramatically reduce database load.
- Consider read-through vs. write-through based on update frequency.

**Pitfalls & Best Practices**
- **Cold Cache**: After deployment, cache hit rates are initially low.
- **Cache Stampede**: Many simultaneous requests for uncached data.
- **Consistency Issues**: Updates may not immediately reflect in cache.
- **Cache Size Management**: Set reasonable TTLs to prevent memory exhaustion.
- Implement circuit breakers for cache failures.
- Monitor cache hit ratio and eviction rates.
- Use consistent hashing for Redis cluster setups.
- Consider separate caches for different data types (mappings vs. analytics).

### Distribution & Partitioning

**Concept & Principles**
- URL shortener services often need global distribution for low latency.
- Data partitioning enables horizontal scaling of the database layer.
- Consistent hashing algorithms help distribute load evenly.
- Replication strategies ensure high availability and read scaling.

**Data Partitioning Strategies**

1. **Hash-Based Partitioning**:
   - Distribute URL mappings based on a hash of the short code.
   - Ensures even distribution across database shards.
   - Example: `shard_id = hash(short_code) % num_shards`

2. **Range-Based Partitioning**:
   - Divide URLs based on creation time or ID ranges.
   - Simplifies data archiving but can lead to hot spots.

3. **Directory-Based Partitioning**:
   - Maintain a lookup service that maps short codes to specific shards.
   - More flexible but adds complexity and potential bottleneck.

**Global Distribution Architecture**

1. **Multi-Region Deployment**:
   - Deploy application servers in multiple geographic regions.
   - Use global DNS routing (like Route 53 or Cloudflare).
   - Direct users to nearest region for lowest latency.

2. **Data Replication Strategies**:
   - **Master-Slave**: Single write region with read replicas.
   - **Multi-Master**: Allow writes in any region with conflict resolution.
   - **Geo-Partitioning**: Assign different regions ownership of different data partitions.

3. **CDN Integration**:
   - Use CDN edge locations for first-level response.
   - Cache common redirects at edge to minimize latency.

**Java Implementation (Regional Routing)**

```java
@Service
public class RegionalRoutingService {
    private final List<String> regions;
    private final ConsistentHashRouter router;
    
    public RegionalRoutingService(@Value("${app.regions}") List<String> regions) {
        this.regions = regions;
        this.router = new ConsistentHashRouter(regions, 10); // 10 virtual nodes per region
    }
    
    public String getRegionForShortCode(String shortCode) {
        return router.getNode(shortCode);
    }
    
    // Simplified consistent hash implementation
    private static class ConsistentHashRouter {
        private final TreeMap<Integer, String> ring = new TreeMap<>();
        
        public ConsistentHashRouter(List<String> nodes, int virtualNodesCount) {
            for (String node : nodes) {
                addNode(node, virtualNodesCount);
            }
        }
        
        private void addNode(String node, int virtualNodesCount) {
            for (int i = 0; i < virtualNodesCount; i++) {
                String virtualNode = node + "#" + i;
                int hash = getHash(virtualNode);
                ring.put(hash, node);
            }
        }
        
        public String getNode(String key) {
            if (ring.isEmpty()) {
                return null;
            }
            
            int hash = getHash(key);
            if (!ring.containsKey(hash)) {
                Map.Entry<Integer, String> entry = ring.ceilingEntry(hash);
                if (entry == null) {
                    entry = ring.firstEntry();
                }
                return entry.getValue();
            }
            return ring.get(hash);
        }
        
        private int getHash(String key) {
            return Math.abs(key.hashCode());
        }
    }
}
```

**Database Sharding Implementation**

```java
@Configuration
public class ShardingConfiguration {
    private final List<DataSource> shards;
    private final int shardCount;
    
    public ShardingConfiguration(@Value("${db.shard.count}") int shardCount,
                                @Value("${db.shard.urls}") List<String> shardUrls) {
        this.shardCount = shardCount;
        this.shards = createDataSources(shardUrls);
    }
    
    public DataSource getShardForCode(String shortCode) {
        int shardIndex = Math.abs(shortCode.hashCode() % shardCount);
        return shards.get(shardIndex);
    }
    
    // Database routing in Spring
    @Bean
    public DataSource routingDataSource() {
        ShardingDataSource dataSource = new ShardingDataSource();
        
        Map<Object, Object> targetDataSources = new HashMap<>();
        for (int i = 0; i < shards.size(); i++) {
            targetDataSources.put(i, shards.get(i));
        }
        
        dataSource.setTargetDataSources(targetDataSources);
        dataSource.setDefaultTargetDataSource(shards.get(0));
        
        return dataSource;
    }
    
    // Custom routing data source
    private static class ShardingDataSource extends AbstractRoutingDataSource {
        @Override
        protected Object determineCurrentLookupKey() {
            return ShardingContextHolder.getShardIndex();
        }
    }
    
    // Helper class to store current shard in ThreadLocal
    public static class ShardingContextHolder {
        private static final ThreadLocal<Integer> contextHolder = new ThreadLocal<>();
        
        public static void setShardIndex(int shardIndex) {
            contextHolder.set(shardIndex);
        }
        
        public static Integer getShardIndex() {
            return contextHolder.get();
        }
        
        public static void clear() {
            contextHolder.remove();
        }
    }
    
    private List<DataSource> createDataSources(List<String> shardUrls) {
        // Implementation to create DataSource for each shard URL
        // ...
    }
}
```

**Handling Cross-Shard Operations**

For operations that span multiple shards (like retrieving all URLs for a user):

```java
@Service
public class UserUrlService {
    private final ShardingConfiguration shardingConfig;
    private final UrlMappingRepository repository;
    
    // Constructor injection...
    
    public List<UrlMapping> getAllUrlsForUser(Long userId) {
        List<UrlMapping> allUrls = new ArrayList<>();
        
        // Query each shard for the user's URLs
        for (int i = 0; i < shardingConfig.getShardCount(); i++) {
            try {
                ShardingConfiguration.ShardingContextHolder.setShardIndex(i);
                List<UrlMapping> shardUrls = repository.findByUserId(userId);
                allUrls.addAll(shardUrls);
            } finally {
                ShardingConfiguration.ShardingContextHolder.clear();
            }
        }
        
        return allUrls;
    }
}
```

**Performance & Scaling**
- Each shard can handle a portion of the total traffic.
- Read replicas per shard multiply read capacity.
- Geographic distribution reduces latency for global users.
- Adding new shards enables nearly linear scaling.

**Pitfalls & Best Practices**
- **Resharding Complexity**: Adding/removing shards requires careful data migration.
- **Cross-Shard Transactions**: Avoid operations that span multiple shards.
- **Hot Spots**: Poor partitioning schemes can lead to uneven load.
- **Replication Lag**: Read replicas may temporarily show stale data.
- Use consistent hashing to minimize data movement during resharding.
- Consider shard isolation for large tenants or high-volume users.
- Implement regular shard rebalancing to maintain even distribution.
- Design for graceful degradation if a shard becomes unavailable.

### Security Fundamentals

**Concept & Principles**
- URL shorteners can be abused for phishing, malware distribution, or spam.
- Security encompasses both protecting the service and protecting users.
- Privacy considerations include what data is stored and how it's accessed.
- Compliance requirements may apply (GDPR, CCPA) for user data.

**Key Security Measures**

1. **URL Validation & Safety Checks**:
   - Prevent shortening of malicious URLs.
   - Check against known phishing/malware databases.
   - Scan for suspicious patterns or banned domains.

```java
@Service
public class UrlSafetyService {
    private final RestTemplate restTemplate;
    private final List<String> maliciousPatterns;
    private final List<String> blockedDomains;
    
    // Constructor injection...
    
    public SafetyCheckResult checkUrlSafety(String longUrl) {
        try {
            URI uri = new URI(longUrl);
            String domain = uri.getHost().toLowerCase();
            
            // Check against blocked domains
            if (blockedDomains.contains(domain)) {
                return SafetyCheckResult.unsafe("Domain is blocked");
            }
            
            // Check for suspicious patterns
            for (String pattern : maliciousPatterns) {
                if (longUrl.contains(pattern)) {
                    return SafetyCheckResult.suspicious("URL contains suspicious pattern");
                }
            }
            
            // Check with external API (like Google Safe Browsing)
            ThreatInfo threatInfo = checkWithExternalApi(longUrl);
            if (threatInfo.isThreat()) {
                return SafetyCheckResult.unsafe("URL flagged by threat intelligence");
            }
            
            return SafetyCheckResult.safe();
        } catch (URISyntaxException e) {
            return SafetyCheckResult.invalid("Malformed URL");
        }
    }
    
    private ThreatInfo checkWithExternalApi(String url) {
        // Implementation to call external threat intelligence API
        // ...
    }
    
    public static class SafetyCheckResult {
        private final boolean safe;
        private final String reason;
        private final SafetyLevel level;
        
        // Implementation with factory methods, getters, etc.
        // ...
        
        public enum SafetyLevel {
            SAFE, SUSPICIOUS, UNSAFE, INVALID
        }
    }
}
```

2. **Rate Limiting & Abuse Prevention**:
   - Limit requests per IP/user to prevent scraping or DoS.
   - Implement progressive penalties for abuse.
   - Use CAPTCHA for suspicious activity patterns.

```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {
    private final RedisTemplate<String, String> redisTemplate;
    private final int maxRequestsPerMinute;
    
    public RateLimitInterceptor(RedisTemplate<String, String> redisTemplate,
                               @Value("${ratelimit.requests-per-minute}") int maxRequestsPerMinute) {
        this.redisTemplate = redisTemplate;
        this.maxRequestsPerMinute = maxRequestsPerMinute;
    }
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) throws Exception {
        String ip = getClientIp(request);
        String key = "ratelimit:" + ip;
        
        // Get current count
        String countStr = redisTemplate.opsForValue().get(key);
        int count = (countStr != null) ? Integer.parseInt(countStr) : 0;
        
        if (count >= maxRequestsPerMinute) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.getWriter().write("Rate limit exceeded. Try again later.");
            return false;
        }
        
        // Increment count with TTL
        if (count == 0) {
            redisTemplate.opsForValue().set(key, "1", Duration.ofMinutes(1));
        } else {
            redisTemplate.opsForValue().increment(key);
        }
        
        return true;
    }
    
    private String getClientIp(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```

3. **Short Code Protection**:
   - Prevent enumeration or scanning of short codes.
   - Use random/unpredictable codes rather than sequential.
   - Consider rate limiting by IP for redirection attempts.

4. **Authentication & Authorization**:
   - Secure APIs with OAuth2/JWT tokens.
   - Implement proper RBAC for administrative functions.
   - Protect user-specific data and analytics.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .authorizeRequests()
            // Public endpoints
            .antMatchers(HttpMethod.GET, "/{shortCode:[a-zA-Z0-9]{1,10}}").permitAll()
            .antMatchers("/api/v1/public/**").permitAll()
            
            // API requires authentication
            .antMatchers("/api/v1/urls/**").authenticated()
            .antMatchers("/api/v1/analytics/**").authenticated()
            
            // Admin endpoints require ADMIN role
            .antMatchers("/api/v1/admin/**").hasRole("ADMIN")
            
            .anyRequest().authenticated()
            .and()
            .oauth2ResourceServer().jwt();
    }
}
```

5. **Data Privacy & Compliance**:
   - Store minimal necessary data for operations.
   - Implement data retention policies.
   - Provide transparency on what data is collected.
   - Allow users to delete their data.

6. **Secure Redirects & Warnings**:
   - Optionally provide warning/preview page for external links.
   - Add appropriate security headers to all responses.

```java
@Component
public class SecurityHeadersFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                   HttpServletResponse response, 
                                   FilterChain filterChain) throws IOException, ServletException {
        // Add security headers
        response.setHeader("X-Content-Type-Options", "nosniff");
        response.setHeader("X-Frame-Options", "DENY");
        response.setHeader("X-XSS-Protection", "1; mode=block");
        response.setHeader("Referrer-Policy", "strict-origin-when-cross-origin");
        response.setHeader("Content-Security-Policy", "default-src 'self'");
        
        filterChain.doFilter(request, response);
    }
}
```

**HTTPS & TLS Configuration**

- All endpoints should use HTTPS to prevent traffic interception.
- Implement secure TLS configuration with modern protocols and ciphers.
- Use proper certificate management with auto-renewal.

**Performance & Security Tradeoffs**
- Security checks add some latency to URL creation and redirection.
- Caching can mitigate performance impact of repeated checks.
- Real-time threat intelligence APIs may have rate limits or costs.

**Pitfalls & Best Practices**
- **Security vs. Usability**: Excessive security can frustrate legitimate users.
- **False Positives**: Overzealous URL filtering may block valid URLs.
- **Privacy Concerns**: Collect only necessary data with clear purpose.
- **Token Security**: Protect API keys and JWT tokens from exposure.
- Implement defense in depth with multiple security layers.
- Regularly update threat intelligence and blocked domains.
- Conduct security audits and penetration testing.
- Have clear incident response procedures for abuse or breaches.

### Analytics & Tracking

**Concept & Principles**
- URL shorteners often provide analytics as a key value proposition.
- Click tracking must be performant and not impact redirection latency.
- Data collection must balance usefulness with privacy considerations.
- Aggregate metrics vs. individual click data have different storage needs.

**Key Analytics Features**

1. **Click Tracking & Counting**:
   - Basic count of total clicks per short URL.
   - Temporal data (clicks over time, by hour/day/month).
   - Unique vs. repeat visitors.

2. **Referrer Tracking**:
   - Where clicks originated (websites, social media, email).
   - Campaign attribution with UTM parameters.

3. **Geographic Data**:
   - Country, region, and city of visitors.
   - Heatmaps of global usage.

4. **Device & Browser Analytics**:
   - Mobile vs. desktop usage.
   - Browser and operating system statistics.

5. **Conversion Tracking**:
   - Integration with destination sites for goal completion.
   - Click-through rates for marketing campaigns.

**Analytics Architecture**

1. **Event Collection Pipeline**:
   - Asynchronous event capture during redirection.
   - Queuing system (Kafka/Kinesis) to buffer events.
   - Consumers process events into analytics database.

```java
@Service
public class ClickEventPublisher {
    private final KafkaTemplate<String, ClickEvent> kafkaTemplate;
    private final String topicName;
    
    // Constructor injection...
    
    public void publishClickEvent(String shortCode, HttpServletRequest request) {
        // Extract data from request
        String referrer = request.getHeader("Referer");
        String userAgent = request.getHeader("User-Agent");
        String ipAddress = getClientIp(request);
        
        // Create event
        ClickEvent event = new ClickEvent();
        event.setShortCode(shortCode);
        event.setTimestamp(Instant.now());
        event.setReferrer(referrer);
        event.setUserAgent(userAgent);
        event.setIpAddress(ipAddress);
        
        // Send to Kafka asynchronously with callback
        kafkaTemplate.send(topicName, shortCode, event)
            .addCallback(
                result -> log.debug("Click event sent for {}", shortCode),
                ex -> log.error("Failed to send click event for {}: {}", shortCode, ex.getMessage())
            );
    }
    
    // Helper method to get client IP safely
    private String getClientIp(HttpServletRequest request) {
        // Implementation...
    }
}
```

2. **Real-time Processing**:
   - Stream processing for immediate metrics updates.
   - Update counters and dashboards in near real-time.

```java
@Component
public class ClickEventProcessor {
    private final TimeSeriesRepository timeSeriesRepository;
    private final CounterRepository counterRepository;
    private final GeoIpService geoIpService;
    
    // Constructor injection...
    
    @KafkaListener(topics = "${kafka.topic.click-events}", groupId = "${kafka.group.analytics}")
    public void processClickEvent(ClickEvent event) {
        // Enrich event with geo data
        String countryCode = geoIpService.getCountryCode(event.getIpAddress());
        event.setCountryCode(countryCode);
        
        // Update real-time counters
        String shortCode = event.getShortCode();
        counterRepository.incrementCounter("clicks:" + shortCode);
        counterRepository.incrementCounter("clicks:country:" + shortCode + ":" + countryCode);
        
        // Parse user agent for device info
        UserAgentInfo uaInfo = parseUserAgent(event.getUserAgent());
        String deviceType = uaInfo.getDeviceType(); // mobile, desktop, tablet
        counterRepository.incrementCounter("clicks:device:" + shortCode + ":" + deviceType);
        
        // Store time-series data point
        timeSeriesRepository.recordClick(
            shortCode, 
            event.getTimestamp(), 
            countryCode, 
            deviceType, 
            event.getReferrer()
        );
    }
    
    private UserAgentInfo parseUserAgent(String userAgent) {
        // Implementation using a library like UADetector or user-agent-utils
        // ...
    }
}
```

3. **Batch Processing**:
   - Aggregate raw events into summary tables.
   - Generate periodic reports and insights.
   - Archival of detailed data to cold storage.

4. **Data Visualization**:
   - Real-time dashboards for current metrics.
   - Historical charts and trend analysis.
   - Exportable reports for business users.

**Analytics API Implementation**

```java
@RestController
@RequestMapping("/api/v1/analytics")
public class AnalyticsController {
    private final AnalyticsService analyticsService;
    
    // Constructor injection...
    
    @GetMapping("/urls/{shortCode}/clicks")
    public ClickStatsResponse getClickStats(
            @PathVariable String shortCode,
            @RequestParam(required = false) 
                @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate from,
            @RequestParam(required = false) 
                @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate to) {
        
        LocalDate fromDate = (from != null) ? from : LocalDate.now().minusDays(30);
        LocalDate toDate = (to != null) ? to : LocalDate.now();
        
        return analyticsService.getClickStats(shortCode, fromDate, toDate);
    }
    
    @GetMapping("/urls/{shortCode}/countries")
    public List<CountryStatsResponse> getCountryStats(@PathVariable String shortCode) {
        return analyticsService.getCountryStats(shortCode);
    }
    
    @GetMapping("/urls/{shortCode}/referrers")
    public List<ReferrerStatsResponse> getReferrerStats(@PathVariable String shortCode) {
        return analyticsService.getReferrerStats(shortCode);
    }
    
    @GetMapping("/urls/{shortCode}/devices")
    public DeviceStatsResponse getDeviceStats(@PathVariable String shortCode) {
        return analyticsService.getDeviceStats(shortCode);
    }
    
    @GetMapping("/urls/{shortCode}/export")
    public ResponseEntity<Resource> exportClickData(
            @PathVariable String shortCode,
            @RequestParam(required = false) 
                @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate from,
            @RequestParam(required = false) 
                @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate to) {
        
        // Implementation to export CSV/Excel data
        // ...
    }
}
```

**Data Storage Considerations**

1. **Time-Series Database**:
   - Efficient storage for click events over time.
   - InfluxDB, TimescaleDB, or similar specialized DB.
   - Optimized for temporal queries and aggregations.

2. **Aggregation Tables**:
   - Pre-computed summaries for common queries.
   - Materialized views or dedicated tables.
   - Updated on schedule or incrementally with new data.

```sql
-- Example aggregation tables in PostgreSQL/TimescaleDB

-- Hourly aggregates
CREATE TABLE click_stats_hourly (
    short_code VARCHAR(10) NOT NULL,
    hour_bucket TIMESTAMP NOT NULL,
    click_count INT NOT NULL,
    unique_visitors INT NOT NULL,
    PRIMARY KEY (short_code, hour_bucket)
);

-- Country aggregates
CREATE TABLE click_stats_country (
    short_code VARCHAR(10) NOT NULL,
    country_code VARCHAR(2) NOT NULL,
    click_count INT NOT NULL,
    last_updated TIMESTAMP NOT NULL,
    PRIMARY KEY (short_code, country_code)
);

-- Referrer aggregates
CREATE TABLE click_stats_referrer (
    short_code VARCHAR(10) NOT NULL,
    referrer_domain VARCHAR(255) NOT NULL,
    click_count INT NOT NULL,
    last_updated TIMESTAMP NOT NULL,
    PRIMARY KEY (short_code, referrer_domain)
);
```

3. **NoSQL for Flexible Analytics**:
   - Schema flexibility for evolving metrics.
   - Document stores (MongoDB) for nested analytics data.
   - Column stores (Cassandra) for wide analytics tables.

**Performance & Scaling**
- Decouple click capturing from analytics processing.
- Use materialized views or aggregate tables for frequent queries.
- Implement data retention policies based on age/importance.
- Consider data sampling for ultra-high-volume URLs.

**Pitfalls & Best Practices**
- **Data Volume**: Click events can grow extremely large very quickly.
- **Query Performance**: Analytics queries can be resource-intensive.
- **Privacy Compliance**: Consider GDPR/CCPA requirements for user data.
- **Accuracy vs. Performance**: Some approximation is often acceptable.
- Use time-bucketed tables for efficient temporal queries.
- Anonymize IP addresses to protect privacy while maintaining geographic data.
- Implement proper data partitioning and archiving strategies.
- Consider approximation algorithms for high-cardinality metrics (e.g., HyperLogLog for unique visitors).

### Redirection Techniques

**Concept & Principles**
- Redirection is the core functionality of a URL shortener.
- Performance is critical: users expect near-instant redirects.
- HTTP redirect codes have different implications for SEO and tracking.
- Browser caching behavior varies by redirect type.

**HTTP Redirect Status Codes**

1. **301 (Moved Permanently)**:
   - Indicates the resource has permanently moved.
   - Browsers cache this redirect, potentially bypassing the shortener on subsequent visits.
   - Best for SEO as link equity is passed to the destination URL.

2. **302 (Found)** or **307 (Temporary Redirect)**:
   - Indicates a temporary redirect.
   - Browsers will check with the shortener server each time.
   - Better for analytics as every click goes through the shortener.

3. **303 (See Other)**:
   - Forces a GET request to the target URL.
   - Useful when redirecting from a POST operation.

**Java Implementation Example**

```java
@RestController
public class RedirectionController {
    private final RedirectionService redirectionService;
    private final ClickEventPublisher eventPublisher;
    
    // Constructor injection...
    
    @GetMapping("/{shortCode}")
    public ResponseEntity<Void> redirectToLongUrl(
            @PathVariable String shortCode,
            HttpServletRequest request) {
        
        Optional<String> longUrlOpt = redirectionService.getLongUrl(shortCode);
        
        if (longUrlOpt.isPresent()) {
            String longUrl = longUrlOpt.get();
            
            // Asynchronously publish click event
            eventPublisher.publishClickEvent(shortCode, request);
            
            // Determine redirect type (configurable)
            HttpStatus redirectType = HttpStatus.TEMPORARY_REDIRECT; // 307
            
            // Return the redirect
            return ResponseEntity
                .status(redirectType)
                .location(URI.create(longUrl))
                .build();
        } else {
            // URL not found
            return ResponseEntity.notFound().build();
        }
    }
    
    @GetMapping("/{shortCode}/preview")
    public ModelAndView previewUrl(@PathVariable String shortCode) {
        Optional<String> longUrlOpt = redirectionService.getLongUrl(shortCode);
        
        if (longUrlOpt.isPresent()) {
            ModelAndView mav = new ModelAndView("preview");
            mav.addObject("shortCode", shortCode);
            mav.addObject("longUrl", longUrlOpt.get());
            return mav;
        } else {
            return new ModelAndView("not-found");
        }
    }
}
```

**Advanced Redirection Features**

1. **URL Preview Page**:
   - Shows destination before redirecting.
   - Helps users avoid phishing or unexpected content.
   - Can display safety information or warnings.

2. **Smart Redirects**:
   - Device-specific destinations (mobile vs. desktop).
   - Geographic targeting (different URLs based on country).
   - A/B testing multiple destination URLs.

```java
@Service
public class SmartRedirectionService {
    private final UrlMappingRepository repository;
    private final GeoIpService geoIpService;
    private final DeviceDetectionService deviceService;
    
    // Constructor injection...
    
    public String getTargetUrl(String shortCode, HttpServletRequest request) {
        UrlMapping baseMapping = repository.findByShortCode(shortCode)
            .orElseThrow(() -> new NotFoundException("Short URL not found"));
        
        // Check for specific device targeting
        String userAgent = request.getHeader("User-Agent");
        DeviceType deviceType = deviceService.detectDevice(userAgent);
        
        if (deviceType == DeviceType.MOBILE && baseMapping.getMobileUrl() != null) {
            return baseMapping.getMobileUrl();
        }
        
        // Check for geo-targeting
        String ipAddress = getClientIp(request);
        String countryCode = geoIpService.getCountryCode(ipAddress);
        
        Optional<GeoTargetMapping> geoMapping = repository.findGeoMappingByShortCodeAndCountry(
            shortCode, countryCode);
            
        if (geoMapping.isPresent()) {
            return geoMapping.get().getTargetUrl();
        }
        
        // Fall back to default URL
        return baseMapping.getLongUrl();
    }
    
    // Helper method to get client IP
    private String getClientIp(HttpServletRequest request) {
        // Implementation...
    }
}
```

3. **Interstitial Pages**:
   - Display ads or custom content before redirecting.
   - Delay redirection with a countdown timer.
   - Show branding or promotional content.

4. **Deep Linking**:
   - Direct to specific app content on mobile devices.
   - Universal links (iOS) and App Links (Android) integration.
   - Fallback URLs for users without the app installed.

```java
@Service
public class DeepLinkingService {
    
    public RedirectStrategy getDeepLinkStrategy(String shortCode, HttpServletRequest request) {
        UrlMapping mapping = repository.findByShortCode(shortCode)
            .orElseThrow(() -> new NotFoundException("Short URL not found"));
        
        String userAgent = request.getHeader("User-Agent");
        
        if (isIosDevice(userAgent) && mapping.getIosAppLink() != null) {
            return new AppleUniversalLinkStrategy(mapping);
        } else if (isAndroidDevice(userAgent) && mapping.getAndroidAppLink() != null) {
            return new AndroidAppLinkStrategy(mapping);
        } else {
            return new WebRedirectStrategy(mapping);
        }
    }
    
    // Abstraction for different redirect strategies
    public interface RedirectStrategy {
        void applyToResponse(HttpServletResponse response);
    }
    
    // Implementation classes for each strategy
    // ...
}
```

**Performance Considerations**

1. **Latency Optimization**:
   - Keep redirect logic simple and fast.
   - Cache hot URLs in memory/Redis.
   - Use CDN edge locations for popular redirects.

2. **Header Tuning**:
   - Set appropriate cache-control headers based on redirect type.
   - Consider TTL for redirects that might change.

```java
// Example of setting cache headers for a 301 redirect
response.setHeader("Cache-Control", "public, max-age=86400"); // 1 day
response.setHeader("Expires", getExpiresHeaderValue(86400));

// Example of preventing caching for a 302 redirect
response.setHeader("Cache-Control", "no-cache, no-store, must-revalidate");
response.setHeader("Pragma", "no-cache");
response.setHeader("Expires", "0");
```

3. **Async Event Publishing**:
   - Decouple analytics from the redirect path.
   - Use thread pools or event queues for background processing.

**Pitfalls & Best Practices**
- **Redirect Loops**: Prevent shortening of already shortened URLs.
- **SEO Considerations**: 301 redirects pass link equity but limit analytics.
- **Caching Issues**: Browsers might cache 301 redirects permanently.
- **Mobile App Handling**: Consider app deep linking for better user experience.
- Test redirects across different browsers and devices.
- Monitor redirect latency as a key performance metric.
- Consider geography and network conditions for global users.
- Implement proper error handling for invalid or expired URLs.

### Custom Short URLs

**Concept & Principles**
- Custom short URLs (vanity URLs) allow users to create memorable, branded links.
- These URLs are typically a premium feature and require special handling.
- They may have different validation rules and conflict resolution strategies.
- They're often used for marketing campaigns or important brand links.

**Implementation Approaches**

1. **Custom Alias Reservation**:
   - Allow users to request specific aliases.
   - Check availability and reserve the alias.
   - May require authentication or payment.

```java
@Service
public class CustomUrlService {
    private final UrlMappingRepository repository;
    private final UserService userService;
    private final ReservedWordsValidator reservedWordsValidator;
    
    // Constructor injection...
    
    @Transactional
    public String createCustomUrl(String longUrl, String customAlias, Long userId) {
        // Validate custom alias format
        if (!isValidCustomAlias(customAlias)) {
            throw new InvalidAliasException("Custom alias contains invalid characters");
        }
        
        // Check against reserved words
        if (reservedWordsValidator.isReserved(customAlias)) {
            throw new ReservedAliasException("This alias is reserved and cannot be used");
        }
        
        // Check if user has permission for custom URLs
        User user = userService.getUserById(userId);
        if (!user.hasPermission(Permission.CREATE_CUSTOM_URL)) {
            throw new UnauthorizedException("User does not have permission to create custom URLs");
        }
        
        // Check availability
        if (repository.existsByShortCode(customAlias)) {
            throw new AliasAlreadyExistsException("This alias is already taken");
        }
        
        // Create the custom URL
        UrlMapping mapping = new UrlMapping();
        mapping.setShortCode(customAlias);
        mapping.setLongUrl(longUrl);
        mapping.setUserId(userId);
        mapping.setCreatedAt(Instant.now());
        mapping.setCustom(true);
        
        repository.save(mapping);
        
        return customAlias;
    }
    
    private boolean isValidCustomAlias(String alias) {
        // Only allow alphanumeric characters and hyphens, with length limits
        return alias.matches("^[a-zA-Z0-9-]{3,30}$");
    }
}
```

2. **Reserved Words Protection**:
   - Maintain a list of reserved words that cannot be claimed.
   - Protect brand names, offensive terms, and system paths.
   - Implement a moderation system for custom alias requests.

```java
@Component
public class ReservedWordsValidator {
    private final Set<String> reservedWords;
    private final List<Pattern> reservedPatterns;
    
    public ReservedWordsValidator(
            @Value("${url.reserved-words-file}") String reservedWordsFile) {
        this.reservedWords = loadReservedWords(reservedWordsFile);
        this.reservedPatterns = compilePatterns();
    }
    
    public boolean isReserved(String alias) {
        // Convert to lowercase for case-insensitive matching
        String lowerAlias = alias.toLowerCase();
        
        // Check exact matches
        if (reservedWords.contains(lowerAlias)) {
            return true;
        }
        
        // Check pattern matches
        for (Pattern pattern : reservedPatterns) {
            if (pattern.matcher(lowerAlias).matches()) {
                return true;
            }
        }
        
        return false;
    }
    
    private Set<String> loadReservedWords(String filename) {
        try {
            return Files.lines(Path.of(filename))
                .map(String::trim)
                .map(String::toLowerCase)
                .filter(word -> !word.isEmpty() && !word.startsWith("#"))
                .collect(Collectors.toSet());
        } catch (IOException e) {
            log.error("Failed to load reserved words file", e);
            // Fallback to a minimal default set
            return Set.of("admin", "api", "login", "status", "help", "support");
        }
    }
    
    private List<Pattern> compilePatterns() {
        // Add regex patterns for things like offensive words
        return List.of(
            Pattern.compile("\\badmin\\w*"),
            Pattern.compile("\\bapi\\b"),
            // Add more patterns as needed
            Pattern.compile("\\b(support|help)\\b")
        );
    }
}
```

3. **Domain-Based Customization**:
   - Offer custom domains for enterprise users.
   - Allow users to create branded short URLs with their own domain.
   - Requires DNS configuration and SSL certificate management.

```java
@Service
public class CustomDomainService {
    private final DomainRepository domainRepository;
    private final SslCertificateService certificateService;
    
    // Constructor injection...
    
    @Transactional
    public void registerCustomDomain(String domainName, Long userId) {
        // Validate domain format
        if (!isValidDomain(domainName)) {
            throw new InvalidDomainException("Invalid domain format");
        }
        
        // Check if domain is already registered
        if (domainRepository.existsByDomainName(domainName)) {
            throw new DomainAlreadyRegisteredException("This domain is already registered");
        }
        
        // Validate domain ownership (DNS TXT record verification)
        String verificationToken = generateVerificationToken();
        boolean verified = verifyDomainOwnership(domainName, verificationToken);
        
        if (!verified) {
            throw new DomainVerificationException("Domain ownership verification failed");
        }
        
        // Request SSL certificate
        certificateService.requestCertificate(domainName);
        
        // Save domain record
        CustomDomain domain = new CustomDomain();
        domain.setDomainName(domainName);
        domain.setUserId(userId);
        domain.setStatus(DomainStatus.PENDING);
        domain.setCreatedAt(Instant.now());
        
        domainRepository.save(domain);
        
        // Schedule DNS and SSL check
        scheduleDomainSetupCheck(domainName);
    }
    
    private boolean verifyDomainOwnership(String domain, String token) {
        // Implementation to check DNS TXT record
        // ...
    }
    
    private void scheduleDomainSetupCheck(String domain) {
        // Schedule async job to verify DNS and SSL setup
        // ...
    }
}
```

**API for Custom URL Management**

```java
@RestController
@RequestMapping("/api/v1/custom-urls")
public class CustomUrlController {
    private final CustomUrlService customUrlService;
    private final CustomDomainService customDomainService;
    
    // Constructor injection...
    
    @PostMapping
    public ResponseEntity<CustomUrlResponse> createCustomUrl(
            @RequestBody @Valid CreateCustomUrlRequest request,
            @AuthenticationPrincipal UserDetails userDetails) {
        
        User user = ((CustomUserDetails) userDetails).getUser();
        String shortCode = customUrlService.createCustomUrl(
            request.getLongUrl(), 
            request.getCustomAlias(), 
            user.getId()
        );
        
        CustomUrlResponse response = new CustomUrlResponse();
        response.setShortCode(shortCode);
        response.setLongUrl(request.getLongUrl());
        response.setFullShortUrl(buildFullUrl(shortCode, request.getDomain()));
        
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
    
    @PostMapping("/domains")
    public ResponseEntity<DomainRegistrationResponse> registerCustomDomain(
            @RequestBody @Valid RegisterDomainRequest request,
            @AuthenticationPrincipal UserDetails userDetails) {
        
        User user = ((CustomUserDetails) userDetails).getUser();
        customDomainService.registerCustomDomain(request.getDomainName(), user.getId());
        
        DomainRegistrationResponse response = new DomainRegistrationResponse();
        response.setDomainName(request.getDomainName());
        response.setStatus(DomainStatus.PENDING.name());
        response.setSetupInstructions(generateSetupInstructions(request.getDomainName()));
        
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
    
    private String buildFullUrl(String shortCode, String domain) {
        if (domain != null) {
            return "https://" + domain + "/" + shortCode;
        }
        return "https://" + defaultDomain + "/" + shortCode;
    }
    
    private SetupInstructions generateSetupInstructions(String domain) {
        // Create DNS and verification instructions
        // ...
    }
}
```

**Performance & Scaling**
- Custom URLs are typically a small percentage of total URLs.
- Validation and reservation are more compute-intensive than regular shortening.
- Custom domains require additional infrastructure (DNS, SSL, routing).

**Pitfalls & Best Practices**
- **Name Squatting**: Implement policies to prevent abuse of branded terms.
- **Validation Complexity**: Balance security with usability in custom alias rules.
- **Domain Management**: Automated verification and certificate renewal are crucial.
- **Pricing & Limits**: Consider tiered pricing for different custom URL features.
- Have a clear dispute resolution process for trademark conflicts.
- Implement alias suggestion systems to help users find available options.
- Regular auditing of custom URLs for policy compliance.
- Consider the lifecycle of custom URLs (expiration, transfer, reclamation).

## Performance & Scalability Considerations

1. **Extremely Read-Heavy Workload**:
   - The vast majority of operations are redirects (often >95%).
   - System should be heavily optimized for read performance.
   - Write operations (creating URLs) are much less frequent.

2. **Multi-Tier Caching Strategy**:
   - Local in-memory caches at application level.
   - Distributed Redis cache for frequently accessed URLs.
   - CDN caching for extremely popular URLs.
   - Cache hit rates of >95% are achievable and desirable.

3. **Database Scaling**:
   - Horizontal sharding by short code prefix.
   - Read replicas for analytics and reporting queries.
   - Vertical scaling for primary write nodes.
   - Consider NoSQL for the mapping store at very large scale.

4. **Geographic Distribution**:
   - Deploy application instances in multiple regions.
   - Use global DNS routing (latency-based).
   - Regional database clusters with cross-region replication.
   - Edge computing for ultra-low latency redirects.

5. **Short URL ID Space Management**:
   - With 6-character codes using Base62, you have 62^6 = 56.8 billion combinations.
   - Random generation with collision checking works well initially.
   - As system grows, consider distributed ID generation (Snowflake, UUID, KGS).
   - Reserve a portion of ID space for custom/premium URLs.

6. **Analytics Data Volume**:
   - Click data grows much faster than URL mappings.
   - Implement time-based partitioning and archiving.
   - Consider data sampling for ultra-high-volume URLs.
   - Separate analytics storage from core mapping functionality.

7. **Latency Budget**:
   - Redirect operations should complete in <100ms (ideally <50ms).
   - Cache lookup should be <10ms.
   - Database lookup (cache miss) should be <50ms.
   - Analytics recording should not impact redirect performance.

8. **Traffic Patterns**:
   - Spiky traffic when popular links are shared.
   - Implement auto-scaling based on traffic patterns.
   - Rate limiting to prevent abuse or DoS attacks.
   - Circuit breakers to maintain core functionality during failures.

## Common Pitfalls & How to Avoid Them

1. **Cache Inconsistency**:
   - Problem: Stale data in cache after URL updates or deletions.
   - Solution: Implement cache invalidation on writes, TTL for cache entries.

2. **Database Bottlenecks**:
   - Problem: Single database becomes a bottleneck at scale.
   - Solution: Implement sharding, separate read/write concerns, use appropriate NoSQL solutions.

3. **URL Scanning/Enumeration**:
   - Problem: Malicious users scanning short codes to find sensitive URLs.
   - Solution: Use unpredictable codes, rate limiting by IP, avoid sequential allocation.

4. **Analytics Impact on Performance**:
   - Problem: Tracking slows down redirect operations.
   - Solution: Asynchronous event publishing, queuing systems, background processing.

5. **Custom URL Conflicts**:
   - Problem: Disputes over custom URL ownership, trademark issues.
   - Solution: Clear policies, reserved word lists, verification for branded terms.

6. **Reliability Under Load**:
   - Problem: System failures during traffic spikes.
   - Solution: Graceful degradation, circuit breakers, robust caching, auto-scaling.

7. **Malicious URL Distribution**:
   - Problem: Service used to distribute phishing/malware links.
   - Solution: URL scanning, reputation systems, user reporting, preview pages.

8. **Data Growth Management**:
   - Problem: Unbounded growth of URL and analytics data.
   - Solution: Archival strategies, time-based partitioning, data retention policies.

9. **Redirect Loop Detection**:
   - Problem: Shortening already shortened URLs creates loops.
   - Solution: Detect and prevent shortening of URLs from your own domain.

10. **Global Performance Variation**:
    - Problem: Users in different regions experience varying latency.
    - Solution: Multi-region deployment, CDN integration, global load balancing.

## Best Practices & Maintenance

1. **Monitoring & Alerting**:
   - Track key metrics: redirect latency, cache hit ratio, error rates.
   - Set alerts for unusual patterns: traffic spikes, increased error rates.
   - Monitor database and cache performance closely.
   - Use distributed tracing for end-to-end visibility.

2. **Regular Data Maintenance**:
   - Archive old/inactive URLs to cold storage.
   - Implement data retention policies for analytics.
   - Cleanup expired or deleted URLs.
   - Optimize database indices periodically.

3. **Security & Compliance**:
   - Regular security audits and penetration testing.
   - Keep software dependencies updated.
   - Comply with data protection regulations (GDPR, CCPA).
   - Implement proper access controls and authentication.

4. **API Management**:
   - Versioned APIs with clear documentation.
   - Consistent error responses and status codes.
   - Rate limiting and throttling for fair usage.
   - Authentication and authorization controls.

5. **Disaster Recovery**:
   - Regular backups of URL mappings.
   - Cross-region replication for high availability.
   - Documented recovery procedures and regular drills.
   - Fallback strategies for component failures.

6. **Performance Optimization**:
   - Regular load testing to identify bottlenecks.
   - Performance regression testing with each release.
   - Cache tuning based on access patterns.
   - Query optimization for database operations.

7. **Operational Excellence**:
   - Automated deployments and infrastructure as code.
   - Canary releases for new features.
   - Blue-green deployments for zero downtime.
   - Comprehensive logging and error tracking.

8. **Documentation & Knowledge Sharing**:
   - Well-documented architecture and code.
   - Runbooks for common operational tasks.
   - Post-mortems for incidents with learnings.
   - Cross-training of team members.

## How to Discuss in a Principal Engineer Interview

1. **Clarify Requirements First**:
   - Expected scale (URLs, redirects per second).
   - Functional requirements beyond basic shortening.
   - SLAs for availability and latency.
   - Geographic distribution needs.

2. **Present Overall Architecture**:
   - Outline microservices and their responsibilities.
   - Explain data flow for key operations.
   - Discuss technology choices with rationale.
   - Highlight scalability and reliability features.

3. **Deep Dive on Key Components**:
   - Short code generation strategy with pros/cons.
   - Storage architecture and partitioning approach.
   - Caching strategy and consistency considerations.
   - Analytics collection and processing pipeline.

4. **Discuss Trade-offs**:
   - Performance vs. functionality (e.g., analytics impact).
   - Consistency vs. availability (e.g., caching policies).
   - Simplicity vs. scale (e.g., when to introduce complexity).
   - Security vs. usability (e.g., preview pages, rate limiting).

5. **Address Scaling Challenges**:
   - How the system evolves from prototype to massive scale.
   - Incremental improvements and migration strategies.
   - Identifying and addressing bottlenecks.
   - Cost optimization at scale.

6. **Cover Operational Aspects**:
   - Monitoring and alerting approach.
   - Deployment and maintenance strategies.
   - Disaster recovery and fault tolerance.
   - Security measures and compliance considerations.

7. **Demonstrate Technical Depth**:
   - Code examples for critical components.
   - Database schema design and query patterns.
   - Caching strategies and invalidation approaches.
   - Performance optimization techniques.

8. **Show Business Understanding**:
   - How the system supports business objectives.
   - Feature prioritization based on user/business needs.
   - Cost considerations and efficiency measures.
   - Potential monetization strategies for the service.

## Conclusion

This URL shortener service architecture provides a comprehensive approach to building a scalable, reliable, and feature-rich system. By leveraging a microservices architecture, distributed caching, and event-driven design, the system can handle billions of URLs and millions of redirects per second while maintaining extremely low latency.

The architecture emphasizes:
- **Performance optimization** for the core redirection flow, which represents the vast majority of system traffic.
- **Scalability** through database sharding, caching layers, and stateless services.
- **Reliability** via redundancy, failover mechanisms, and graceful degradation.
- **Security measures** to prevent abuse and protect users from malicious content.
- **Analytics capabilities** that provide valuable insights without impacting core functionality.

The design provides a balance between simplicity and sophistication, allowing for incremental adoption of advanced features as the service grows. From custom short URLs and branded domains to advanced analytics and security features, the architecture supports a complete feature set while maintaining the core promise of URL shortening: turning long, unwieldy URLs into short, manageable links that redirect quickly and reliably.

By following the principles and best practices outlined in this document, organizations can build a URL shortening service that scales effectively, performs reliably, and delivers value to both casual users and enterprise customers alike.
