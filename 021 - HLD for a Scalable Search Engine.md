# High-Level Design for a Scalable Search Engine

## Table of Contents
- [Overview & Core Requirements](#overview--core-requirements)
- [Major Microservices & Responsibilities](#major-microservices--responsibilities)
  - [Crawler Service](#crawler-service)
  - [Indexer Service](#indexer-service)
  - [URL Processor](#url-processor)
  - [Query Service](#query-service)
  - [Storage Service](#storage-service)
  - [Ranking Service](#ranking-service)
  - [URL Refresh Service](#url-refresh-service)
  - [Analytics Service](#analytics-service)
  - [User Interface Service](#user-interface-service)
  - [Monitoring & SLA Service](#monitoring--sla-service)
- [High-Level Architecture & Data Flow](#high-level-architecture--data-flow)
- [Key System Design Topics & Deep Dives](#key-system-design-topics--deep-dives)
  - [Web Crawling Fundamentals](#web-crawling-fundamentals)
  - [Inverted Index & Query Processing](#inverted-index--query-processing)
  - [Distributed Storage (HDFS)](#distributed-storage-hdfs)
  - [Stemming & NLP Processing](#stemming--nlp-processing)
  - [Caching Strategy](#caching-strategy)
  - [MapReduce & Distributed Processing](#mapreduce--distributed-processing)
  - [Elasticsearch Implementation](#elasticsearch-implementation)
  - [Daily URL Refreshing](#daily-url-refreshing)
  - [Monitoring & Reliability](#monitoring--reliability)
- [Performance & Scalability Considerations](#performance--scalability-considerations)
- [Common Pitfalls & How to Avoid Them](#common-pitfalls--how-to-avoid-them)
- [Best Practices & Maintenance](#best-practices--maintenance)
- [How to Discuss in a Principal Engineer Interview](#how-to-discuss-in-a-principal-engineer-interview)
- [Conclusion](#conclusion)

## Overview & Core Requirements

We want to build a **search engine** where users can:

1. **Enter a single keyword** and get up to 10 relevant results.
2. **Click on a result** to be directed to the corresponding page.
3. Receive results where a "good match" means the URL contains the specific keyword.
4. Search across URLs that are **refreshed daily** for updates.
5. Have **all grammatical tenses map** to the same base words (lemmatization).
6. Have searches ignore **stop words** (a, an, you, or, etc.).

### Additional Requirements

- **99.99% availability** (allows for ~52 minutes of downtime per year).
- **Search latency less than 200ms** for query response.
- Simple search model that doesn't require complex relevancy ranking.
- Ability to handle single-keyword queries efficiently.
- Daily crawling/refreshing of indexed URLs.
- Support for basic natural language processing (NLP) for word stemming.

## Major Microservices & Responsibilities

### Crawler Service
- **Responsibilities**: Discovers and fetches web pages, schedules crawling, manages crawl policies.
- **Data Store**: URL frontier storage in Redis or a distributed queue system.
- **Notes**: 
  - Must respect robots.txt and crawl etiquette.
  - Implements politeness policies to avoid overwhelming web servers.
  - Partitions the web for distributed crawling.
  - Detects and handles duplicates to avoid redundant crawling.

### Indexer Service
- **Responsibilities**: Processes crawled pages, extracts keywords, builds inverted index.
- **Data Store**: Distributed file system (HDFS) for raw index data.
- **Notes**:
  - Implements word stemming to handle grammatical tenses.
  - Filters out stop words.
  - Builds forward and inverted indices.
  - Distributes index creation using MapReduce or Spark.

### URL Processor
- **Responsibilities**: Analyzes URLs for keyword presence, metadata extraction.
- **Data Store**: NoSQL database (e.g., MongoDB) for URL metadata.
- **Notes**:
  - Extracts and stores URL components.
  - Identifies keywords in URLs for "good match" detection.
  - Maintains URL relationship graph.

### Query Service
- **Responsibilities**: Processes user queries, performs search against index, returns results.
- **Data Store**: In-memory cache (Redis) for frequent queries.
- **Notes**:
  - Must respond in under 200ms.
  - Handles query preprocessing (stop word removal, stemming).
  - Optimized for single-keyword queries.
  - Implements caching for hot queries.

### Storage Service
- **Responsibilities**: Manages distributed storage for indices, document storage, and metadata.
- **Data Store**: 
  - HDFS for raw crawled content.
  - Distributed NoSQL (HBase/Cassandra) for processed indices.
- **Notes**:
  - Ensures data replication and fault tolerance.
  - Implements efficient access patterns for inverted indices.
  - Manages storage lifecycle.

### Ranking Service
- **Responsibilities**: Determines the order of search results based on simple matching criteria.
- **Data Store**: In-memory computation, potentially backed by Redis.
- **Notes**:
  - Simple implementation since complex relevancy ranking isn't required.
  - Focuses on keyword presence in URL as primary signal.
  - Maintains basic ranking signals.

### URL Refresh Service
- **Responsibilities**: Manages daily recrawling of indexed URLs, detects changes.
- **Data Store**: 
  - Queue system (RabbitMQ/Kafka) for refresh tasks.
  - Database for tracking last refresh times.
- **Notes**:
  - Schedules URL refreshes based on daily requirement.
  - Prioritizes URLs based on change frequency.
  - Updates index when changes are detected.

### Analytics Service
- **Responsibilities**: Tracks query patterns, performance metrics, indexing statistics.
- **Data Store**: Time-series database (InfluxDB/Prometheus) for metrics.
- **Notes**:
  - Provides insights for service optimization.
  - Tracks performance against SLA.
  - Identifies popular queries for caching strategies.

### User Interface Service
- **Responsibilities**: Handles user interactions, displays search results, manages click redirects.
- **Data Flow**: Communicates with Query Service, tracks user interactions.
- **Notes**:
  - Lightweight frontend focused on query input and result display.
  - Manages user session for analytics.

### Monitoring & SLA Service
- **Responsibilities**: Ensures system meets 99.99% availability and latency requirements.
- **Data Store**: Metrics database, alerting system.
- **Notes**:
  - Tracks all components for health and performance.
  - Implements alerting for approaching SLA breaches.
  - Provides dashboards for system status visualization.

## High-Level Architecture & Data Flow

1. **Crawling Flow**
   - **Crawler Service** fetches web pages according to crawl policy.
   - Raw HTML/content saved to **Storage Service** (HDFS).
   - **URL Processor** analyzes URLs and extracts metadata.
   - Crawl metadata published to Kafka to trigger indexing.

2. **Indexing Flow**
   - **Indexer Service** processes crawled content from HDFS.
   - Content parsed, tokenized, with stop words removed and stemming applied.
   - MapReduce jobs build inverted index mapping terms to document IDs.
   - Completed indices stored in distributed database (HBase/Cassandra).
   - Index metadata published to Kafka for query service.

3. **Query Flow**
   - User submits keyword query via **User Interface Service**.
   - **Query Service** preprocesses query (removes stop words, applies stemming).
   - Service checks cache for results, or queries index if cache miss.
   - Retrieved document IDs passed to **Ranking Service** to order results.
   - Top 10 results returned to user, with click tracking.

4. **URL Refresh Flow**
   - **URL Refresh Service** schedules daily crawls of indexed URLs.
   - Changed content detected by comparing checksums/timestamps.
   - Updates published to Kafka to trigger re-indexing of changed content.
   - Indices updated incrementally without full rebuild.

5. **Monitoring Flow**
   - All services emit metrics to **Monitoring & SLA Service**.
   - System tracks latency, availability, error rates.
   - Alerting triggered for potential SLA violations.
   - Analytics processed for system optimization.

## Key System Design Topics & Deep Dives

### Web Crawling Fundamentals

**Concept & Principles**
- Distributed, polite web crawling with controlled parallelism.
- Handling robots.txt and crawl delays.
- URL frontier management and prioritization.
- Duplicate detection and avoidance.

**Real-World Usage**
- Partitioned crawlers with consistent hashing by domain.
- Rate-limited requests to respect target websites.
- Priority queues for important domains/pages.

**Java Code Example (Crawler)**

```java
@Component
public class WebCrawler {
    
    @Autowired
    private RobotsTxtService robotsTxtService;
    
    @Autowired
    private DocumentRepository documentRepository;
    
    @Autowired
    private UrlFrontierQueue urlFrontier;
    
    @Autowired
    private KafkaTemplate<String, CrawlEvent> kafkaTemplate;
    
    // Crawl rate (pages per second) per domain
    private static final Map<String, RateLimiter> domainRateLimiters = new ConcurrentHashMap<>();
    
    @Async
    public void crawlUrl(String url) {
        try {
            URL parsedUrl = new URL(url);
            String domain = parsedUrl.getHost();
            
            // Check robots.txt compliance
            if (!robotsTxtService.isAllowed(parsedUrl)) {
                log.info("Skipping URL {} - disallowed by robots.txt", url);
                return;
            }
            
            // Apply rate limiting per domain
            RateLimiter limiter = domainRateLimiters.computeIfAbsent(
                domain, k -> RateLimiter.create(2.0) // 2 requests per second
            );
            
            if (!limiter.tryAcquire(1, TimeUnit.SECONDS)) {
                // Re-queue for later if rate limit exceeded
                urlFrontier.enqueue(url, System.currentTimeMillis() + 5000);
                return;
            }
            
            // Fetch document
            Document document = fetchDocument(url);
            
            // Store document
            String documentId = documentRepository.store(document);
            
            // Extract links for further crawling
            List<String> links = extractLinks(document);
            for (String link : links) {
                urlFrontier.enqueue(link, System.currentTimeMillis());
            }
            
            // Publish event for indexing
            CrawlEvent event = new CrawlEvent(documentId, url, System.currentTimeMillis());
            kafkaTemplate.send("crawl-events", domain, event);
            
        } catch (Exception e) {
            log.error("Error crawling URL {}: {}", url, e.getMessage());
        }
    }
    
    private Document fetchDocument(String url) {
        // Implementation to fetch the document
        // Includes handling of HTTP requests, headers, timeouts, etc.
        // ...
    }
    
    private List<String> extractLinks(Document document) {
        // Extract and normalize links from document
        // ...
    }
}
```

**Performance & Scaling**
- Horizontal scaling with domain partitioning.
- Politeness through rate limiting and crawl delays.
- Priority-based crawling to focus on important content.

**Pitfalls & Best Practices**
- **Crawl traps**: Detect and avoid infinite URL spaces.
- **Host overload**: Implement adaptive rate limiting.
- **Freshness vs. coverage**: Balance recrawl frequency with new URL discovery.
- Set clear crawl policies with respect for website requirements.
- Implement exponential backoff for retries on failure.
- Use consistent URL canonicalization to avoid duplicates.

### Inverted Index & Query Processing

**Concept & Principles**
- Inverted index mapping terms to document IDs.
- Efficient term lookup and document retrieval.
- Query preprocessing (stemming, stop word removal).
- Intersection operations for multi-term queries (future extension).

**Real-World Usage**
- Term-to-document mapping for efficient search.
- Position information for phrase queries (future extension).
- Term frequency statistics for basic relevance (future extension).

**Java Code Example (Inverted Index)**

```java
@Service
public class InvertedIndexService {

    @Autowired
    private TermDictionaryRepository termDictionary;
    
    @Autowired
    private PostingListRepository postingLists;
    
    @Autowired
    private StemmingService stemmingService;
    
    private final Set<String> stopWords = new HashSet<>(Arrays.asList(
        "a", "an", "the", "and", "or", "of", "to", "in", "is", "you", "that"
        // more stop words...
    ));
    
    public void indexDocument(String documentId, String content, String url) {
        // Tokenize content
        List<String> tokens = tokenize(content);
        
        // Process URL for "good match" detection
        List<String> urlTokens = tokenizeUrl(url);
        
        // Count term frequencies
        Map<String, TermInfo> termCounts = new HashMap<>();
        
        // Process all tokens
        for (int position = 0; position < tokens.size(); position++) {
            String token = tokens.get(position);
            
            // Skip stop words
            if (stopWords.contains(token)) {
                continue;
            }
            
            // Apply stemming
            String stem = stemmingService.stem(token);
            
            // Update term info
            TermInfo info = termCounts.computeIfAbsent(stem, k -> new TermInfo());
            info.frequency++;
            info.positions.add(position);
        }
        
        // Index URL tokens separately (for "good match")
        for (String urlToken : urlTokens) {
            if (stopWords.contains(urlToken)) {
                continue;
            }
            
            String stem = stemmingService.stem(urlToken);
            
            // Mark as present in URL
            termCounts.computeIfAbsent(stem, k -> new TermInfo()).inUrl = true;
        }
        
        // Update posting lists for each term
        for (Map.Entry<String, TermInfo> entry : termCounts.entrySet()) {
            String term = entry.getKey();
            TermInfo info = entry.getValue();
            
            // Get or create term ID
            long termId = termDictionary.getOrCreateTermId(term);
            
            // Update posting list
            PostingEntry posting = new PostingEntry(
                documentId,
                info.frequency,
                info.positions,
                info.inUrl
            );
            
            postingLists.addPosting(termId, posting);
        }
    }
    
    public List<SearchResult> search(String query) {
        // Preprocess query
        List<String> queryTokens = tokenize(query);
        List<String> stems = new ArrayList<>();
        
        for (String token : queryTokens) {
            if (stopWords.contains(token)) {
                continue;
            }
            
            String stem = stemmingService.stem(token);
            stems.add(stem);
        }
        
        // For now, just handle single-term query (as per requirements)
        if (stems.isEmpty()) {
            return Collections.emptyList();
        }
        
        String stem = stems.get(0);
        long termId = termDictionary.getTermId(stem);
        
        if (termId == -1) {
            return Collections.emptyList();
        }
        
        // Retrieve posting list
        List<PostingEntry> postings = postingLists.getPostings(termId);
        
        // Sort by "good match" (URL contains term) first, then by frequency
        postings.sort((a, b) -> {
            if (a.isInUrl() != b.isInUrl()) {
                return a.isInUrl() ? -1 : 1;
            }
            return Integer.compare(b.getFrequency(), a.getFrequency());
        });
        
        // Convert to search results (limit to 10)
        return postings.stream()
            .limit(10)
            .map(p -> new SearchResult(p.getDocumentId(), p.isInUrl()))
            .collect(Collectors.toList());
    }
    
    private List<String> tokenize(String content) {
        // Implementation to tokenize text
        // ...
    }
    
    private List<String> tokenizeUrl(String url) {
        // Implementation to extract terms from URL
        // ...
    }
    
    // Helper classes
    private static class TermInfo {
        int frequency = 0;
        List<Integer> positions = new ArrayList<>();
        boolean inUrl = false;
    }
}
```

**Performance & Scaling**
- Distributed index building with MapReduce.
- Sharded index for parallel query processing.
- Memory-mapped files for fast disk-based access.

**Pitfalls & Best Practices**
- **Index size explosion**: Efficient compression of posting lists.
- **Query latency**: Optimize for common cases like single-term queries.
- **Update cost**: Incremental index updates for changed documents.
- Use skip lists or other optimizations for posting list traversal.
- Balance index size vs. query performance with appropriate compression.
- Consider tiered storage with hot terms in memory, cold terms on disk.

### Distributed Storage (HDFS)

**Concept & Principles**
- Reliable distributed storage for massive datasets.
- Block-based file system with replication.
- Data locality for efficient processing.
- Fault tolerance and high availability.

**Real-World Usage**
- Storing raw crawled web pages.
- Supporting MapReduce-based indexing.
- Archiving historical crawl data.

**Java Code Example (HDFS Integration)**

```java
@Service
public class CrawlDataStorageService {

    private FileSystem hdfs;
    
    @PostConstruct
    public void init() throws IOException {
        Configuration conf = new Configuration();
        conf.set("fs.defaultFS", "hdfs://namenode:8020");
        hdfs = FileSystem.get(conf);
    }
    
    public String storeRawDocument(String url, byte[] content) throws IOException {
        // Generate document ID based on URL
        String documentId = generateDocumentId(url);
        
        // Determine storage path (organized by domain/date)
        URL parsedUrl = new URL(url);
        String domain = parsedUrl.getHost();
        String date = new SimpleDateFormat("yyyy/MM/dd").format(new Date());
        Path filePath = new Path("/crawl-data/" + domain + "/" + date + "/" + documentId);
        
        // Write content to HDFS
        try (FSDataOutputStream out = hdfs.create(filePath)) {
            out.write(content);
        }
        
        return documentId;
    }
    
    public byte[] retrieveRawDocument(String documentId) throws IOException {
        // Find the document path
        Path filePath = findDocumentPath(documentId);
        
        if (filePath == null || !hdfs.exists(filePath)) {
            throw new FileNotFoundException("Document not found: " + documentId);
        }
        
        // Read content from HDFS
        try (FSDataInputStream in = hdfs.open(filePath)) {
            FileStatus status = hdfs.getFileStatus(filePath);
            byte[] content = new byte[(int) status.getLen()];
            in.readFully(content);
            return content;
        }
    }
    
    private String generateDocumentId(String url) {
        // Generate a unique ID based on URL
        return DigestUtils.md5Hex(url);
    }
    
    private Path findDocumentPath(String documentId) throws IOException {
        // Find document path in the HDFS structure
        // This is a simplified implementation; in practice, would use a metadata
        // database to track document locations
        Path crawlDataRoot = new Path("/crawl-data");
        
        RemoteIterator<LocatedFileStatus> domainDirs = hdfs.listFiles(crawlDataRoot, true);
        while (domainDirs.hasNext()) {
            LocatedFileStatus status = domainDirs.next();
            if (status.getPath().getName().equals(documentId)) {
                return status.getPath();
            }
        }
        
        return null;
    }
}
```

**Performance & Scaling**
- Horizontal scaling by adding data nodes.
- Block-level replication for fault tolerance.
- Data locality optimization for processing.

**Pitfalls & Best Practices**
- **Small file problem**: HDFS is inefficient for many small files.
- **Namenode bottleneck**: Single point of metadata management.
- **Replication overhead**: Storage multiplied by replication factor.
- Use HAR (Hadoop Archive) files for small file consolidation.
- Implement metadata database alongside HDFS for file tracking.
- Consider HDFS federation for scaling metadata operations.

### Stemming & NLP Processing

**Concept & Principles**
- Reducing words to their base or root form.
- Handling different grammatical tenses and forms.
- Stop word removal for noise reduction.
- Language detection and specialized processing.

**Real-World Usage**
- Query preprocessing to standardize terms.
- Index building to normalize document terms.
- Supporting "tense-agnostic" matching.

**Java Code Example (Stemming Service)**

```java
@Service
public class StemmingService {

    private final PorterStemmer stemmer;
    private final Set<String> stopWords;
    
    public StemmingService() {
        // Initialize Porter stemmer
        this.stemmer = new PorterStemmer();
        
        // Initialize stop words
        this.stopWords = new HashSet<>(Arrays.asList(
            "a", "an", "the", "and", "or", "of", "to", "in", "is", "you", "that"
            // more stop words...
        ));
    }
    
    public String stem(String word) {
        if (word == null || word.isEmpty()) {
            return word;
        }
        
        // Convert to lowercase for consistency
        word = word.toLowerCase();
        
        // Skip stop words
        if (stopWords.contains(word)) {
            return null;
        }
        
        // Apply Porter stemming algorithm
        stemmer.setCurrent(word);
        stemmer.stem();
        return stemmer.getCurrent();
    }
    
    public List<String> tokenizeAndStem(String text) {
        List<String> tokens = tokenize(text);
        List<String> stems = new ArrayList<>();
        
        for (String token : tokens) {
            String stem = stem(token);
            if (stem != null) {
                stems.add(stem);
            }
        }
        
        return stems;
    }
    
    private List<String> tokenize(String text) {
        // Simple tokenization by whitespace and punctuation
        return Arrays.asList(text.toLowerCase()
            .replaceAll("[^a-zA-Z0-9]", " ")
            .split("\\s+"));
    }
    
    public boolean isStopWord(String word) {
        return stopWords.contains(word.toLowerCase());
    }
}

// Simplified Porter Stemmer implementation
class PorterStemmer {
    private String current;
    
    public void setCurrent(String word) {
        this.current = word;
    }
    
    public String getCurrent() {
        return current;
    }
    
    public void stem() {
        // This is a simplified implementation
        // In practice, would use a full Porter stemming algorithm
        // with multiple steps for suffix removal
        
        // Example rules (extremely simplified):
        if (current.endsWith("ing")) {
            current = current.substring(0, current.length() - 3);
        } else if (current.endsWith("ed")) {
            current = current.substring(0, current.length() - 2);
        } else if (current.endsWith("s") && !current.endsWith("ss")) {
            current = current.substring(0, current.length() - 1);
        }
        
        // Many more rules would be implemented here
    }
}
```

**Performance & Scaling**
- Precomputed stem dictionaries for common words.
- Distributed processing for bulk document stemming.
- Caching frequently stemmed terms.

**Pitfalls & Best Practices**
- **Over-stemming**: Too aggressive stemming reduces precision.
- **Under-stemming**: Too conservative stemming reduces recall.
- **Language detection failures**: Wrong language model applied.
- Use established algorithms like Porter or Snowball stemming.
- Consider lemmatization for higher accuracy (but higher cost).
- Build language-specific stemming rules for multi-language support.

### Caching Strategy

**Concept & Principles**
- Multi-level caching for performance optimization.
- Query result caching to minimize index lookups.
- Cache invalidation strategies for freshness.
- Distributed caching for scalability.

**Real-World Usage**
- Hot query results cached in Redis.
- Posting list fragments cached for frequent terms.
- Cache warming for anticipated queries.

**Java Code Example (Redis Query Cache)**

```java
@Service
public class QueryCacheService {

    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    // Cache TTL in seconds (30 minutes)
    private static final long CACHE_TTL = 1800;
    
    // Key prefix for query cache
    private static final String CACHE_KEY_PREFIX = "query:result:";
    
    public List<SearchResult> getCachedResults(String query) {
        String normalizedQuery = normalizeQuery(query);
        String cacheKey = CACHE_KEY_PREFIX + normalizedQuery;
        
        String cachedJson = redisTemplate.opsForValue().get(cacheKey);
        if (cachedJson != null) {
            try {
                return objectMapper.readValue(
                    cachedJson, 
                    new TypeReference<List<SearchResult>>() {}
                );
            } catch (Exception e) {
                log.error("Error deserializing cached results: {}", e.getMessage());
            }
        }
        
        return null; // Cache miss
    }
    
    public void cacheResults(String query, List<SearchResult> results) {
        String normalizedQuery = normalizeQuery(query);
        String cacheKey = CACHE_KEY_PREFIX + normalizedQuery;
        
        try {
            String json = objectMapper.writeValueAsString(results);
            redisTemplate.opsForValue().set(cacheKey, json, CACHE_TTL, TimeUnit.SECONDS);
        } catch (Exception e) {
            log.error("Error caching search results: {}", e.getMessage());
        }
    }
    
    public void invalidateCache(String query) {
        String normalizedQuery = normalizeQuery(query);
        String cacheKey = CACHE_KEY_PREFIX + normalizedQuery;
        redisTemplate.delete(cacheKey);
    }
    
    private String normalizeQuery(String query) {
        // Normalize query for caching (lowercase, trim whitespace)
        return query.toLowerCase().trim();
    }
    
    // Warm cache for common queries
    @Scheduled(fixedRate = 3600000) // Every hour
    public void warmCache() {
        List<String> popularQueries = getPopularQueries();
        for (String query : popularQueries) {
            // Only warm if not in cache
            if (getCachedResults(query) == null) {
                try {
                    // Execute query and cache results
                    // This would call the search service in a real implementation
                } catch (Exception e) {
                    log.error("Error warming cache for query {}: {}", query, e.getMessage());
                }
            }
        }
    }
    
    private List<String> getPopularQueries() {
        // In a real implementation, this would retrieve popular queries
        // from an analytics database
        return Arrays.asList("java", "python", "javascript", "spring", "cloud");
    }
}
```

**Performance & Scaling**
- Redis cluster for distributed caching.
- Memory-optimized storage for high-volume queries.
- Tiered caching with in-process and distributed layers.

**Pitfalls & Best Practices**
- **Cache invalidation**: Determining when to refresh cached results.
- **Memory pressure**: Managing cache size with eviction policies.
- **Cold cache**: Poor performance after restarts or scaling events.
- Use cache warming for predictable query patterns.
- Implement intelligent TTL based on term volatility.
- Monitor cache hit rates to optimize caching strategy.

### MapReduce & Distributed Processing

**Concept & Principles**
- Distributed processing framework for large datasets.
- Map phase for parallel processing of input chunks.
- Reduce phase for aggregation of intermediate results.
- Fault tolerance and automatic retries.

**Real-World Usage**
- Building inverted index from crawled documents.
- Batch processing for analytics and metrics.
- URL filtering and de-duplication.

**Java Code Example (MapReduce for Inverted Index)**

```java
public class InvertedIndexJob {

    public static class TokenizerMapper extends Mapper<Object, Text, Text, Text> {
        private Text word = new Text();
        private Text docInfo = new Text();
        private StemmingService stemmingService = new StemmingService();
        
        @Override
        public void map(Object key, Text value, Context context) throws IOException, InterruptedException {
            // Input format: <document_id><tab><document_content><tab><url>
            String[] parts = value.toString().split("\t");
            if (parts.length < 3) {
                return;
            }
            
            String documentId = parts[0];
            String content = parts[1];
            String url = parts[2];
            
            // Process content tokens
            tokenizeAndEmit(documentId, content, url, false, context);
            
            // Process URL tokens specially (for "good match")
            tokenizeAndEmit(documentId, url, url, true, context);
        }
        
        private void tokenizeAndEmit(String documentId, String text, String url, boolean isUrl, 
                                   Context context) throws IOException, InterruptedException {
            // Tokenize and count term frequencies
            Map<String, Integer> termFrequencies = new HashMap<>();
            
            for (String token : text.split("\\s+")) {
                // Skip stop words
                if (stemmingService.isStopWord(token)) {
                    continue;
                }
                
                // Apply stemming
                String stem = stemmingService.stem(token);
                if (stem == null || stem.isEmpty()) {
                    continue;
                }
                
                // Count frequency
                termFrequencies.put(stem, termFrequencies.getOrDefault(stem, 0) + 1);
            }
            
            // Emit <term, doc_id:freq:isUrl> pairs
            for (Map.Entry<String, Integer> entry : termFrequencies.entrySet()) {
                word.set(entry.getKey());
                
                // Format: <doc_id>:<frequency>:<is_url>
                docInfo.set(documentId + ":" + entry.getValue() + ":" + (isUrl ? "1" : "0"));
                
                context.write(word, docInfo);
            }
        }
    }
    
    public static class IndexReducer extends Reducer<Text, Text, Text, Text> {
        private Text result = new Text();
        
        @Override
        public void reduce(Text key, Iterable<Text> values, Context context)
                throws IOException, InterruptedException {
            // Combine posting list for the term
            StringBuilder postingList = new StringBuilder();
            
            for (Text val : values) {
                if (postingList.length() > 0) {
                    postingList.append(",");
                }
                postingList.append(val.toString());
            }
            
            result.set(postingList.toString());
            context.write(key, result);
        }
    }
    
    public static void main(String[] args) throws Exception {
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf, "inverted index");
        
        job.setJarByClass(InvertedIndexJob.class);
        job.setMapperClass(TokenizerMapper.class);
        job.setReducerClass(IndexReducer.class);
        
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(Text.class);
        
        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}
```

**Performance & Scaling**
- Parallel processing across hundreds or thousands of nodes.
- Data locality optimization to minimize network transfer.
- Incremental processing for index updates.

**Pitfalls & Best Practices**
- **Skewed data**: Some reducers getting much more work than others.
- **Shuffle bottleneck**: Network congestion during data transfer.
- **Job failures**: Long-running jobs more likely to face node failures.
- Use combiners to reduce network traffic during the shuffle phase.
- Implement custom partitioners for balanced workload distribution.
- Design for idempotent operations to handle retries gracefully.

### Elasticsearch Implementation

**Concept & Principles**
- Distributed search and analytics engine.
- Built on Lucene for indexing and retrieval.
- JSON-based schema and REST API.
- Clustering for high availability and scaling.

**Real-World Usage**
- Alternative implementation for index storage and query.
- Advanced text analysis for multilingual support.
- Near real-time search capabilities.

**Java Code Example (Elasticsearch Integration)**

```java
@Service
public class ElasticsearchIndexService {

    private final RestHighLevelClient client;
    
    @Autowired
    private StemmingService stemmingService;
    
    public ElasticsearchIndexService(@Value("${elasticsearch.host}") String host,
                                    @Value("${elasticsearch.port}") int port) {
        this.client = new RestHighLevelClient(
            RestClient.builder(new HttpHost(host, port, "http"))
        );
    }
    
    @PreDestroy
    public void closeClient() throws IOException {
        if (client != null) {
            client.close();
        }
    }
    
    public void indexDocument(String documentId, String content, String url) throws IOException {
        // Extract URL keywords for "good match" detection
        List<String> urlTokens = tokenizeUrl(url);
        
        // Prepare the document
        Map<String, Object> document = new HashMap<>();
        document.put("content", content);
        document.put("url", url);
        document.put("url_tokens", urlTokens);
        document.put("indexed_at", new Date());
        
        // Index the document
        IndexRequest request = new IndexRequest("pages")
            .id(documentId)
            .source(document);
        
        client.index(request, RequestOptions.DEFAULT);
    }
    
    public List<SearchResult> search(String keyword) throws IOException {
        // Process the keyword
        if (stemmingService.isStopWord(keyword)) {
            return Collections.emptyList();
        }
        
        String stem = stemmingService.stem(keyword);
        if (stem == null || stem.isEmpty()) {
            return Collections.emptyList();
        }
        
        // Build the query
        QueryBuilder contentQuery = QueryBuilders.matchQuery("content", stem);
        QueryBuilder urlQuery = QueryBuilders.termQuery("url_tokens", stem);
        
        // Boost URL matches (good matches)
        QueryBuilder combinedQuery = QueryBuilders.boolQuery()
            .should(contentQuery)
            .should(urlQuery.boost(5.0f));
        
        // Execute search
        SearchRequest searchRequest = new SearchRequest("pages");
        searchRequest.source(new SearchSourceBuilder()
            .query(combinedQuery)
            .size(10)); // Limit to 10 results
        
        SearchResponse response = client.search(searchRequest, RequestOptions.DEFAULT);
        
        // Process results
        List<SearchResult> results = new ArrayList<>();
        for (SearchHit hit : response.getHits().getHits()) {
            Map<String, Object> source = hit.getSourceAsMap();
            String url = (String) source.get("url");
            boolean isGoodMatch = isGoodMatch(hit, stem);
            
            results.add(new SearchResult(hit.getId(), url, isGoodMatch));
        }
        
        return results;
    }
    
    private List<String> tokenizeUrl(String url) {
        // Extract meaningful tokens from URL
        // ...
        
        // Apply stemming to each token
        List<String> stems = new ArrayList<>();
        for (String token : tokens) {
            if (!stemmingService.isStopWord(token)) {
                String stem = stemmingService.stem(token);
                if (stem != null && !stem.isEmpty()) {
                    stems.add(stem);
                }
            }
        }
        
        return stems;
    }
    
    private boolean isGoodMatch(SearchHit hit, String term) {
        // Check if the URL tokens field contains the term
        Map<String, Object> source = hit.getSourceAsMap();
        @SuppressWarnings("unchecked")
        List<String> urlTokens = (List<String>) source.get("url_tokens");
        
        return urlTokens != null && urlTokens.contains(term);
    }
}
```

**Performance & Scaling**
- Horizontal scaling with shard distribution.
- Query routing based on document ID.
- Replicas for high availability and read scaling.

**Pitfalls & Best Practices**
- **Shard sizing**: Too many small shards cause overhead.
- **Field explosion**: Too many fields in mapping cause memory issues.
- **Query complexity**: Certain queries can be very resource-intensive.
- Define explicit mappings rather than relying on dynamic mapping.
- Use appropriate shard counts based on node resources.
- Monitor cluster health and adjust settings based on load.

### Daily URL Refreshing

**Concept & Principles**
- Systematic recrawling of indexed URLs.
- Change detection to minimize unnecessary processing.
- Prioritization based on update frequency or importance.
- Incremental index updates for efficiency.

**Real-World Usage**
- Daily scheduling of URL recrawls.
- Comparing document checksums for change detection.
- Efficient updating of affected index entries.

**Java Code Example (URL Refresh Service)**

```java
@Service
public class UrlRefreshService {

    @Autowired
    private UrlRepository urlRepository;
    
    @Autowired
    private WebCrawler crawler;
    
    @Autowired
    private DocumentRepository documentRepository;
    
    @Autowired
    private IndexerService indexerService;
    
    // Number of URLs to process in each batch
    private static final int BATCH_SIZE = 1000;
    
    @Scheduled(cron = "0 0 0 * * *") // Run at midnight every day
    public void scheduleUrlRefresh() {
        log.info("Starting daily URL refresh");
        
        // Get the total number of URLs to refresh
        long totalUrls = urlRepository.count();
        long totalBatches = (totalUrls + BATCH_SIZE - 1) / BATCH_SIZE;
        
        for (int batch = 0; batch < totalBatches; batch++) {
            int offset = batch * BATCH_SIZE;
            refreshUrlBatch(offset, BATCH_SIZE);
        }
        
        log.info("Completed daily URL refresh");
    }
    
    @Async
    public void refreshUrlBatch(int offset, int batchSize) {
        List<UrlEntity> urls = urlRepository.findAll(PageRequest.of(offset / batchSize, batchSize));
        
        for (UrlEntity url : urls) {
            try {
                refreshUrl(url);
            } catch (Exception e) {
                log.error("Error refreshing URL {}: {}", url.getUrl(), e.getMessage());
                // Schedule for retry
                scheduleRetry(url);
            }
        }
    }
    
    private void refreshUrl(UrlEntity urlEntity) throws IOException {
        String url = urlEntity.getUrl();
        String documentId = urlEntity.getDocumentId();
        
        // Retrieve the current document
        DocumentEntity oldDocument = documentRepository.findById(documentId).orElse(null);
        
        // Fetch current version of the page
        Document newDocument = crawler.fetchDocument(url);
        
        // Check if content has changed
        if (oldDocument != null && !hasChanged(oldDocument, newDocument)) {
            // No changes, just update last check time
            urlEntity.setLastChecked(new Date());
            urlRepository.save(urlEntity);
            return;
        }
        
        // Content has changed, update document
        String newContent = newDocument.getContent();
        String newChecksum = calculateChecksum(newContent);
        
        if (oldDocument != null) {
            // Update existing document
            oldDocument.setContent(newContent);
            oldDocument.setChecksum(newChecksum);
            oldDocument.setLastUpdated(new Date());
            documentRepository.save(oldDocument);
        } else {
            // Create new document
            DocumentEntity doc = new DocumentEntity();
            doc.setId(documentId);
            doc.setUrl(url);
            doc.setContent(newContent);
            doc.setChecksum(newChecksum);
            doc.setLastUpdated(new Date());
            documentRepository.save(doc);
        }
        
        // Update URL entity
        urlEntity.setLastChecked(new Date());
        urlEntity.setLastUpdated(new Date());
        urlRepository.save(urlEntity);
        
        // Re-index the document
        indexerService.reindexDocument(documentId, newContent, url);
    }
    
    private boolean hasChanged(DocumentEntity oldDoc, Document newDoc) {
        String oldChecksum = oldDoc.getChecksum();
        String newChecksum = calculateChecksum(newDoc.getContent());
        
        return !oldChecksum.equals(newChecksum);
    }
    
    private String calculateChecksum(String content) {
        return DigestUtils.md5Hex(content);
    }
    
    private void scheduleRetry(UrlEntity url) {
        // Schedule for retry with exponential backoff
        // ...
    }
}
```

**Performance & Scaling**
- Parallelized refreshing with worker pools.
- Prioritization based on expected change frequency.
- Batch processing to minimize overhead.

**Pitfalls & Best Practices**
- **Thundering herd**: All URLs refreshed at once causing load spikes.
- **Wasted resources**: Refreshing URLs that rarely change.
- **Change detection failures**: Missing subtle but important changes.
- Distribute refresh schedule throughout the day.
- Use adaptive refresh scheduling based on observed change patterns.
- Implement efficient change detection beyond simple checksums.

### Monitoring & Reliability

**Concept & Principles**
- Comprehensive monitoring of system health and performance.
- SLA tracking for availability and latency.
- Alerting for potential issues.
- Fault tolerance and recovery mechanisms.

**Real-World Usage**
- Latency tracking for query service.
- Availability monitoring across all services.
- Resource utilization tracking for capacity planning.

**Java Code Example (Monitoring Service)**

```java
@Service
public class MonitoringService {

    @Autowired
    private MeterRegistry meterRegistry;
    
    @Autowired
    private AlertService alertService;
    
    // Latency threshold for search queries (in ms)
    private static final double LATENCY_THRESHOLD = 200.0;
    
    // Availability window for SLA calculation (in minutes)
    private static final int AVAILABILITY_WINDOW = 60;
    
    // Target availability (99.99%)
    private static final double TARGET_AVAILABILITY = 99.99;
    
    // Counter for successful queries
    private Counter successCounter;
    
    // Counter for failed queries
    private Counter failureCounter;
    
    // Timer for query latency
    private Timer queryTimer;
    
    @PostConstruct
    public void init() {
        successCounter = Counter.builder("search.queries.success")
            .description("Number of successful search queries")
            .register(meterRegistry);
            
        failureCounter = Counter.builder("search.queries.failure")
            .description("Number of failed search queries")
            .register(meterRegistry);
            
        queryTimer = Timer.builder("search.queries.latency")
            .description("Search query latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .publishPercentileHistogram()
            .register(meterRegistry);
    }
    
    public void recordQuerySuccess(long latencyMs) {
        successCounter.increment();
        queryTimer.record(latencyMs, TimeUnit.MILLISECONDS);
        
        // Check if latency exceeds threshold
        if (latencyMs > LATENCY_THRESHOLD) {
            alertService.sendAlert(
                AlertLevel.WARNING,
                "Search query latency exceeded threshold",
                String.format("Query latency: %d ms (threshold: %d ms)", latencyMs, LATENCY_THRESHOLD)
            );
        }
    }
    
    public void recordQueryFailure(String errorMessage) {
        failureCounter.increment();
        
        alertService.sendAlert(
            AlertLevel.ERROR,
            "Search query failed",
            errorMessage
        );
    }
    
    @Scheduled(fixedRate = 60000) // Check every minute
    public void checkAvailability() {
        double availability = calculateAvailability();
        
        // If availability drops below target, send alert
        if (availability < TARGET_AVAILABILITY) {
            alertService.sendAlert(
                AlertLevel.CRITICAL,
                "System availability below SLA target",
                String.format("Current availability: %.2f%% (target: %.2f%%)", 
                             availability, TARGET_AVAILABILITY)
            );
        }
    }
    
    private double calculateAvailability() {
        double total = successCounter.count() + failureCounter.count();
        if (total == 0) {
            return 100.0; // No requests, assume 100% availability
        }
        
        return (successCounter.count() / total) * 100.0;
    }
    
    // Health check endpoint for the service
    public Map<String, Object> getHealthStatus() {
        Map<String, Object> status = new HashMap<>();
        
        status.put("service", "search-engine");
        status.put("status", isHealthy() ? "UP" : "DOWN");
        status.put("timestamp", new Date());
        
        // Latency metrics
        status.put("avg_latency_ms", queryTimer.mean(TimeUnit.MILLISECONDS));
        status.put("p95_latency_ms", queryTimer.percentile(0.95, TimeUnit.MILLISECONDS));
        status.put("p99_latency_ms", queryTimer.percentile(0.99, TimeUnit.MILLISECONDS));
        
        // Request counts
        status.put("success_count", successCounter.count());
        status.put("failure_count", failureCounter.count());
        
        // Availability
        status.put("availability", calculateAvailability());
        
        return status;
    }
    
    private boolean isHealthy() {
        // Check if critical dependencies are available
        boolean indexerHealthy = checkIndexerHealth();
        boolean storageHealthy = checkStorageHealth();
        boolean cacheHealthy = checkCacheHealth();
        
        return indexerHealthy && storageHealthy && cacheHealthy;
    }
    
    private boolean checkIndexerHealth() {
        // Implementation to check indexer health
        // ...
        return true;
    }
    
    private boolean checkStorageHealth() {
        // Implementation to check storage health
        // ...
        return true;
    }
    
    private boolean checkCacheHealth() {
        // Implementation to check cache health
        // ...
        return true;
    }
}
```

**Performance & Scaling**
- Distributed monitoring with central aggregation.
- Time-series databases for metric storage.
- Dashboards for visualization and trend analysis.

**Pitfalls & Best Practices**
- **Alert fatigue**: Too many alerts causing desensitization.
- **Blind spots**: Missing critical metrics or components.
- **Resource overhead**: Excessive monitoring impacting performance.
- Implement intelligent alerting with proper thresholds.
- Focus on user-impacting metrics first (latency, errors, availability).
- Use automated remediation where possible to reduce manual intervention.

## Performance & Scalability Considerations

1. **Query Processing Optimization**:
   - Cache frequent queries and results.
   - Optimize stemming and preprocessing.
   - Use in-memory processing for critical path operations.

2. **Index Design**:
   - Shard indices by term prefixes for parallel search.
   - Optimize posting list storage and compression.
   - Use memory-mapped files for fast access.

3. **Distributed Architecture**:
   - Stateless query services for easy horizontal scaling.
   - Separate read and write paths with different scaling characteristics.
   - Geographic distribution for lower global latency.

4. **Data Partitioning**:
   - Partition crawler by domain for politeness.
   - Shard index by term ranges to distribute query load.
   - Distribute document storage by URL hash.

5. **Caching Strategy**:
   - Multi-level caching (in-memory, Redis, CDN).
   - Cache warming for predictable workloads.
   - Tiered storage with hot/warm/cold data separation.

6. **Asynchronous Processing**:
   - Background crawling and indexing.
   - Queue-based job processing for URL refreshes.
   - Event-driven architecture for system coordination.

7. **Resource Management**:
   - Adaptive allocation based on query patterns.
   - Auto-scaling for variable load.
   - Throttling and rate limiting to prevent overload.

8. **Load Distribution**:
   - Consistent hashing for balanced routing.
   - Global load balancing for geographic distribution.
   - Query routing based on term affinity.

## Common Pitfalls & How to Avoid Them

1. **Crawler Politeness**: Overly aggressive crawling causing site bans or network issues.
   - Implement per-domain rate limiting.
   - Respect robots.txt and crawl-delay directives.
   - Use exponential backoff for retries.

2. **Index Growth**: Unbounded index growth consuming excessive storage.
   - Implement term pruning for rare or meaningless terms.
   - Use efficient compression techniques.
   - Consider time-based indexing with archival policies.

3. **Query Latency**: Slow responses causing SLA violations.
   - Optimize critical path processing.
   - Implement aggressive caching for common queries.
   - Pre-compute results for predictable queries.

4. **Stale Results**: Infrequent refreshing causing outdated search results.
   - Prioritize refresh based on content volatility.
   - Implement efficient change detection.
   - Consider real-time notification mechanisms for important sources.

5. **Cache Invalidation**: Difficulty in knowing when to refresh cached results.
   - Use time-based expiration as a safety net.
   - Implement event-based invalidation when index changes.
   - Consider probabilistic early expiration for critical terms.

6. **Stop Word Filtering**: Excessive filtering causing relevant content to be missed.
   - Carefully curate stop word lists.
   - Consider context-sensitive stop word handling.
   - Allow exceptions for quoted phrases.

7. **Stemming Errors**: Over-stemming or under-stemming affecting result quality.
   - Use proven stemming algorithms like Porter or Snowball.
   - Maintain exceptions for irregular words.
   - Consider machine learning approaches for improved accuracy.

8. **Monitoring Blind Spots**: Missing critical indicators leading to undetected issues.
   - Implement comprehensive monitoring covering all components.
   - Focus on user-perceived metrics (latency, relevance).
   - Use synthetic queries to test end-to-end functionality.

## Best Practices & Maintenance

1. **Continuous Integration/Deployment**:
   - Automated testing for all components.
   - Canary deployments to detect issues early.
   - Blue-green deployments for zero-downtime updates.

2. **Index Maintenance**:
   - Regular optimization of index structures.
   - Monitoring for index bloat or inefficiencies.
   - Scheduled reindexing for major schema changes.

3. **Crawler Management**:
   - Adaptive crawl scheduling based on change frequency.
   - Blacklisting of problematic or spam domains.
   - Regular review of crawl patterns and efficiency.

4. **Performance Tuning**:
   - Regular load testing to identify bottlenecks.
   - Profiling of critical path operations.
   - Optimization based on real query patterns.

5. **Monitoring & Alerting**:
   - Comprehensive dashboards for system health.
   - Alerting for SLA violations or anomalies.
   - Historical trending for capacity planning.

6. **Security Practices**:
   - Regular security audits and vulnerability scanning.
   - Proper authentication for administrative functions.
   - Protection against injection attacks in query processing.

7. **Disaster Recovery**:
   - Regular backups of critical data.
   - Cross-region replication for high availability.
   - Documented recovery procedures with regular drills.

8. **Documentation & Knowledge Sharing**:
   - Clear documentation of system architecture and components.
   - Runbooks for common operational tasks.
   - Knowledge sharing sessions for team members.

9. **Query Analysis**:
   - Regular review of slow queries for optimization.
   - Analysis of failed queries for improvement.
   - Tracking of user behavior for relevance enhancement.

10. **Capacity Planning**:
    - Regular review of resource utilization.
    - Predictive scaling based on historical patterns.
    - Budgeting for growth in data volume and query traffic.

## How to Discuss in a Principal Engineer Interview

1. **Start with Requirements**:
   - Emphasize the 99.99% availability and 200ms latency requirements.
   - Discuss the single-keyword search model and "good match" criteria.
   - Highlight the daily refresh requirement and its implications.

2. **System Architecture Overview**:
   - Present the high-level components and their interactions.
   - Explain the separation of concerns between services.
   - Discuss the data flow from crawling to querying.

3. **Scaling Approach**:
   - Detail horizontal scaling strategies for each component.
   - Explain how the system handles growth in both data and query volume.
   - Discuss the balance between performance and resource efficiency.

4. **Technical Decisions & Trade-offs**:
   - Justify technology choices (e.g., ElasticSearch vs. custom indexing).
   - Explain trade-offs between latency, consistency, and resource usage.
   - Discuss the balance between refresh frequency and system load.

5. **Performance Optimizations**:
   - Detail caching strategies for common queries.
   - Explain index optimizations for fast retrieval.
   - Discuss distributed processing for crawling and indexing.

6. **Resilience & Availability**:
   - Explain how the system achieves 99.99% availability.
   - Discuss failure handling and recovery mechanisms.
   - Detail redundancy and replication strategies.

7. **Monitoring & Maintenance**:
   - Describe the metrics tracked for SLA compliance.
   - Explain approaches to index maintenance and optimization.
   - Discuss capacity planning for growth.

8. **Operational Aspects**:
   - Detail deployment strategies and update procedures.
   - Explain debugging and troubleshooting approaches.
   - Discuss day-to-day operational concerns.

9. **Edge Cases & Challenges**:
   - Address handling of very common or very rare terms.
   - Discuss approaches to crawler politeness and rate limiting.
   - Explain handling of system overload scenarios.

10. **Future Improvements**:
    - Discuss potential extensions beyond single-keyword search.
    - Explain how relevancy ranking could be implemented.
    - Detail approaches to scaling for larger web coverage.

## Conclusion

This design outlines a **robust, scalable search engine system** capable of meeting the specified functional and non-functional requirements. The architecture leverages **distributed processing**, **efficient indexing**, and **multi-level caching** to deliver search results with sub-200ms latency while maintaining 99.99% availability.

The system's modular design with clearly separated services enables **independent scaling** of components based on their specific resource needs. The **crawler service** efficiently discovers and fetches web content while respecting site policies. The **indexer service** processes this content with proper stemming and stop word removal to build an optimized inverted index. The **query service** provides fast lookup with extensive caching to meet the strict latency requirements.

By implementing a **daily URL refresh** mechanism with efficient change detection, the system ensures search results remain current without excessive resource usage. The focus on **single-keyword queries** with "good match" optimization for URL keywords allows for a streamlined implementation that still delivers relevant results.

The comprehensive **monitoring and alerting** system ensures compliance with SLA requirements and enables proactive identification of potential issues. Through careful attention to common pitfalls like crawler politeness, index growth, and cache invalidation, the design provides a foundation for a reliable, maintainable search engine that can grow and evolve over time.
