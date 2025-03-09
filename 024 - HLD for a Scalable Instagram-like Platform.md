# High-Level Design for a Scalable Instagram-like Platform

## Table of Contents
- [Overview & Core Requirements](#overview--core-requirements)
- [Major Microservices & Responsibilities](#major-microservices--responsibilities)
  - [User Service](#user-service)
  - [Post Service](#post-service)
  - [Media Service](#media-service)
  - [Feed Service](#feed-service)
  - [Story Service](#story-service)
  - [Graph Service](#graph-service)
  - [Engagement Service](#engagement-service)
  - [Notification Service](#notification-service)
  - [Search Service](#search-service)
  - [Discovery & Explore Service](#discovery--explore-service)
  - [Content Moderation Service](#content-moderation-service)
  - [Analytics Service](#analytics-service)
- [High-Level Architecture & Data Flow](#high-level-architecture--data-flow)
- [Key System Design Topics & Deep Dives](#key-system-design-topics--deep-dives)
  - [Media Storage & Delivery](#media-storage--delivery)
  - [Feed Generation & Delivery](#feed-generation--delivery)
  - [Story Architecture](#story-architecture)
  - [Graph Database & Follow Relationships](#graph-database--follow-relationships)
  - [Caching Strategies](#caching-strategies)
  - [Database Design & Storage](#database-design--storage)
  - [Notification System](#notification-system)
  - [Search & Discovery](#search--discovery)
  - [Content Moderation](#content-moderation)
  - [Security Fundamentals](#security-fundamentals)
- [Performance & Scalability Considerations](#performance--scalability-considerations)
- [Common Pitfalls & How to Avoid Them](#common-pitfalls--how-to-avoid-them)
- [Best Practices & Maintenance](#best-practices--maintenance)
- [How to Discuss in a Principal Engineer Interview](#how-to-discuss-in-a-principal-engineer-interview)
- [Conclusion](#conclusion)

## Overview & Core Requirements

We want to build an **Instagram-like platform** where users can:

1. **Upload and view photos/videos** with various filters and edits.
2. **Follow other users** to create a social graph.
3. **View feeds** containing posts from users they follow.
4. Create and view **Stories** that automatically expire after 24 hours.
5. **Engage with content** through likes and comments.
6. Receive **notifications** for interactions (likes, comments, follows).
7. **Discover** new content and users through search and explore features.
8. Access the platform from **multiple devices** with consistent experience.

### Additional Requirements

- **High availability** (99.9%) and **low latency** (<1000ms for image loading).
- **Adaptive content delivery** based on network conditions (lower quality images for poor connections).
- **Strong security** around user data and authentication.
- **Scalable architecture** to handle millions of users and billions of photos.
- **Real-time interactions** for immediate feedback on engagements.
- **Content moderation** to filter inappropriate content.
- **Analytics** to track user engagement and platform health.
- **Global reach** with multi-region deployment.

## Major Microservices & Responsibilities

### User Service
- **Responsibilities**: Manages user accounts, profiles, authentication/authorization.
- **Data Store**: Relational DB (PostgreSQL) for user profiles, authentication data.
- **Notes**: 
  - Handles registration, login, profile updates.
  - Stores user metadata (username, email, profile picture URL, bio).
  - Implements OAuth or JWT-based authentication.

### Post Service
- **Responsibilities**: Manages post metadata, creation, and retrieval.
- **Data Store**:
  - NoSQL database (Cassandra) for post metadata.
  - References to actual media in object storage.
- **Notes**:
  - Stores post captions, timestamps, location data.
  - Maintains post ownership and visibility settings.
  - Publishes "PostCreated" events to Kafka for downstream processing.

### Media Service
- **Responsibilities**: Handles upload, processing, storage, and delivery of photos and videos.
- **Data Store**:
  - Object storage (S3/GCS) for raw media files.
  - CDN for optimized delivery.
- **Notes**:
  - Processes uploads (validation, virus scanning).
  - Generates multiple resolutions and formats of each media.
  - Optimizes delivery based on device and network conditions.
  - Implements video transcoding for different formats and bitrates.

### Feed Service
- **Responsibilities**: Generates and delivers personalized feeds of posts.
- **Data Store**:
  - Redis for caching active user feeds.
  - Cassandra for persistent storage of feed items.
- **Notes**:
  - Implements hybrid fan-out model for feed generation.
  - Consumes "PostCreated" events to update feeds.
  - Supports pagination and feed position memory.
  - Handles ranking algorithms (chronological or engagement-based).

### Story Service
- **Responsibilities**: Manages ephemeral content that expires after 24 hours.
- **Data Store**:
  - Redis for active stories.
  - Object storage (with TTL) for story media.
- **Notes**:
  - Implements strict TTL mechanisms.
  - Tracks story views and interactions.
  - Handles story highlights (persistent collections of past stories).
  - Creates story rings/UI indicators for users with active stories.

### Graph Service
- **Responsibilities**: Manages social connections (follows/followers).
- **Data Store**:
  - Graph database (Neo4j) or specialized sharded relational DB.
  - Redis for caching active user relationships.
- **Notes**:
  - Optimized for "who follows whom" queries.
  - Handles follow/unfollow events.
  - Supports privacy settings (public/private accounts).
  - Publishes graph change events to Kafka for feed updates.

### Engagement Service
- **Responsibilities**: Manages likes, comments, and other interactions.
- **Data Store**:
  - NoSQL database (Cassandra) for scalable engagement storage.
  - Redis for caching hot post engagement counts.
- **Notes**:
  - Handles high write throughput for viral content.
  - Maintains comment threads and replies.
  - Publishes engagement events to Kafka for notifications.
  - Provides engagement metrics to analytics.

### Notification Service
- **Responsibilities**: Manages and delivers in-app and push notifications.
- **Data Flow**:
  - Consumes events from Kafka (likes, comments, follows).
  - Pushes to device-specific notification services (APNS, FCM).
- **Notes**:
  - Supports notification preferences and aggregation.
  - Handles delivery status tracking.
  - Implements backpressure mechanisms for notification storms.
  - Maintains notification history.

### Search Service
- **Responsibilities**: Provides search functionality for users, posts, hashtags.
- **Data Store**: Elasticsearch for full-text search and complex queries.
- **Notes**:
  - Consumes user, post, and hashtag events from Kafka.
  - Supports typeahead/autocomplete.
  - Implements personalized search ranking.
  - Handles multilingual search capabilities.

### Discovery & Explore Service
- **Responsibilities**: Generates personalized content discovery experiences.
- **Data Store**:
  - Redis for caching trending content.
  - Integration with recommendation models.
- **Notes**:
  - Analyzes user behavior for content recommendations.
  - Identifies trending hashtags and content.
  - Balances freshness, relevance, and diversity.
  - Segments content by categories and interests.

### Content Moderation Service
- **Responsibilities**: Filters inappropriate content through automated and manual review.
- **Data Flow**:
  - Consumes new content events from Kafka.
  - Integrates with ML services for automated filtering.
- **Notes**:
  - Implements text analysis for harmful content.
  - Uses image/video recognition for inappropriate visual content.
  - Manages user reporting and review queues.
  - Enforces content policies and regulations.

### Analytics Service
- **Responsibilities**: Collects and processes platform usage data.
- **Data Store**:
  - Kafka for event streaming.
  - Data warehouse (Snowflake/BigQuery) for aggregated metrics.
  - HDFS/S3 data lake for raw event storage.
- **Notes**:
  - Tracks user engagement metrics.
  - Provides business insights and reporting.
  - Supports A/B testing frameworks.
  - Generates inputs for recommendation systems.

## High-Level Architecture & Data Flow

1. **User Registration & Authentication Flow**
   - User registers through the **User Service**.
   - Service stores user data in PostgreSQL.
   - Authentication tokens (JWT) are generated for session management.
   - Profile creation events trigger related services (Graph, Search indexing).

2. **Post Creation Flow**
   - User uploads media through the **Media Service**.
   - Media is processed, validated, and stored in object storage.
   - Multiple resolutions are generated and cached in CDN.
   - **Post Service** creates metadata entry linking to media.
   - "PostCreated" event is published to Kafka.
   - **Feed Service** consumes the event and updates followers' feeds.
   - **Search Service** indexes the post content.
   - **Content Moderation** scans for policy violations.

3. **Story Creation Flow**
   - Similar to post flow, but with **Story Service** handling metadata.
   - TTL is set for automatic expiration after 24 hours.
   - Story visibility is determined by privacy settings and "close friends" lists.
   - Story view events are tracked for creator analytics.

4. **Feed Generation Flow**
   - **Feed Service** maintains pre-computed feeds for active users.
   - On feed request, service checks Redis cache first.
   - For cache misses, feeds are rebuilt from Cassandra.
   - Hybrid approach: Push model for regular users, pull model for high-follower accounts.
   - Feed pagination and position tracking for seamless scrolling.

5. **Engagement Flow**
   - User likes or comments via the **Engagement Service**.
   - Counters are updated in Redis and asynchronously persisted to Cassandra.
   - Events are published to Kafka for notifications and analytics.
   - Comment threads are retrieved with pagination for scalability.

6. **Discovery Flow**
   - **Discovery Service** generates personalized content based on user behavior.
   - Trending hashtags and posts are identified through real-time analytics.
   - Content is filtered based on user preferences and past interactions.
   - Results are cached and refreshed periodically to balance freshness and performance.

7. **Notification Flow**
   - **Notification Service** consumes events from Kafka.
   - Notifications are delivered based on user preferences.
   - Real-time notifications use WebSockets for active users.
   - Push notifications are sent via APNS/FCM for mobile devices.

## Key System Design Topics & Deep Dives

### Media Storage & Delivery

**Concept & Principles**
- Efficient storage and global delivery of photos and videos.
- Multiple resolution generation for different devices and network conditions.
- CDN utilization for low-latency global access.
- Adaptive bitrate streaming for videos.

**Real-World Usage**
- User uploads original photo (4K resolution).
- System generates multiple variants (thumbnail, medium, high-res).
- CDN edge locations cache popular content.
- Client requests appropriate resolution based on device and network.

**Implementation Example (Media Processing Pipeline)**

```java
@Service
public class MediaProcessingService {
    
    @Autowired
    private S3Client s3Client;
    
    @Autowired
    private ImageProcessor imageProcessor;
    
    @Autowired
    private KafkaTemplate<String, MediaProcessedEvent> kafkaTemplate;
    
    @Value("${media.bucket.name}")
    private String mediaBucket;
    
    @Value("${kafka.topic.media-processed}")
    private String mediaTopic;
    
    public MediaMetadata processUploadedImage(MultipartFile file, Long userId) throws IOException {
        // Generate unique ID for the media
        String mediaId = UUID.randomUUID().toString();
        
        // Read original image
        BufferedImage originalImage = ImageIO.read(file.getInputStream());
        
        // Process different resolutions
        Map<String, byte[]> variants = new HashMap<>();
        variants.put("original", file.getBytes());
        variants.put("high", imageProcessor.resize(originalImage, 1080));
        variants.put("medium", imageProcessor.resize(originalImage, 540));
        variants.put("thumbnail", imageProcessor.resize(originalImage, 150));
        
        // Upload each variant to S3
        Map<String, String> urls = new HashMap<>();
        for (Map.Entry<String, byte[]> variant : variants.entrySet()) {
            String key = String.format("users/%d/media/%s_%s.jpg", 
                    userId, mediaId, variant.getKey());
            
            PutObjectRequest request = PutObjectRequest.builder()
                    .bucket(mediaBucket)
                    .key(key)
                    .build();
            
            s3Client.putObject(request, 
                    RequestBody.fromBytes(variant.getValue()));
            
            urls.put(variant.getKey(), 
                    String.format("https://cdn.example.com/%s", key));
        }
        
        // Create metadata
        MediaMetadata metadata = new MediaMetadata(
                mediaId, 
                userId,
                urls,
                originalImage.getWidth(),
                originalImage.getHeight(),
                file.getSize(),
                new Date()
        );
        
        // Publish event
        kafkaTemplate.send(mediaTopic, mediaId, 
                new MediaProcessedEvent(metadata));
        
        return metadata;
    }
}
```

**Performance & Scaling**
- Content-aware image compression to reduce size without quality loss.
- Lazy generation of variants (generate on first request).
- Regional caching to reduce latency based on user location.
- Batch processing for high-throughput periods.

**Pitfalls & Best Practices**
- **Storage Costs**: Balance quality vs. storage with smart compression.
- **Cold Storage Migration**: Move old media to cheaper storage tiers.
- **CDN Cache Invalidation**: Use versioned URLs to avoid stale content.
- **Media Security**: Signed URLs for private content access.
- Implement pre-signed URLs for direct upload to object storage.
- Use image processing libraries optimized for server environments.
- Implement progressive loading for better perceived performance.

### Feed Generation & Delivery

**Concept & Principles**
- Fan-out models: push (write time) vs. pull (read time).
- Hybrid approaches for different user types.
- Feed caching and pagination strategies.
- Ranking algorithms for content relevance.

**Real-World Usage**
- When user A posts, the system updates feeds for all of A's followers.
- For users with millions of followers, a different strategy is used.
- Feeds are cached and paginated for efficient delivery.
- Feed ranking may consider recency, engagement, and relationship strength.

**Implementation Example (Hybrid Feed Generation)**

```java
@Service
public class FeedService {

    @Autowired
    private GraphService graphService;
    
    @Autowired
    private PostRepository postRepository;
    
    @Autowired
    private FeedRepository feedRepository;
    
    @Autowired
    private RedisTemplate<String, List<String>> redisTemplate;
    
    private static final int CELEBRITY_THRESHOLD = 10000;
    private static final int FEED_CACHE_TTL = 3600; // 1 hour
    private static final int MAX_FEED_ITEMS = 1000;
    
    @KafkaListener(topics = "post-created")
    public void processNewPost(PostEvent event) {
        // Get author's followers
        List<Long> followers = graphService.getFollowers(event.getUserId());
        
        // Check if this is a "celebrity" post
        if (followers.size() > CELEBRITY_THRESHOLD) {
            // For celebrities, mark post for pull-based delivery
            markPostAsCelebrity(event.getPostId());
            return;
        }
        
        // For regular users, push to all followers' feeds
        for (Long followerId : followers) {
            // Skip inactive users
            if (!isActiveUser(followerId)) continue;
            
            // Add to persistent storage
            feedRepository.addPostToFeed(followerId, event.getPostId());
            
            // Update cache if it exists
            String feedKey = "feed:" + followerId;
            if (redisTemplate.hasKey(feedKey)) {
                List<String> feed = redisTemplate.opsForValue().get(feedKey);
                feed.add(0, event.getPostId().toString());
                
                // Trim to prevent unbounded growth
                if (feed.size() > MAX_FEED_ITEMS) {
                    feed = feed.subList(0, MAX_FEED_ITEMS);
                }
                
                redisTemplate.opsForValue().set(feedKey, feed, 
                        FEED_CACHE_TTL, TimeUnit.SECONDS);
            }
        }
    }
    
    public List<Post> getUserFeed(Long userId, int page, int pageSize) {
        // Try cache first
        String feedKey = "feed:" + userId;
        List<String> postIds = redisTemplate.opsForValue().get(feedKey);
        
        if (postIds == null) {
            // Cache miss - rebuild from database
            postIds = feedRepository.getFeedPostIds(userId);
            
            // Merge with celebrity posts
            List<Long> followedCelebrities = 
                    graphService.getFollowedCelebrities(userId);
            
            if (!followedCelebrities.isEmpty()) {
                List<String> celebrityPostIds = 
                        postRepository.getRecentPostIdsByUsers(followedCelebrities);
                postIds.addAll(celebrityPostIds);
                
                // Sort by timestamp (newest first)
                postIds.sort((id1, id2) -> {
                    Post p1 = postRepository.findById(id1).orElse(null);
                    Post p2 = postRepository.findById(id2).orElse(null);
                    if (p1 == null || p2 == null) return 0;
                    return p2.getCreatedAt().compareTo(p1.getCreatedAt());
                });
            }
            
            // Cache the result
            redisTemplate.opsForValue().set(feedKey, postIds, 
                    FEED_CACHE_TTL, TimeUnit.SECONDS);
        }
        
        // Apply pagination
        int start = page * pageSize;
        int end = Math.min(start + pageSize, postIds.size());
        
        if (start >= postIds.size()) {
            return Collections.emptyList();
        }
        
        List<String> pageIds = postIds.subList(start, end);
        
        // Fetch full post data
        return postRepository.findAllByIds(pageIds);
    }
}
```

**Performance & Scaling**
- Pre-compute feeds for active users to reduce latency.
- Implement pagination to limit memory and network usage.
- Use cursor-based pagination for large feeds.
- Separate read and write paths for independent scaling.

**Pitfalls & Best Practices**
- **Feed Consistency**: Users seeing different content on different devices.
- **Empty Feed Problem**: New users with limited connections see empty feeds.
- **Feed Regeneration Cost**: Cache misses causing expensive rebuilds.
- Implement intelligent caching based on user activity patterns.
- Use background jobs to pre-warm caches for active users.
- Consider time-windowing to limit feed depth for very active users.

### Story Architecture

**Concept & Principles**
- Ephemeral content with 24-hour lifecycle.
- Efficient storage and delivery of time-limited media.
- View tracking and privacy controls.
- Automatic cleanup mechanisms.

**Real-World Usage**
- User creates a story that is visible for 24 hours.
- Stories appear in a carousel at the top of followers' feeds.
- Views and interactions are tracked for creator visibility.
- After 24 hours, stories are automatically removed from feeds.

**Implementation Example (Story Service)**

```java
@Service
public class StoryService {

    @Autowired
    private StoryRepository storyRepository;
    
    @Autowired
    private MediaService mediaService;
    
    @Autowired
    private RedisTemplate<String, List<StoryMetadata>> storyRedis;
    
    @Value("${story.ttl.seconds}")
    private int storyTtlSeconds = 86400; // 24 hours
    
    public Story createStory(StoryCreateRequest request, Long userId) {
        // Process media
        MediaMetadata media = mediaService.processStoryMedia(
                request.getMedia(), userId);
        
        // Create story entity
        Story story = new Story();
        story.setUserId(userId);
        story.setMediaId(media.getId());
        story.setMediaUrls(media.getUrls());
        story.setCreatedAt(new Date());
        story.setExpiresAt(new Date(System.currentTimeMillis() + 
                storyTtlSeconds * 1000L));
        
        // Set privacy
        story.setVisibleToAll(!request.isClosefriendsOnly());
        if (request.isClosefriendsOnly()) {
            story.setVisibleTo(userService.getCloseFriends(userId));
        }
        
        // Save to persistent storage
        Story savedStory = storyRepository.save(story);
        
        // Update active stories in Redis
        updateUserActiveStories(userId, savedStory);
        
        // Schedule deletion job
        scheduleStoryDeletion(savedStory.getId(), storyTtlSeconds);
        
        return savedStory;
    }
    
    private void updateUserActiveStories(Long userId, Story newStory) {
        String key = "user:stories:" + userId;
        
        List<StoryMetadata> stories = storyRedis.opsForValue().get(key);
        if (stories == null) {
            stories = new ArrayList<>();
        }
        
        stories.add(new StoryMetadata(
                newStory.getId(),
                newStory.getMediaUrls().get("thumbnail"),
                newStory.getCreatedAt()
        ));
        
        // Set with TTL
        storyRedis.opsForValue().set(key, stories, storyTtlSeconds, TimeUnit.SECONDS);
    }
    
    public List<StoryRing> getStoriesForFeed(Long viewerId) {
        // Get users the viewer follows who have active stories
        List<Long> following = graphService.getFollowing(viewerId);
        List<StoryRing> storyRings = new ArrayList<>();
        
        for (Long userId : following) {
            String key = "user:stories:" + userId;
            List<StoryMetadata> stories = storyRedis.opsForValue().get(key);
            
            if (stories != null && !stories.isEmpty()) {
                // Filter by privacy
                stories = filterStoriesByPrivacy(stories, userId, viewerId);
                
                if (!stories.isEmpty()) {
                    UserBasic user = userService.getBasicInfo(userId);
                    storyRings.add(new StoryRing(user, stories));
                }
            }
        }
        
        // Sort by recency, close friends, etc.
        sortStoryRings(storyRings, viewerId);
        
        return storyRings;
    }
    
    @Scheduled(fixedRate = 3600000) // Every hour
    public void cleanupExpiredStories() {
        Date now = new Date();
        List<Story> expiredStories = storyRepository.findByExpiresAtLessThan(now);
        
        for (Story story : expiredStories) {
            // Remove from active stories in Redis
            removeFromActiveStories(story.getUserId(), story.getId());
            
            // Mark as expired in database
            story.setExpired(true);
            storyRepository.save(story);
            
            // Optionally move media to cold storage or delete
            if (!story.isSavedToHighlights()) {
                mediaService.markForColdStorage(story.getMediaId());
            }
        }
    }
}
```

**Performance & Scaling**
- In-memory storage for active stories reduces database load.
- Scheduled cleanup jobs for expired content.
- Batch processing for view counting and analytics.
- Regional caching for lower latency.

**Pitfalls & Best Practices**
- **Clock Synchronization**: Ensure consistent expiration times across services.
- **Storage Reclamation**: Efficient cleanup of expired content.
- **View Tracking Overhead**: High write load for popular stories.
- Use TTL-based storage systems (Redis) for automatic expiration.
- Implement background processes for cleaning up expired media.
- Track views in batches to reduce database write pressure.

### Graph Database & Follow Relationships

**Concept & Principles**
- Efficient storage and traversal of social connections.
- Bidirectional relationships (follower/following).
- Privacy controls and relationship metadata.
- Fast path finding for recommendations.

**Real-World Usage**
- User follows/unfollows are high-frequency operations.
- Feed generation requires efficient follower lookups.
- Recommendations leverage graph relationships.
- Privacy settings affect traversal rules.

**Implementation Example (Graph Service with Neo4j)**

```java
@Service
public class GraphService {

    @Autowired
    private Neo4jTemplate neo4jTemplate;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private static final int CACHE_TTL = 3600; // 1 hour
    
    public void createFollowRelationship(Long followerId, Long followeeId) {
        // Check if followee allows this follower
        if (!canUserFollow(followerId, followeeId)) {
            throw new PrivacyException("This account is private");
        }
        
        // Create relationship in Neo4j
        String cypher = "MATCH (follower:User {id: $followerId}), "
                + "(followee:User {id: $followeeId}) "
                + "CREATE (follower)-[r:FOLLOWS {createdAt: datetime()}]->(followee) "
                + "RETURN r";
        
        Map<String, Object> params = new HashMap<>();
        params.put("followerId", followerId);
        params.put("followeeId", followeeId);
        
        neo4jTemplate.query(cypher, params);
        
        // Invalidate caches
        invalidateFollowerCache(followeeId);
        invalidateFollowingCache(followerId);
        
        // Publish event
        publishFollowEvent(followerId, followeeId);
    }
    
    public List<Long> getFollowers(Long userId) {
        // Try cache first
        String cacheKey = "followers:" + userId;
        List<Long> followers = (List<Long>) redisTemplate.opsForValue().get(cacheKey);
        
        if (followers == null) {
            // Cache miss - query Neo4j
            String cypher = "MATCH (follower:User)-[:FOLLOWS]->(user:User {id: $userId}) "
                    + "RETURN follower.id AS id";
            
            Map<String, Object> params = new HashMap<>();
            params.put("userId", userId);
            
            followers = neo4jTemplate.query(cypher, params)
                    .map(result -> (Long) result.get("id"))
                    .collect(Collectors.toList());
            
            // Cache the result
            redisTemplate.opsForValue().set(cacheKey, followers, CACHE_TTL, TimeUnit.SECONDS);
        }
        
        return followers;
    }
    
    public List<Long> getFollowing(Long userId) {
        // Similar to getFollowers but with reversed direction
        // Implementation omitted for brevity
    }
    
    public boolean isFollowing(Long followerId, Long followeeId) {
        // Check cache or query directly for small result
        String cacheKey = "following:" + followerId;
        List<Long> following = (List<Long>) redisTemplate.opsForValue().get(cacheKey);
        
        if (following != null) {
            return following.contains(followeeId);
        }
        
        // Direct query
        String cypher = "MATCH (follower:User {id: $followerId})"
                + "-[:FOLLOWS]->(followee:User {id: $followeeId}) "
                + "RETURN count(*) > 0 AS follows";
        
        Map<String, Object> params = new HashMap<>();
        params.put("followerId", followerId);
        params.put("followeeId", followeeId);
        
        return neo4jTemplate.queryForObject(cypher, params, Boolean.class);
    }
    
    public List<UserSuggestion> suggestUsers(Long userId) {
        // Find friends-of-friends and users with similar interests
        String cypher = "MATCH (user:User {id: $userId})-[:FOLLOWS]->(friend)"
                + "-[:FOLLOWS]->(suggestion:User) "
                + "WHERE NOT (user)-[:FOLLOWS]->(suggestion) "
                + "AND suggestion.id <> $userId "
                + "RETURN suggestion.id AS id, "
                + "count(distinct friend) AS commonConnections "
                + "ORDER BY commonConnections DESC LIMIT 10";
        
        Map<String, Object> params = new HashMap<>();
        params.put("userId", userId);
        
        List<UserSuggestion> suggestions = neo4jTemplate.query(cypher, params)
                .map(result -> new UserSuggestion(
                        (Long) result.get("id"),
                        ((Long) result.get("commonConnections")).intValue()))
                .collect(Collectors.toList());
        
        return suggestions;
    }
}
```

**Performance & Scaling**
- Caching popular follower/following lists.
- Sharding the graph by user clusters.
- Batch processing for graph analytics.
- Read replicas for query-heavy workloads.

**Pitfalls & Best Practices**
- **Hotspots**: Celebrity users with millions of followers.
- **Cache Invalidation**: Keeping graph caches consistent.
- **Traversal Depth**: Deep traversals can be expensive.
- Limit traversal depth for recommendation queries.
- Use materialized paths for frequently accessed relationships.
- Implement efficient batch operations for bulk relationship changes.

### Caching Strategies

**Concept & Principles**
- Multi-level caching for different data types.
- Write-through vs. lazy loading approaches.
- TTL and eviction policies.
- Cache consistency and invalidation strategies.

**Real-World Usage**
- User profile data cached for active sessions.
- Feed data cached for active users.
- Post metadata and engagement counters cached for trending content.
- Follow relationships cached for feed generation.

**Implementation Example (Layered Caching)**

```java
@Configuration
public class CacheConfig {

    @Bean
    public CacheManager redisCacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1))
                .serializeKeysWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new JdkSerializationRedisSerializer()));
        
        Map<String, RedisCacheConfiguration> configs = new HashMap<>();
        // Short TTL for volatile data
        configs.put("posts", config.entryTtl(Duration.ofMinutes(30)));
        configs.put("feeds", config.entryTtl(Duration.ofMinutes(15)));
        // Longer TTL for stable data
        configs.put("users", config.entryTtl(Duration.ofHours(6)));
        configs.put("media", config.entryTtl(Duration.ofDays(1)));
        
        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(config)
                .withInitialCacheConfigurations(configs)
                .build();
    }
}

@Service
public class PostCachingService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private PostRepository postRepository;
    
    // Example of a write-through cache
    public void cachePost(Post post) {
        String postKey = "post:" + post.getId();
        redisTemplate.opsForValue().set(postKey, post, 1, TimeUnit.HOURS);
        
        // Cache counters separately for high-frequency updates
        String likesKey = "post:likes:" + post.getId();
        redisTemplate.opsForValue().set(likesKey, post.getLikeCount(), 1, TimeUnit.HOURS);
        
        String commentsKey = "post:comments:" + post.getId();
        redisTemplate.opsForValue().set(commentsKey, post.getCommentCount(), 1, TimeUnit.HOURS);
    }
    
    // Example of cache-aside pattern
    public Post getPost(String postId) {
        String postKey = "post:" + postId;
        
        // Try cache first
        Post post = (Post) redisTemplate.opsForValue().get(postKey);
        
        if (post == null) {
            // Cache miss - get from database
            post = postRepository.findById(postId).orElse(null);
            
            if (post != null) {
                // Populate cache
                cachePost(post);
            }
        } else {
            // Update volatile counters from separate cache entries
            String likesKey = "post:likes:" + postId;
            Integer likes = (Integer) redisTemplate.opsForValue().get(likesKey);
            if (likes != null) {
                post.setLikeCount(likes);
            }
            
            String commentsKey = "post:comments:" + postId;
            Integer comments = (Integer) redisTemplate.opsForValue().get(commentsKey);
            if (comments != null) {
                post.setCommentCount(comments);
            }
        }
        
        return post;
    }
    
    // Update only counter cache for high-frequency operations
    public void incrementLikeCount(String postId) {
        String likesKey = "post:likes:" + postId;
        redisTemplate.opsForValue().increment(likesKey);
        
        // Asynchronously update database
        asyncTaskExecutor.execute(() -> {
            postRepository.incrementLikeCount(postId);
        });
    }
    
    // Batch invalidation for related caches
    public void invalidateUserPosts(Long userId) {
        // Find all post IDs for this user
        List<String> postIds = postRepository.findIdsByUserId(userId);
        
        // Batch delete from cache
        List<String> keys = new ArrayList<>();
        for (String postId : postIds) {
            keys.add("post:" + postId);
            keys.add("post:likes:" + postId);
            keys.add("post:comments:" + postId);
        }
        
        if (!keys.isEmpty()) {
            redisTemplate.delete(keys);
        }
    }
}
```

**Performance & Scaling**
- Use Redis Cluster for distributed caching.
- Local in-memory caches for ultra-low latency data.
- Hierarchical caching (L1/L2) for different access patterns.
- Intelligent prefetching for predictable access patterns.

**Pitfalls & Best Practices**
- **Cache Stampede**: Concurrent cache rebuilds for popular items.
- **Cache Coherence**: Inconsistent data across cache layers.
- **Memory Pressure**: Over-caching causing evictions.
- Implement cache warming for predictable traffic patterns.
- Use probabilistic early expiration to prevent mass expirations.
- Monitor cache hit rates and adjust TTLs accordingly.
- Implement request coalescing for concurrent cache misses.

### Database Design & Storage

**Concept & Principles**
- Polyglot persistence for different data types.
- Sharding strategies for horizontal scaling.
- Data partitioning and replication.
- Access pattern optimization.

**Real-World Usage**
- User data in relational DB (PostgreSQL) for ACID properties.
- Post metadata in NoSQL (Cassandra) for write scalability.
- Graph data in specialized DB (Neo4j) for relationship queries.
- Media metadata in document store (MongoDB) for flexible schema.

**Database Schema Examples**

```sql
-- PostgreSQL schema for User Service
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(30) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(100) NOT NULL,
    bio TEXT,
    profile_image_url VARCHAR(255),
    is_private BOOLEAN DEFAULT false,
    is_verified BOOLEAN DEFAULT false,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE user_device_tokens (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    device_token VARCHAR(255) NOT NULL,
    device_type VARCHAR(20) NOT NULL, -- 'ios', 'android', 'web'
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_used_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(user_id, device_token)
);
```

**NoSQL Schema (Cassandra CQL for Posts)**

```sql
-- Posts table
CREATE TABLE posts (
    id UUID PRIMARY KEY,
    user_id BIGINT,
    caption TEXT,
    location TEXT,
    media_refs LIST<TEXT>,
    created_at TIMESTAMP,
    is_archived BOOLEAN DEFAULT false,
    like_count COUNTER,
    comment_count COUNTER
);

-- User timeline (posts by user)
CREATE TABLE user_posts (
    user_id BIGINT,
    post_id UUID,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, created_at, post_id)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- Comments table
CREATE TABLE post_comments (
    post_id UUID,
    comment_id UUID,
    user_id BIGINT,
    content TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (post_id, created_at, comment_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

**Document Schema (MongoDB for Media Metadata)**

```javascript
// Media collection
{
  "_id": ObjectId("5f8a3d2e1c9d440000b7d5a7"),
  "mediaId": "550e8400-e29b-41d4-a716-446655440000",
  "userId": 12345,
  "mediaType": "image",
  "variants": {
    "original": {
      "url": "s3://media-bucket/original/550e8400-e29b-41d4-a716-446655440000.jpg",
      "width": 4000,
      "height": 3000,
      "size": 2458732
    },
    "high": {
      "url": "s3://media-bucket/high/550e8400-e29b-41d4-a716-446655440000.jpg",
      "width": 1080,
      "height": 810,
      "size": 245873
    },
    "medium": {
      "url": "s3://media-bucket/medium/550e8400-e29b-41d4-a716-446655440000.jpg",
      "width": 540,
      "height": 405,
      "size": 98349
    },
    "thumbnail": {
      "url": "s3://media-bucket/thumbnail/550e8400-e29b-41d4-a716-446655440000.jpg",
      "width": 150,
      "height": 112.5,
      "size": 18734
    }
  },
  "contentType": "image/jpeg",
  "filter": "clarendon",
  "metadata": {
    "location": {
      "latitude": 40.7128,
      "longitude": -74.0060,
      "name": "New York, NY"
    },
    "takenAt": ISODate("2023-04-15T14:30:00Z"),
    "camera": "iPhone 14 Pro",
    "exif": {
      // EXIF data
    }
  },
  "status": "active",
  "createdAt": ISODate("2023-04-15T15:45:00Z"),
  "updatedAt": ISODate("2023-04-15T15:45:00Z")
}
```

**Performance & Scaling**
- Horizontal sharding by user ID distributes load.
- Denormalization optimizes for read patterns.
- Time-based partitioning for historical data.
- Read replicas for query scaling.

**Pitfalls & Best Practices**
- **Hotspots**: Careful shard key selection to avoid overloaded partitions.
- **Join Complexity**: Denormalization to avoid cross-database joins.
- **Schema Evolution**: Flexible schemas for evolving data models.
- Choose appropriate consistency levels for different operations.
- Implement data archiving strategies for old content.
- Use materialized views for complex query patterns.

### Notification System

**Concept & Principles**
- Real-time delivery of user interactions.
- Multi-channel delivery (in-app, push, email).
- Notification aggregation and batching.
- User preferences and filtering.

**Real-World Usage**
- User receives push notification for likes, comments, follows.
- Notifications appear in in-app notification center.
- Batch notifications for high-frequency interactions ("X and Y liked your post").
- Delivery based on user activity and preferences.

**Implementation Example (Notification Service)**

```java
@Service
public class NotificationService {

    @Autowired
    private NotificationRepository notificationRepository;
    
    @Autowired
    private PushNotificationService pushService;
    
    @Autowired
    private WebSocketService webSocketService;
    
    @Autowired
    private UserPreferenceService preferenceService;
    
    @KafkaListener(topics = {
            "post-liked", "post-commented", "user-followed", "comment-liked"
    })
    public void processEngagementEvent(EngagementEvent event) {
        // Skip if user has notifications disabled
        if (!shouldNotify(event.getTargetUserId(), event.getType())) {
            return;
        }
        
        // Create notification entity
        Notification notification = createNotification(event);
        
        // Save to database
        notificationRepository.save(notification);
        
        // Check if we should aggregate
        List<Notification> related = findRelatedNotifications(notification);
        if (related.size() > 0) {
            // Create aggregated notification
            notification = aggregateNotifications(notification, related);
            // Mark related as aggregated
            markAsAggregated(related, notification.getId());
        }
        
        // Send real-time notification if user is active
        if (webSocketService.isUserActive(notification.getUserId())) {
            webSocketService.sendNotification(
                    notification.getUserId(), 
                    serializeNotification(notification));
        } else {
            // Send push notification if user has push enabled
            if (shouldSendPush(notification.getUserId(), notification.getType())) {
                pushService.sendPushNotification(notification);
            }
        }
    }
    
    private boolean shouldNotify(Long userId, String notificationType) {
        // Check user preferences
        UserPreferences prefs = preferenceService.getPreferences(userId);
        
        switch (notificationType) {
            case "post-liked":
                return prefs.isNotifyOnLikes();
            case "post-commented":
                return prefs.isNotifyOnComments();
            case "user-followed":
                return prefs.isNotifyOnFollows();
            // Other types
            default:
                return true;
        }
    }
    
    private List<Notification> findRelatedNotifications(Notification notification) {
        // Find recent similar notifications for aggregation
        // E.g., multiple likes on same post within short time window
        
        if (!isAggregatable(notification.getType())) {
            return Collections.emptyList();
        }
        
        return notificationRepository.findRelatedNotifications(
                notification.getUserId(),
                notification.getType(),
                notification.getObjectId(),
                notification.getCreatedAt().minusMinutes(30),
                notification.getCreatedAt(),
                10 // Limit
        );
    }
    
    private Notification aggregateNotifications(
            Notification current, List<Notification> related) {
        
        // Create new aggregated notification
        Notification aggregated = new Notification();
        aggregated.setUserId(current.getUserId());
        aggregated.setType(current.getType() + "_aggregated");
        aggregated.setObjectId(current.getObjectId());
        
        // Collect actor IDs
        Set<Long> actorIds = new HashSet<>();
        actorIds.add(current.getActorId());
        for (Notification n : related) {
            actorIds.add(n.getActorId());
        }
        
        // Set aggregated data
        aggregated.setActorCount(actorIds.size());
        aggregated.setActorIds(new ArrayList<>(actorIds));
        aggregated.setCreatedAt(current.getCreatedAt());
        
        return aggregated;
    }
    
    public List<Notification> getUserNotifications(
            Long userId, int page, int pageSize) {
        
        // Fetch paginated notifications
        Pageable pageable = PageRequest.of(
                page, pageSize, Sort.by("createdAt").descending());
        
        return notificationRepository.findByUserIdAndIsAggregatedFalse(
                userId, pageable);
    }
    
    @Scheduled(fixedRate = 3600000) // Every hour
    public void cleanupOldNotifications() {
        // Delete notifications older than 3 months
        LocalDateTime cutoff = LocalDateTime.now().minusMonths(3);
        notificationRepository.deleteByCreatedAtBefore(cutoff);
    }
}
```

**Performance & Scaling**
- Asynchronous processing of notification events.
- Batching for high-frequency notifications.
- WebSockets for real-time delivery to active users.
- Regional notification routing for lower latency.

**Pitfalls & Best Practices**
- **Notification Storms**: Viral content causing notification floods.
- **Delivery Guarantees**: Ensuring notifications reach offline devices.
- **Battery Drain**: Excessive push notifications on mobile devices.
- Implement rate limiting for notifications per user.
- Use intelligent batching and aggregation.
- Respect user preferences and quiet hours.
- Implement backpressure mechanisms for high-volume events.

### Search & Discovery

**Concept & Principles**
- Full-text search for users, hashtags, locations.
- Content discovery based on user preferences.
- Trending content identification.
- Personalized recommendations.

**Real-World Usage**
- User searches for hashtags, usernames, or locations.
- Explore page shows trending and personalized content.
- Search suggestions and autocomplete.
- Location-based content discovery.

**Implementation Example (Search Service with Elasticsearch)**

```java
@Service
public class SearchService {

    @Autowired
    private ElasticsearchOperations operations;
    
    @Autowired
    private UserSearchRepository userRepository;
    
    @Autowired
    private PostSearchRepository postRepository;
    
    @Autowired
    private HashtagSearchRepository hashtagRepository;
    
    public SearchResults search(String query, String type, int page, int size) {
        SearchResults results = new SearchResults();
        
        if (type == null || type.equals("all") || type.equals("users")) {
            // Search users
            results.setUsers(searchUsers(query, page, size));
        }
        
        if (type == null || type.equals("all") || type.equals("hashtags")) {
            // Search hashtags
            results.setHashtags(searchHashtags(query, page, size));
        }
        
        if (type == null || type.equals("all") || type.equals("posts")) {
            // Search posts
            results.setTopPosts(searchPosts(query, page, size));
        }
        
        return results;
    }
    
    private List<UserSearchResult> searchUsers(String query, int page, int size) {
        // Build Elasticsearch query
        NativeSearchQuery searchQuery = new NativeSearchQueryBuilder()
                .withQuery(
                    QueryBuilders.boolQuery()
                        .should(QueryBuilders.matchQuery("username", query)
                                .boost(3.0f))
                        .should(QueryBuilders.matchQuery("fullName", query)
                                .boost(2.0f))
                        .should(QueryBuilders.matchQuery("bio", query)
                                .boost(1.0f))
                )
                .withPageable(PageRequest.of(page, size))
                .build();
        
        // Execute search
        SearchHits<UserDocument> hits = operations.search(
                searchQuery, UserDocument.class);
        
        // Map results
        return StreamSupport.stream(hits.spliterator(), false)
                .map(hit -> {
                    UserDocument doc = hit.getContent();
                    return new UserSearchResult(
                            doc.getId(),
                            doc.getUsername(),
                            doc.getFullName(),
                            doc.getProfileImageUrl(),
                            doc.isVerified());
                })
                .collect(Collectors.toList());
    }
    
    private List<HashtagSearchResult> searchHashtags(String query, int page, int size) {
        // Remove # if present in query
        if (query.startsWith("#")) {
            query = query.substring(1);
        }
        
        // Build query
        NativeSearchQuery searchQuery = new NativeSearchQueryBuilder()
                .withQuery(QueryBuilders.prefixQuery("name", query))
                .withPageable(PageRequest.of(page, size))
                .withSort(SortBuilders.fieldSort("postCount").order(SortOrder.DESC))
                .build();
        
        // Execute search
        SearchHits<HashtagDocument> hits = operations.search(
                searchQuery, HashtagDocument.class);
        
        // Map results
        return StreamSupport.stream(hits.spliterator(), false)
                .map(hit -> {
                    HashtagDocument doc = hit.getContent();
                    return new HashtagSearchResult(
                            doc.getName(),
                            doc.getPostCount(),
                            doc.getThumbnailUrl());
                })
                .collect(Collectors.toList());
    }
    
    private List<PostSearchResult> searchPosts(String query, int page, int size) {
        // Build query for searching post captions
        NativeSearchQuery searchQuery = new NativeSearchQueryBuilder()
                .withQuery(QueryBuilders.matchQuery("caption", query))
                .withPageable(PageRequest.of(page, size))
                .withSort(SortBuilders.fieldSort("engagementScore").order(SortOrder.DESC))
                .build();
        
        // Execute search
        SearchHits<PostDocument> hits = operations.search(
                searchQuery, PostDocument.class);
        
        // Map results
        return StreamSupport.stream(hits.spliterator(), false)
                .map(hit -> {
                    PostDocument doc = hit.getContent();
                    return new PostSearchResult(
                            doc.getId(),
                            doc.getUserId(),
                            doc.getUsername(),
                            doc.getThumbnailUrl(),
                            doc.getLikeCount(),
                            doc.getCommentCount());
                })
                .collect(Collectors.toList());
    }
    
    @KafkaListener(topics = "post-created")
    public void indexNewPost(PostEvent event) {
        // Extract hashtags from caption
        Set<String> hashtags = extractHashtags(event.getCaption());
        
        // Index post document
        PostDocument postDoc = new PostDocument();
        postDoc.setId(event.getPostId());
        postDoc.setUserId(event.getUserId());
        postDoc.setUsername(event.getUsername());
        postDoc.setCaption(event.getCaption());
        postDoc.setThumbnailUrl(event.getMediaUrls().get("thumbnail"));
        postDoc.setCreatedAt(event.getCreatedAt());
        postDoc.setHashtags(new ArrayList<>(hashtags));
        
        postRepository.save(postDoc);
        
        // Update hashtag documents
        for (String tag : hashtags) {
            HashtagDocument hashtagDoc = hashtagRepository.findById(tag)
                    .orElse(new HashtagDocument(tag, 0, null));
            
            hashtagDoc.setPostCount(hashtagDoc.getPostCount() + 1);
            
            // Update thumbnail if not set or randomly to refresh
            if (hashtagDoc.getThumbnailUrl() == null || Math.random() < 0.1) {
                hashtagDoc.setThumbnailUrl(event.getMediaUrls().get("thumbnail"));
            }
            
            hashtagRepository.save(hashtagDoc);
        }
    }
}
```

**Performance & Scaling**
- Sharded Elasticsearch cluster for search scalability.
- Indexing pipeline for efficient document processing.
- Caching popular search results.
- Query optimization for typeahead suggestions.

**Pitfalls & Best Practices**
- **Search Relevance**: Balancing text matching with popularity and freshness.
- **Index Growth**: Managing index size for large content volumes.
- **Query Complexity**: Optimizing complex queries for performance.
- Implement custom analyzers for social media content (hashtags, mentions).
- Use time-based indices for efficient data rotation.
- Balance precision and recall based on query type.
- Implement search personalization based on user behavior.

### Content Moderation

**Concept & Principles**
- Automated and manual review processes.
- ML-based detection of inappropriate content.
- User reporting and escalation flows.
- Policy enforcement and compliance.

**Real-World Usage**
- Automated scanning of new uploads for policy violations.
- User reports triggering human review.
- Content filtering based on user age and preferences.
- Handling appeals and escalations.

**Implementation Example (Moderation Service)**

```java
@Service
public class ContentModerationService {

    @Autowired
    private ImageModerationClient imageClient;
    
    @Autowired
    private TextModerationClient textClient;
    
    @Autowired
    private ModerationRepository moderationRepository;
    
    @Autowired
    private ModerationQueueService queueService;
    
    @Autowired
    private MediaService mediaService;
    
    @Autowired
    private NotificationService notificationService;
    
    @KafkaListener(topics = "post-created")
    public void moderateNewPost(PostEvent event) {
        // Create moderation record
        ModerationCase moderationCase = new ModerationCase();
        moderationCase.setObjectId(event.getPostId());
        moderationCase.setObjectType("post");
        moderationCase.setUserId(event.getUserId());
        moderationCase.setCreatedAt(new Date());
        moderationCase.setStatus("PENDING");
        
        // Save initial record
        moderationRepository.save(moderationCase);
        
        // Moderate text content
        if (event.getCaption() != null && !event.getCaption().isEmpty()) {
            CompletableFuture.runAsync(() -> {
                TextModerationResult textResult = 
                        textClient.moderateText(event.getCaption());
                
                if (textResult.isViolation()) {
                    handleViolation(
                            moderationCase, 
                            "TEXT", 
                            textResult.getReason(),
                            textResult.getConfidence());
                } else {
                    updateModerationStatus(
                            moderationCase.getId(), 
                            "TEXT_APPROVED");
                }
            });
        }
        
        // Moderate image content
        if (event.getMediaUrls() != null && event.getMediaUrls().containsKey("original")) {
            CompletableFuture.runAsync(() -> {
                String imageUrl = event.getMediaUrls().get("original");
                byte[] imageBytes = mediaService.downloadMedia(imageUrl);
                
                ImageModerationResult imageResult = 
                        imageClient.moderateImage(imageBytes);
                
                if (imageResult.isViolation()) {
                    handleViolation(
                            moderationCase, 
                            "IMAGE", 
                            imageResult.getReason(),
                            imageResult.getConfidence());
                } else {
                    updateModerationStatus(
                            moderationCase.getId(), 
                            "IMAGE_APPROVED");
                }
            });
        }
    }
    
    private void handleViolation(
            ModerationCase moderationCase, 
            String contentType, 
            String reason, 
            double confidence) {
        
        // Update moderation case
        moderationCase.setStatus("FLAGGED");
        moderationCase.setContentType(contentType);
        moderationCase.setReason(reason);
        moderationCase.setConfidence(confidence);
        moderationCase.setUpdatedAt(new Date());
        
        moderationRepository.save(moderationCase);
        
        // High confidence violations are auto-rejected
        if (confidence > 0.95) {
            takeAction(moderationCase, "REJECTED", "Automated high-confidence rejection");
            return;
        }
        
        // Medium confidence goes to human review
        if (confidence > 0.70) {
            queueService.addToReviewQueue(moderationCase);
            return;
        }
        
        // Low confidence with certain violation types still needs review
        if (Arrays.asList("VIOLENCE", "CHILD_SAFETY", "HATE_SPEECH")
                .contains(reason)) {
            queueService.addToReviewQueue(moderationCase);
            return;
        }
        
        // Other low confidence cases are approved with warning
        updateModerationStatus(moderationCase.getId(), "APPROVED_WITH_WARNING");
    }
    
    public void processUserReport(ContentReport report) {
        // Check if case already exists
        Optional<ModerationCase> existingCase = 
                moderationRepository.findByObjectTypeAndObjectId(
                        report.getObjectType(), report.getObjectId());
        
        ModerationCase moderationCase;
        if (existingCase.isPresent()) {
            moderationCase = existingCase.get();
            // Increment report count
            moderationCase.setReportCount(moderationCase.getReportCount() + 1);
            
            // Multiple reports trigger review
            if (moderationCase.getReportCount() >= 3 && 
                    !"UNDER_REVIEW".equals(moderationCase.getStatus())) {
                moderationCase.setStatus("UNDER_REVIEW");
                queueService.addToReviewQueue(moderationCase);
            }
        } else {
            // Create new case
            moderationCase = new ModerationCase();
            moderationCase.setObjectId(report.getObjectId());
            moderationCase.setObjectType(report.getObjectType());
            moderationCase.setReportCount(1);
            moderationCase.setCreatedAt(new Date());
            moderationCase.setStatus("REPORTED");
            moderationCase.setReason(report.getReason());
            
            // Add to review queue
            queueService.addToReviewQueue(moderationCase);
        }
        
        moderationRepository.save(moderationCase);
    }
    
    public void reviewDecision(
            String caseId, String decision, String moderatorNotes) {
        
        ModerationCase moderationCase = 
                moderationRepository.findById(caseId).orElse(null);
        
        if (moderationCase == null) {
            throw new IllegalArgumentException("Moderation case not found");
        }
        
        takeAction(moderationCase, decision, moderatorNotes);
    }
    
    private void takeAction(
            ModerationCase moderationCase, 
            String decision, 
            String notes) {
        
        // Update case
        moderationCase.setStatus(decision);
        moderationCase.setModeratorNotes(notes);
        moderationCase.setResolvedAt(new Date());
        
        moderationRepository.save(moderationCase);
        
        // Take appropriate action based on decision
        switch (decision) {
            case "REJECTED":
                // Remove content
                if ("post".equals(moderationCase.getObjectType())) {
                    postService.setPostStatus(
                            moderationCase.getObjectId(), "REMOVED");
                }
                
                // Notify user
                notificationService.sendContentRemovalNotification(
                        moderationCase.getUserId(),
                        moderationCase.getObjectType(),
                        moderationCase.getReason());
                break;
                
            case "WARNING":
                // Content stays up but user gets warning
                notificationService.sendContentWarningNotification(
                        moderationCase.getUserId(),
                        moderationCase.getObjectType(),
                        moderationCase.getObjectId(),
                        moderationCase.getReason());
                break;
                
            case "APPROVED":
                // Clear any flags
                if ("post".equals(moderationCase.getObjectType())) {
                    postService.setPostStatus(
                            moderationCase.getObjectId(), "ACTIVE");
                }
                break;
        }
        
        // Log moderation action for audit
        auditService.logModerationAction(moderationCase, decision, notes);
    }
}
```

**Performance & Scaling**
- Parallelized moderation pipelines for high throughput.
- Prioritization based on visibility and risk.
- Batch processing for efficiency.
- Caching of previous decisions for similar content.

**Pitfalls & Best Practices**
- **False Positives**: Legitimate content incorrectly flagged.
- **Review Latency**: Delayed moderation for reported content.
- **Moderation at Scale**: Handling billions of pieces of content.
- Implement tiered moderation approach (automated -> human).
- Use confidence thresholds to balance precision and recall.
- Provide clear appeals process for users.
- Continuously train models with feedback loops.

### Security Fundamentals

**Concept & Principles**
- Authentication and authorization.
- Data privacy and protection.
- Secure media access.
- Platform integrity and abuse prevention.

**Real-World Usage**
- User authentication via JWT or OAuth.
- Private accounts with restricted content access.
- Signed URLs for media access control.
- Rate limiting to prevent scraping and abuse.

**Implementation Example (Security Config)**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private JwtTokenProvider jwtTokenProvider;
    
    @Autowired
    private UserDetailsService userDetailsService;
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authorizeRequests()
            .antMatchers("/api/auth/**").permitAll()
            .antMatchers("/api/public/**").permitAll()
            .antMatchers(HttpMethod.GET, "/api/users/*/public").permitAll()
            .antMatchers(HttpMethod.GET, "/api/media/*/thumbnail").permitAll()
            .anyRequest().authenticated()
            .and()
            .apply(new JwtConfigurer(jwtTokenProvider));
    }
    
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService)
            .passwordEncoder(passwordEncoder());
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    
    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
}

@Service
public class MediaSecurityService {

    @Value("${media.secret.key}")
    private String secretKey;
    
    @Autowired
    private GraphService graphService;
    
    @Autowired
    private UserService userService;
    
    public String generateSignedUrl(String mediaId, Long requesterId) {
        MediaMetadata media = mediaService.getMediaMetadata(mediaId);
        
        // Check permissions
        if (!canAccessMedia(media, requesterId)) {
            throw new AccessDeniedException("Not authorized to access this media");
        }
        
        // Generate signed URL
        long expiration = System.currentTimeMillis() + 3600000; // 1 hour
        
        String stringToSign = mediaId + ":" + requesterId + ":" + expiration;
        String signature = generateHmac(stringToSign, secretKey);
        
        return "/api/media/" + mediaId + 
                "?requester=" + requesterId + 
                "&expires=" + expiration + 
                "&sig=" + signature;
    }
    
    public boolean validateSignedUrl(
            String mediaId, Long requesterId, long expiration, String signature) {
        
        // Check if expired
        if (System.currentTimeMillis() > expiration) {
            return false;
        }
        
        // Validate signature
        String stringToSign = mediaId + ":" + requesterId + ":" + expiration;
        String expectedSignature = generateHmac(stringToSign, secretKey);
        
        return expectedSignature.equals(signature);
    }
    
    private boolean canAccessMedia(MediaMetadata media, Long requesterId) {
        // Public media is accessible to all
        if (!media.isPrivate()) {
            return true;
        }
        
        // Owner can always access their media
        if (media.getUserId().equals(requesterId)) {
            return true;
        }
        
        // Check if requester follows media owner
        User owner = userService.getUser(media.getUserId());
        if (owner.isPrivate()) {
            return graphService.isFollowing(requesterId, media.getUserId());
        }
        
        return true;
    }
    
    private String generateHmac(String data, String key) {
        try {
            Mac sha256_HMAC = Mac.getInstance("HmacSHA256");
            SecretKeySpec secret_key = 
                    new SecretKeySpec(key.getBytes(StandardCharsets.UTF_8), "HmacSHA256");
            sha256_HMAC.init(secret_key);
            
            byte[] hash = sha256_HMAC.doFinal(data.getBytes(StandardCharsets.UTF_8));
            return Base64.getEncoder().encodeToString(hash);
        } catch (Exception e) {
            throw new RuntimeException("Failed to generate HMAC", e);
        }
    }
}
```

**Performance & Scaling**
- Stateless authentication for horizontal scaling.
- Distributed rate limiting via Redis.
- Caching of authorization decisions.
- Token-based media access for CDN delivery.

**Pitfalls & Best Practices**
- **Permission Complexity**: Complex privacy rules affecting performance.
- **Token Security**: Preventing token theft and replay attacks.
- **Media Protection**: Ensuring private content remains private.
- Implement proper HTTPS throughout the system.
- Use short-lived tokens for sensitive operations.
- Implement defense in depth with multiple security layers.
- Regular security audits and penetration testing.

## Performance & Scalability Considerations

1. **Read-Heavy Optimization**
   - Instagram-like platforms typically have 100:1 or higher read-to-write ratios.
   - Optimize for read performance with extensive caching.
   - Denormalize data to avoid expensive joins for common queries.

2. **Media Delivery Optimization**
   - Use CDN for global content delivery.
   - Implement adaptive quality based on network conditions.
   - Progressive loading for faster perceived performance.
   - Edge caching for popular content.

3. **Feed Generation**
   - Hybrid model: push for regular users, pull for high-follower accounts.
   - Pre-compute feeds for active users.
   - Implement cursor-based pagination for efficiency.
   - Consider time-based sharding for historical feeds.

4. **Caching Strategy**
   - Multi-level caching (CDN, Redis, application cache).
   - Cache active user data aggressively.
   - Use consistent hashing for cache distribution.
   - Implement intelligent cache warming for predictable traffic.

5. **Database Sharding**
   - Horizontal sharding by user ID.
   - Consider additional sharding dimensions for large tables.
   - Implement proper shard key selection to avoid hotspots.
   - Use read replicas for query scaling.

6. **Asynchronous Processing**
   - Offload non-critical operations to background workers.
   - Use event-driven architecture for loose coupling.
   - Implement retry mechanisms for reliability.
   - Batch processing for efficiency.

7. **Microservice Scaling**
   - Independent scaling based on service load.
   - Containerization and orchestration (Kubernetes).
   - Autoscaling based on traffic patterns.
   - Service mesh for resilient communication.

8. **Global Distribution**
   - Multi-region deployment for lower latency.
   - Geo-routing to nearest datacenter.
   - Content replication strategies.
   - Consistent global caching.

9. **Handling Viral Content**
   - Special handling for trending posts.
   - Dedicated caching for viral content.
   - Rate limiting to prevent system overload.
   - Graceful degradation strategies.

10. **Resource Efficiency**
    - Optimize media storage with smart compression.
    - Use tiered storage for cost efficiency.
    - Implement TTL policies for ephemeral content.
    - Carefully manage memory usage in caching layers.

## Common Pitfalls & How to Avoid Them

1. **Feed Generation Bottlenecks**
   - **Issue**: Celebrity users with millions of followers create write amplification.
   - **Solution**: Hybrid push/pull model, with separate handling for high-follower accounts.

2. **Media Storage Costs**
   - **Issue**: Unbounded growth of media storage becomes expensive.
   - **Solution**: Tiered storage, compression optimization, cold storage for old content.

3. **Cache Invalidation Complexity**
   - **Issue**: Keeping caches consistent across services is challenging.
   - **Solution**: Event-based invalidation, TTL policies, versioned cache keys.

4. **Feed Consistency Issues**
   - **Issue**: Users seeing different content on different devices.
   - **Solution**: Consistent hashing, logical timestamps, clear update signals.

5. **Story Expiration Management**
   - **Issue**: Clock skew causing inconsistent expiration times.
   - **Solution**: Centralized TTL management, clear expiration metadata.

6. **Data Hotspots**
   - **Issue**: Popular users/content creating database bottlenecks.
   - **Solution**: Proper sharding, dedicated caching, read replicas.

7. **Notification Overload**
   - **Issue**: Viral content causing notification storms.
   - **Solution**: Intelligent batching, rate limiting, aggregation.

8. **Search Relevance vs. Performance**
   - **Issue**: Complex search queries impacting performance.
   - **Solution**: Query optimization, index tuning, caching popular searches.

9. **Content Moderation at Scale**
   - **Issue**: Reviewing billions of pieces of content efficiently.
   - **Solution**: Tiered approach, ML-based filtering, prioritization.

10. **Real-time Update Challenges**
    - **Issue**: Keeping feeds and notifications real-time without overloading.
    - **Solution**: WebSockets for active users, smart polling, push notifications.

11. **Cold Start Problem**
    - **Issue**: New services with empty caches perform poorly.
    - **Solution**: Cache warming, graceful degradation, consistent hashing in load balancing.

12. **API Versioning**
    - **Issue**: Evolving APIs breaking mobile client compatibility.
    - **Solution**: Explicit versioning, backward compatibility, graceful degradation.

## Best Practices & Maintenance

1. **Monitoring & Alerting**
   - Implement comprehensive monitoring for all services.
   - Set up alerting for anomalies and SLA breaches.
   - Use distributed tracing for request visualization.
   - Monitor cache hit rates, database performance, and service health.

2. **Deployment Strategy**
   - Implement CI/CD pipelines for consistent deployments.
   - Use blue-green or canary deployments for zero downtime.
   - Implement feature flags for controlled rollouts.
   - Regular infrastructure-as-code audits.

3. **Database Maintenance**
   - Regular index optimization and schema evolution.
   - Implement data archiving for old content.
   - Monitor query performance and optimize as needed.
   - Regular backup testing and disaster recovery drills.

4. **Caching Policy**
   - Define clear TTL strategies based on data volatility.
   - Implement cache warming for predictable high-traffic events.
   - Monitor memory usage and eviction rates.
   - Use jittered expirations to prevent thundering herd.

5. **Security Practices**
   - Regular security audits and penetration testing.
   - Implement proper authentication and authorization.
   - Use HTTPS throughout with proper certificate management.
   - Regular secret rotation and secure credential storage.

6. **Incident Response**
   - Define clear escalation paths for different incident types.
   - Implement runbooks for common failure scenarios.
   - Regular incident response drills.
   - Post-mortem analysis and continuous improvement.

7. **Performance Testing**
   - Regular load testing to identify bottlenecks.
   - Simulate viral content scenarios.
   - Test cache rebuild pathways.
   - End-to-end latency testing.

8. **API Governance**
   - Consistent API design and documentation.
   - Versioning strategy for backward compatibility.
   - Rate limiting and quota management.
   - API usage monitoring and optimization.

9. **Cost Optimization**
   - Regular review of storage and compute costs.
   - Implement auto-scaling policies to match demand.
   - Use spot instances for non-critical workloads.
   - Optimize media storage and delivery costs.

10. **Documentation & Knowledge Sharing**
    - Maintain up-to-date architecture diagrams.
    - Document service boundaries and interactions.
    - Create runbooks for operational tasks.
    - Implement knowledge sharing across teams.

## How to Discuss in a Principal Engineer Interview

1. **Start with Core Requirements**
   - Outline the primary user flows: photo uploading, feed viewing, engagement.
   - Highlight non-functional requirements: availability, latency, global scale.
   - Define success metrics: feed load time, media upload/delivery latency.

2. **Explain Architecture Decisions**
   - Justify microservice boundaries based on domain and scale.
   - Discuss polyglot persistence choices for different data types.
   - Explain caching strategy and the push/pull model for feeds.

3. **Dive into Scalability Approach**
   - Discuss how each component scales horizontally.
   - Explain sharding strategy for databases.
   - Detail the caching hierarchy for performance.
   - Address handling of viral content and celebrity users.

4. **Address Data Consistency**
   - Explain consistency models for different operations.
   - Discuss eventual consistency challenges in distributed systems.
   - Detail strategies for maintaining feed and notification consistency.

5. **Cover Failure Handling**
   - Explain resilience patterns (circuit breakers, retries).
   - Discuss partial degradation strategies.
   - Detail disaster recovery approach.

6. **Touch on Performance Optimization**
   - Highlight areas requiring special attention (feed generation, media delivery).
   - Discuss monitoring and alerting for performance bottlenecks.
   - Explain performance testing approach.

7. **Discuss Security & Privacy**
   - Address authentication and authorization strategy.
   - Explain privacy controls for content.
   - Detail content moderation approach.

8. **Explain Global Distribution**
   - Discuss multi-region deployment strategy.
   - Detail content distribution approach.
   - Address consistency challenges across regions.

9. **Mention Future Evolution**
   - Discuss how architecture supports new features.
   - Address potential scaling challenges as user base grows.
   - Explain migration strategies for evolving components.

10. **Highlight Trade-offs Made**
    - Be transparent about architectural trade-offs.
    - Discuss alternative approaches considered.
    - Explain how business requirements influenced technical decisions.

## Conclusion

This high-level design outlines a **scalable, resilient architecture** for an Instagram-like platform capable of handling millions of users and billions of photos. The system leverages **microservices architecture** with **domain-driven boundaries**, allowing for independent scaling and evolution of components.

The design emphasizes **optimized read paths** through extensive caching and denormalization, recognizing the read-heavy nature of social media platforms. It implements a **hybrid fan-out model** for feed generation, balancing efficiency for regular users while handling the scale challenges of celebrity accounts.

For media handling, the system employs a **multi-tiered strategy** with intelligent processing, storage, and delivery optimizations. Content is delivered through **CDNs with adaptive quality** based on network conditions, ensuring fast loading times while maintaining visual quality.

The architecture addresses **real-time features** through event-driven design, enabling immediate notifications and feed updates. It also incorporates **robust content moderation** capabilities, balancing automated filtering with human review for policy enforcement.

By implementing the outlined strategies for caching, sharding, and asynchronous processing, this architecture provides a solid foundation for a platform that delivers a responsive, reliable user experience while efficiently managing backend resources. The design accommodates future growth and feature expansion while maintaining performance and reliability at scale.
