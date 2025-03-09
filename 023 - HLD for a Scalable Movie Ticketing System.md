# High-Level Design for a Scalable Movie Ticketing System

## Table of Contents
- [Overview & Core Requirements](#overview--core-requirements)
- [Major Microservices & Responsibilities](#major-microservices--responsibilities)
  - [User Service](#user-service)
  - [Movie Service](#movie-service)
  - [Theatre Service](#theatre-service)
  - [Showtime Service](#showtime-service)
  - [Seat Inventory Service](#seat-inventory-service)
  - [Booking Service](#booking-service)
  - [Payment Service](#payment-service)
  - [Standby Service](#standby-service)
  - [Notification Service](#notification-service)
  - [Search Service](#search-service)
  - [Analytics Service](#analytics-service)
  - [Archival Service](#archival-service)
- [High-Level Architecture & Data Flow](#high-level-architecture--data-flow)
- [Key System Design Topics & Deep Dives](#key-system-design-topics--deep-dives)
  - [Database Design](#database-design)
  - [Concurrency Control & Seat Locking](#concurrency-control--seat-locking)
  - [Payment Processing & Security](#payment-processing--security)
  - [Caching Strategy](#caching-strategy)
  - [Standby & Waitlist System](#standby--waitlist-system)
  - [Notification System](#notification-system)
  - [Search & Filtering](#search--filtering)
  - [Event-Driven Architecture (Kafka)](#event-driven-architecture-kafka)
  - [Data Retention & Archiving](#data-retention--archiving)
- [Performance & Scalability Considerations](#performance--scalability-considerations)
- [Common Pitfalls & How to Avoid Them](#common-pitfalls--how-to-avoid-them)
- [Best Practices & Maintenance](#best-practices--maintenance)
- [How to Discuss in a Principal Engineer Interview](#how-to-discuss-in-a-principal-engineer-interview)
- [Conclusion](#conclusion)

## Overview & Core Requirements

We want to build a **movie ticketing system** where users can:

1. **Browse movies and theatres** filtered by city, location, or other criteria.
2. **View available showtimes** for chosen films.
3. **Book tickets** for selected showtimes.
4. **Stand by for seats** that might become available due to cancellations.
5. **Purchase additional seats** after an initial booking.
6. Receive **notifications** for booking confirmations, standby alerts, and cancellations.

### Additional Requirements

- **Secure financial transactions** for payment processing.
- **Data retention** for 5 years to comply with regulations.
- **Concurrency control** to prevent double-booking of seats.
- **High availability** during peak booking periods (new releases, weekends).
- **Low latency** for searching and browsing movie listings.
- **Scalable architecture** to handle traffic spikes (blockbuster releases).
- **Location-based filtering** for movies and theatres.
- **User-friendly seat selection** interface.
- **Discount and promotion** support for special offers.
- **Multiple payment methods** integration.

## Major Microservices & Responsibilities

### User Service
- **Responsibilities**: Manages user accounts, profiles, preferences, authentication/authorization.
- **Data Store**: Relational DB (e.g., MySQL/PostgreSQL) for ACID compliance and transactional integrity.
- **Notes**: 
  - Stores user viewing history and preferences for personalization.
  - Integrates with OAuth or JWT-based security.
  - Handles user loyalty programs or subscription plans.

### Movie Service
- **Responsibilities**: Maintains comprehensive movie data including titles, descriptions, genres, cast, posters, trailers, and ratings.
- **Data Store**: MongoDB for flexible schema handling varied movie metadata.
- **Notes**:
  - Integrates with external movie data providers (TMDB, IMDB).
  - Publishes movie update events to Kafka when new movies are added or details change.
  - Supports rich media content like trailers and multiple poster images.

### Theatre Service
- **Responsibilities**: Manages theatre locations, amenities, seating layouts, and operational hours.
- **Data Store**: PostgreSQL for structured theatre data and complex spatial queries.
- **Notes**:
  - Stores geospatial data for location-based filtering.
  - Maintains different auditorium layouts within each theatre.
  - Tracks amenities like IMAX, Dolby Atmos, recliner seats, etc.

### Showtime Service
- **Responsibilities**: Manages movie showtimes, connecting movies with specific theatre screens at particular times.
- **Data Store**: PostgreSQL for relational integrity across movies, theatres, and times.
- **Notes**:
  - Links movies, theatres, and time slots.
  - Handles special shows like premieres, marathons, or limited screenings.
  - Manages dynamic pricing based on time, day, or demand.

### Seat Inventory Service
- **Responsibilities**: Tracks available, reserved, and booked seats for each showtime.
- **Data Store**: 
  - Redis for real-time seat status and locking mechanism.
  - PostgreSQL for persistent storage and transaction history.
- **Notes**:
  - Implements optimistic or pessimistic locking to prevent double bookings.
  - Handles seat reservation timeouts (typically 5-10 minutes).
  - Maintains seat maps and status (available, reserved, booked).

### Booking Service
- **Responsibilities**: Manages the entire booking process, from seat selection to confirmation.
- **Data Store**: PostgreSQL for ACID compliance on critical booking transactions.
- **Notes**:
  - Orchestrates between Seat Inventory, Payment, and Notification services.
  - Implements saga pattern for distributed transactions.
  - Handles booking modifications and cancellations.
  - Publishes "BookingCreated", "BookingConfirmed", or "BookingCancelled" events.

### Payment Service
- **Responsibilities**: Processes payments for bookings, handles refunds for cancellations.
- **Data Store**: PostgreSQL for financial transaction records.
- **Notes**:
  - Integrates with external payment gateways (Stripe, PayPal, etc.).
  - Implements PCI-DSS compliance for credit card handling.
  - Manages discount codes and promotional offers.
  - Uses idempotency keys to prevent double charges.

### Standby Service
- **Responsibilities**: Manages waitlists for fully booked showtimes, notifies users when seats become available.
- **Data Store**:
  - Redis for real-time queue management.
  - PostgreSQL for persistent standby records.
- **Notes**:
  - Prioritizes waitlist based on join time or user loyalty status.
  - Listens to "BookingCancelled" events.
  - Time-limited offers for newly available seats (typically 5-15 minutes).

### Notification Service
- **Responsibilities**: Sends various notifications (booking confirmations, standby alerts, reminders) via email, SMS, or push.
- **Data Flow**: Consumes events from Kafka for various booking activities.
- **Notes**:
  - Sends booking confirmations with QR codes/barcodes.
  - Delivers standby notifications when seats become available.
  - Sends show reminders before screening time.
  - Handles marketing communications and promotions.

### Search Service
- **Responsibilities**: Provides advanced search and filtering for movies and theatres.
- **Data Store**: Elasticsearch for full-text search and complex filtering.
- **Notes**:
  - Supports filtering by location, genre, language, format, etc.
  - Enables text search across movie titles, actors, directors.
  - Integrates with recommendation systems for personalized results.

### Analytics Service
- **Responsibilities**: Collects booking data for business insights, popular movies, peak booking times.
- **Data Store**: 
  - Data Warehouse (Snowflake, Redshift) for historical analysis.
  - Kafka Streams or Flink for real-time analytics.
- **Notes**:
  - Provides insights on booking patterns, popular movies, and revenue trends.
  - Feeds data to recommendation systems.
  - Generates reports for theatre management and film distributors.

### Archival Service
- **Responsibilities**: Archives older booking data for regulatory compliance (5-year retention).
- **Data Store**: Cloud storage (S3, Google Cloud Storage) for cost-effective long-term storage.
- **Notes**:
  - Implements data lifecycle policies.
  - Provides retrieval APIs for compliance or audit purposes.
  - Maintains data in a GDPR/CCPA compliant manner.

## High-Level Architecture & Data Flow

1. **Movie Browsing Flow**
   - User accesses application through web or mobile client.
   - **Search Service** provides movie and theatre listings based on location and filters.
   - **Movie Service** and **Theatre Service** provide detailed information.
   - **Showtime Service** displays available screening times.

2. **Booking Flow**
   - User selects a showtime and seats.
   - **Seat Inventory Service** temporarily locks selected seats (5-10 minute reservation).
   - User proceeds to payment through the **Booking Service**.
   - **Payment Service** processes the transaction.
   - On successful payment, **Booking Service** confirms the reservation.
   - **Seat Inventory Service** updates seat status to "booked".
   - **Notification Service** sends booking confirmation.
   - Events are published to Kafka for analytics and other processing.

3. **Standby Flow**
   - For sold-out showtimes, users can join a standby list via **Standby Service**.
   - When a booking is cancelled, a "BookingCancelled" event is published.
   - **Standby Service** consumes this event and notifies the next user in line.
   - The notified user has a limited time window to book the newly available seats.
   - If they don't respond in time, the next user in line is notified.

4. **Additional Seat Purchase Flow**
   - User requests to add seats to an existing booking.
   - **Booking Service** checks adjacent seat availability via **Seat Inventory Service**.
   - If available, a modified booking process is initiated for the additional seats.
   - New seats are associated with the original booking ID.
   - Payment is processed only for the additional seats.

5. **Cancellation Flow**
   - User requests booking cancellation.
   - **Booking Service** validates cancellation policy (time limits, refund eligibility).
   - If eligible, **Payment Service** processes the refund.
   - **Seat Inventory Service** releases the seats back to inventory.
   - A "BookingCancelled" event is published to Kafka.
   - **Standby Service** is notified of newly available seats.

## Key System Design Topics & Deep Dives

### Database Design

**Concept & Principles**
- Relational databases for structured, transactional data (bookings, payments).
- NoSQL for flexible schema data (movie details, user preferences).
- Caching for high-read, relatively static data (movie info, theatre layouts).
- Proper indexing for location-based and time-based queries.

**Key Database Schemas**

```sql
-- Users table (PostgreSQL)
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    phone VARCHAR(20),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_login TIMESTAMP WITH TIME ZONE
);

-- Theatres table (PostgreSQL)
CREATE TABLE theatres (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    address VARCHAR(255) NOT NULL,
    city VARCHAR(50) NOT NULL,
    state VARCHAR(50),
    zip_code VARCHAR(20),
    latitude DECIMAL(10, 8),
    longitude DECIMAL(11, 8),
    phone VARCHAR(20),
    has_imax BOOLEAN DEFAULT FALSE,
    has_dolby BOOLEAN DEFAULT FALSE,
    has_recliners BOOLEAN DEFAULT FALSE
);

-- Screens table (PostgreSQL)
CREATE TABLE screens (
    id BIGSERIAL PRIMARY KEY,
    theatre_id BIGINT REFERENCES theatres(id),
    name VARCHAR(50) NOT NULL,
    capacity INT NOT NULL,
    screen_type VARCHAR(20), -- Regular, IMAX, Dolby, etc.
    seating_layout JSONB -- Store seating layout as JSON
);

-- Showtimes table (PostgreSQL)
CREATE TABLE showtimes (
    id BIGSERIAL PRIMARY KEY,
    movie_id VARCHAR(24) NOT NULL, -- Reference to MongoDB movie ID
    screen_id BIGINT REFERENCES screens(id),
    start_time TIMESTAMP WITH TIME ZONE NOT NULL,
    end_time TIMESTAMP WITH TIME ZONE NOT NULL,
    price_base DECIMAL(6, 2) NOT NULL,
    price_premium DECIMAL(6, 2), -- For premium seats if applicable
    is_3d BOOLEAN DEFAULT FALSE,
    language VARCHAR(20), -- Original, Dubbed, etc.
    subtitle_language VARCHAR(20),
    status VARCHAR(20) DEFAULT 'SCHEDULED' -- SCHEDULED, CANCELLED, COMPLETED
);

-- Seats table (PostgreSQL)
CREATE TABLE seats (
    id BIGSERIAL PRIMARY KEY,
    screen_id BIGINT REFERENCES screens(id),
    row_label VARCHAR(5) NOT NULL,
    seat_number VARCHAR(5) NOT NULL,
    seat_type VARCHAR(20) DEFAULT 'REGULAR', -- REGULAR, PREMIUM, HANDICAP, etc.
    UNIQUE(screen_id, row_label, seat_number)
);

-- Bookings table (PostgreSQL)
CREATE TABLE bookings (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    showtime_id BIGINT REFERENCES showtimes(id),
    booking_time TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    status VARCHAR(20) NOT NULL, -- PENDING, CONFIRMED, CANCELLED
    total_amount DECIMAL(10, 2) NOT NULL,
    payment_id VARCHAR(100), -- Reference to payment gateway transaction
    confirmation_code VARCHAR(20) UNIQUE NOT NULL
);

-- Booked Seats table (PostgreSQL)
CREATE TABLE booked_seats (
    id BIGSERIAL PRIMARY KEY,
    booking_id BIGINT REFERENCES bookings(id),
    seat_id BIGINT REFERENCES seats(id),
    price DECIMAL(6, 2) NOT NULL,
    UNIQUE(booking_id, seat_id)
);

-- Standby Requests table (PostgreSQL)
CREATE TABLE standby_requests (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    showtime_id BIGINT REFERENCES showtimes(id),
    seat_count INT NOT NULL,
    request_time TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    status VARCHAR(20) DEFAULT 'WAITING', -- WAITING, NOTIFIED, FULFILLED, EXPIRED
    expires_at TIMESTAMP WITH TIME ZONE, -- When notified about availability
    notification_sent BOOLEAN DEFAULT FALSE
);

-- Payments table (PostgreSQL)
CREATE TABLE payments (
    id BIGSERIAL PRIMARY KEY,
    booking_id BIGINT REFERENCES bookings(id),
    amount DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    payment_method VARCHAR(50) NOT NULL, -- CREDIT_CARD, PAYPAL, etc.
    transaction_id VARCHAR(100), -- External payment gateway ID
    status VARCHAR(20) NOT NULL, -- PENDING, COMPLETED, FAILED, REFUNDED
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

**MongoDB Schema for Movies**

```javascript
// Movies collection (MongoDB)
{
  "_id": ObjectId(),
  "title": "Inception",
  "original_title": "Inception",
  "runtime_minutes": 148,
  "release_date": ISODate("2010-07-16"),
  "mpaa_rating": "PG-13",
  "plot_summary": "A thief who steals corporate secrets...",
  "genres": ["Action", "Sci-Fi", "Thriller"],
  "language": "English",
  "country": "USA",
  "director": "Christopher Nolan",
  "writers": ["Christopher Nolan"],
  "cast": [
    {
      "name": "Leonardo DiCaprio",
      "character": "Dom Cobb"
    },
    {
      "name": "Joseph Gordon-Levitt",
      "character": "Arthur"
    }
  ],
  "poster_url": "https://cdn.example.com/posters/inception.jpg",
  "backdrop_url": "https://cdn.example.com/backdrops/inception.jpg",
  "trailer_url": "https://youtube.com/watch?v=abcdefg",
  "imdb_id": "tt1375666",
  "imdb_rating": 8.8,
  "created_at": ISODate("2020-01-01T00:00:00Z"),
  "updated_at": ISODate("2020-01-01T00:00:00Z"),
  "is_active": true
}
```

**Performance & Scaling**
- Horizontal sharding for booking data based on showtime date or theatre location.
- Vertical partitioning to separate hot tables (current bookings) from cold tables (historical data).
- Read replicas for movie and theatre data that's frequently queried but rarely updated.
- Geographic distribution of databases to reduce latency for location-based searches.

**Pitfalls & Best Practices**
- Ensure proper indexing on frequently queried fields (showtime_id, user_id, city).
- Implement connection pooling to handle peak load efficiently.
- Use database transactions for critical operations (seat booking, payment).
- Plan data archiving strategy from the beginning to meet 5-year retention requirements.
- Consider data partitioning by date for efficient archiving of old bookings.

### Concurrency Control & Seat Locking

**Concept & Principles**
- Preventing double bookings through pessimistic or optimistic locking.
- Handling seat reservation timeouts.
- Ensuring atomicity across distributed services.

**Implementation Approaches**

1. **Pessimistic Locking with Redis**
   ```java
   @Service
   public class SeatLockService {
       private final RedisTemplate<String, String> redisTemplate;
       private final static long LOCK_TIMEOUT_SECONDS = 600; // 10 minutes
       
       public SeatLockService(RedisTemplate<String, String> redisTemplate) {
           this.redisTemplate = redisTemplate;
       }
       
       public boolean lockSeats(Long showtimeId, List<Long> seatIds, String userId) {
           String lockKey = "lock:showtime:" + showtimeId;
           
           // Try to acquire lock
           Boolean lockAcquired = redisTemplate.opsForValue()
               .setIfAbsent(lockKey, "LOCKED", Duration.ofSeconds(10));
           
           if (Boolean.TRUE.equals(lockAcquired)) {
               try {
                   // Check if any seat is already locked
                   for (Long seatId : seatIds) {
                       String seatKey = "seat:" + showtimeId + ":" + seatId;
                       String currentHolder = redisTemplate.opsForValue().get(seatKey);
                       
                       if (currentHolder != null && !currentHolder.equals(userId)) {
                           // Seat already locked by someone else
                           return false;
                       }
                   }
                   
                   // Lock all seats
                   for (Long seatId : seatIds) {
                       String seatKey = "seat:" + showtimeId + ":" + seatId;
                       redisTemplate.opsForValue().set(
                           seatKey, userId, Duration.ofSeconds(LOCK_TIMEOUT_SECONDS));
                   }
                   
                   return true;
               } finally {
                   // Release lock
                   redisTemplate.delete(lockKey);
               }
           }
           
           return false;
       }
       
       public void releaseSeats(Long showtimeId, List<Long> seatIds, String userId) {
           for (Long seatId : seatIds) {
               String seatKey = "seat:" + showtimeId + ":" + seatId;
               String currentHolder = redisTemplate.opsForValue().get(seatKey);
               
               // Only release if this user holds the lock
               if (userId.equals(currentHolder)) {
                   redisTemplate.delete(seatKey);
               }
           }
       }
   }
   ```

2. **Optimistic Locking with Version Control**
   ```java
   @Entity
   @Table(name = "seat_inventory")
   public class SeatInventory {
       @Id
       private Long id;
       
       @Column(name = "showtime_id")
       private Long showtimeId;
       
       @Column(name = "seat_id")
       private Long seatId;
       
       @Column(name = "status")
       private String status; // AVAILABLE, RESERVED, BOOKED
       
       @Column(name = "reserved_by")
       private String reservedBy;
       
       @Column(name = "reserved_until")
       private LocalDateTime reservedUntil;
       
       @Version
       private Long version; // For optimistic locking
       
       // Getters and setters
   }
   
   @Service
   public class SeatInventoryService {
       @Autowired
       private SeatInventoryRepository repository;
       
       @Transactional
       public boolean reserveSeats(Long showtimeId, List<Long> seatIds, String userId) {
           List<SeatInventory> seats = repository.findByShowtimeIdAndSeatIdIn(
               showtimeId, seatIds);
               
           if (seats.size() != seatIds.size()) {
               return false; // Not all seats found
           }
           
           LocalDateTime now = LocalDateTime.now();
           LocalDateTime expiry = now.plusMinutes(10);
           
           for (SeatInventory seat : seats) {
               if (!seat.getStatus().equals("AVAILABLE") && 
                   !(seat.getStatus().equals("RESERVED") && 
                     seat.getReservedUntil().isBefore(now))) {
                   return false; // Seat not available
               }
               
               seat.setStatus("RESERVED");
               seat.setReservedBy(userId);
               seat.setReservedUntil(expiry);
           }
           
           try {
               repository.saveAll(seats);
               return true;
           } catch (OptimisticLockingFailureException e) {
               return false; // Someone else modified the seats concurrently
           }
       }
   }
   ```

**Scheduled Job for Expiring Reservations**
```java
@Component
public class ReservationExpiryJob {
    
    @Autowired
    private SeatInventoryRepository repository;
    
    @Scheduled(fixedRate = 60000) // Run every minute
    public void releaseExpiredReservations() {
        LocalDateTime now = LocalDateTime.now();
        List<SeatInventory> expiredReservations = repository
            .findByStatusAndReservedUntilBefore("RESERVED", now);
            
        for (SeatInventory seat : expiredReservations) {
            seat.setStatus("AVAILABLE");
            seat.setReservedBy(null);
            seat.setReservedUntil(null);
        }
        
        repository.saveAll(expiredReservations);
    }
}
```

**Performance & Scaling**
- Redis distributed locking handles high concurrency efficiently.
- Use Redis Cluster for horizontal scaling of locking mechanism.
- Database-level row locking provides strong consistency guarantees.
- Scheduled jobs to clean up expired reservations prevent deadlocks.

**Pitfalls & Best Practices**
- Avoid long locks that could impact user experience.
- Implement deadlock detection and resolution strategies.
- Use appropriate timeout policies based on user behavior analysis.
- Gracefully handle reservation expirations with clear user feedback.
- Consider implementing a "heartbeat" mechanism for long reservation sessions.

### Payment Processing & Security

**Concept & Principles**
- Secure handling of payment information.
- PCI-DSS compliance for credit card processing.
- Integration with third-party payment gateways.
- Handling payment failures and retries.
- Supporting multiple payment methods.

**Implementation Example (with Stripe)**

```java
@Service
public class PaymentService {
    
    @Value("${stripe.api.key}")
    private String stripeApiKey;
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    @Autowired
    private BookingService bookingService;
    
    public PaymentResult processPayment(Long bookingId, PaymentRequest request) {
        // Retrieve booking details
        Booking booking = bookingService.findById(bookingId);
        if (booking == null) {
            throw new EntityNotFoundException("Booking not found");
        }
        
        // Initialize Stripe
        Stripe.apiKey = stripeApiKey;
        
        // Generate idempotency key to prevent double charging
        String idempotencyKey = UUID.randomUUID().toString();
        
        try {
            // Create payment in our system as PENDING
            Payment payment = new Payment();
            payment.setBookingId(bookingId);
            payment.setAmount(booking.getTotalAmount());
            payment.setCurrency("USD");
            payment.setPaymentMethod(request.getPaymentMethod());
            payment.setStatus("PENDING");
            payment = paymentRepository.save(payment);
            
            // Process with Stripe
            Map<String, Object> params = new HashMap<>();
            params.put("amount", (int)(booking.getTotalAmount() * 100)); // Stripe uses cents
            params.put("currency", "usd");
            params.put("source", request.getTokenId());
            params.put("description", "Booking #" + booking.getConfirmationCode());
            
            RequestOptions options = RequestOptions.builder()
                .setIdempotencyKey(idempotencyKey)
                .build();
            
            Charge charge = Charge.create(params, options);
            
            // Update payment record
            payment.setTransactionId(charge.getId());
            payment.setStatus(charge.getStatus().equals("succeeded") ? "COMPLETED" : "FAILED");
            payment.setUpdatedAt(LocalDateTime.now());
            paymentRepository.save(payment);
            
            if (charge.getStatus().equals("succeeded")) {
                bookingService.confirmBooking(bookingId);
                return new PaymentResult(true, "Payment successful", charge.getId());
            } else {
                return new PaymentResult(false, "Payment failed: " + charge.getFailureMessage(), charge.getId());
            }
            
        } catch (StripeException e) {
            // Handle Stripe-specific exceptions
            return new PaymentResult(false, "Payment processing error: " + e.getMessage(), null);
        } catch (Exception e) {
            // Handle general exceptions
            return new PaymentResult(false, "System error during payment processing", null);
        }
    }
    
    public PaymentResult processRefund(Long bookingId) {
        // Similar implementation for refunds
        // ...
    }
}
```

**Security Considerations**
- Never store raw credit card data; use tokenization.
- Implement HTTPS for all payment-related endpoints.
- Apply proper authentication and authorization for payment APIs.
- Log all payment activities for audit purposes.
- Encrypt sensitive payment data at rest.

**Performance & Scaling**
- Asynchronous processing for non-critical payment operations (e.g., refunds).
- Retry mechanisms with exponential backoff for failed payments.
- Rate limiting to prevent payment fraud attempts.

**Pitfalls & Best Practices**
- Always use idempotency keys to prevent double charges during network issues.
- Implement comprehensive error handling for payment gateway integration.
- Provide clear feedback to users about payment status.
- Set up monitoring and alerting for payment failures.
- Conduct regular security audits of payment flows.
- Implement fraud detection mechanisms.

### Caching Strategy

**Concept & Principles**
- Multi-level caching to reduce database load.
- Different caching strategies for different data types.
- Cache invalidation patterns.
- TTL (Time-to-Live) policies based on data volatility.

**Implementation Example (Redis + Spring Cache)**

```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        // Define cache configurations with different TTLs
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(24))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new JdkSerializationRedisSerializer()));
        
        // Configure specific caches with different TTLs
        Map<String, RedisCacheConfiguration> cacheConfigs = new HashMap<>();
        
        // Movie data - rarely changes
        cacheConfigs.put("movies", defaultConfig.entryTtl(Duration.ofDays(7)));
        
        // Theatre data - rarely changes
        cacheConfigs.put("theatres", defaultConfig.entryTtl(Duration.ofDays(3)));
        
        // Showtimes - more volatile
        cacheConfigs.put("showtimes", defaultConfig.entryTtl(Duration.ofHours(6)));
        
        // Seat inventory - highly volatile
        cacheConfigs.put("seatInventory", defaultConfig.entryTtl(Duration.ofMinutes(1)));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigs)
            .transactionAware()
            .build();
    }
}

@Service
public class MovieService {
    
    @Autowired
    private MovieRepository movieRepository;
    
    @Cacheable(value = "movies", key = "#id")
    public Movie findById(String id) {
        return movieRepository.findById(id).orElse(null);
    }
    
    @Cacheable(value = "movies", key = "'nowPlaying:' + #city")
    public List<Movie> findNowPlayingInCity(String city) {
        return movieRepository.findNowPlayingInCity(city);
    }
    
    @CachePut(value = "movies", key = "#movie.id")
    public Movie updateMovie(Movie movie) {
        return movieRepository.save(movie);
    }
    
    @CacheEvict(value = "movies", key = "#id")
    public void deleteMovie(String id) {
        movieRepository.deleteById(id);
    }
    
    // Clear specific cache when data changes significantly
    @CacheEvict(value = "movies", key = "'nowPlaying:' + #city")
    public void refreshNowPlayingCache(String city) {
        // Just evicts the cache
    }
}
```

**Cache Types and Use Cases**

1. **Object Cache (Redis)**
   - Movie details, theatre information, showtimes
   - User profiles and preferences
   - Recently viewed movies

2. **HTTP Cache (CDN)**
   - Static assets (images, CSS, JavaScript)
   - Movie posters and promotional media
   - Theatre maps and seating layouts

3. **Local Cache (In-Memory)**
   - Configuration data
   - Common lookup values
   - Rate limiting counters

**Performance & Scaling**
- Redis Cluster for distributed caching.
- CDN for global content delivery.
- Intelligent TTL based on data change frequency.
- Cache warming for predictable high-traffic periods (weekend evenings, new releases).

**Pitfalls & Best Practices**
- Implement proper cache invalidation to prevent stale data.
- Don't cache highly volatile data (e.g., real-time seat availability).
- Add cache hit/miss monitoring to optimize caching strategy.
- Use cache versioning to enable atomic cache updates.
- Implement fallback mechanisms for cache failures.

### Standby & Waitlist System

**Concept & Principles**
- Managing waitlists for fully booked showtimes.
- Prioritization strategies for standby requests.
- Notification mechanism when seats become available.
- Time-limited offers for newly available seats.

**Implementation Example**

```java
@Service
public class StandbyService {
    
    @Autowired
    private StandbyRepository standbyRepository;
    
    @Autowired
    private SeatInventoryService seatInventoryService;
    
    @Autowired
    private NotificationService notificationService;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    // Add user to standby list
    public StandbyRequest addToStandby(Long userId, Long showtimeId, int seatCount) {
        StandbyRequest request = new StandbyRequest();
        request.setUserId(userId);
        request.setShowtimeId(showtimeId);
        request.setSeatCount(seatCount);
        request.setRequestTime(LocalDateTime.now());
        request.setStatus("WAITING");
        
        return standbyRepository.save(request);
    }
    
    // Process cancellations and notify standby users
    @KafkaListener(topics = "booking-cancelled")
    public void processCancellation(BookingCancelledEvent event) {
        Long showtimeId = event.getShowtimeId();
        int availableSeats = event.getCancelledSeatCount();
        
        // Find eligible standby requests in order of request time
        List<StandbyRequest> standbyRequests = standbyRepository.findByShowtimeIdAndStatusOrderByRequestTimeAsc(
            showtimeId, "WAITING");
            
        for (StandbyRequest request : standbyRequests) {
            // Skip if we don't have enough seats for this request
            if (request.getSeatCount() > availableSeats) {
                continue;
            }
            
            // Mark as notified and set expiration time (15 minutes from now)
            request.setStatus("NOTIFIED");
            request.setNotificationSent(true);
            request.setExpiresAt(LocalDateTime.now().plusMinutes(15));
            standbyRepository.save(request);
            
            // Send notification
            notificationService.sendStandbyAlert(
                request.getUserId(), 
                showtimeId, 
                request.getSeatCount(),
                request.getExpiresAt()
            );
            
            // Reserve these seats temporarily for the standby user
            seatInventoryService.reserveForStandby(
                showtimeId, 
                request.getSeatCount(), 
                request.getUserId().toString(), 
                request.getExpiresAt()
            );
            
            // Update available seats count
            availableSeats -= request.getSeatCount();
            
            // Break if we've allocated all available seats
            if (availableSeats <= 0) {
                break;
            }
        }
    }
    
    // Handle standby expiration
    @Scheduled(fixedRate = 60000) // Run every minute
    public void processExpiredStandby() {
        LocalDateTime now = LocalDateTime.now();
        List<StandbyRequest> expiredRequests = standbyRepository
            .findByStatusAndExpiresAtBefore("NOTIFIED", now);
            
        for (StandbyRequest request : expiredRequests) {
            // Release reserved seats
            seatInventoryService.releaseStandbyReservation(
                request.getShowtimeId(), 
                request.getUserId().toString()
            );
            
            // Mark as expired
            request.setStatus("EXPIRED");
            standbyRepository.save(request);
            
            // Publish event so we can process the next standby request
            kafkaTemplate.send("standby-expired", new StandbyExpiredEvent(
                request.getShowtimeId(), request.getSeatCount()));
        }
    }
    
    // Mark standby request as fulfilled when booking is completed
    public void markAsFulfilled(Long userId, Long showtimeId) {
        List<StandbyRequest> requests = standbyRepository
            .findByUserIdAndShowtimeIdAndStatus(userId, showtimeId, "NOTIFIED");
            
        for (StandbyRequest request : requests) {
            request.setStatus("FULFILLED");
            standbyRepository.save(request);
        }
    }
}
```

**Performance & Scaling**
- Prioritize standby processing based on seat availability.
- Use distributed locks to prevent race conditions in standby allocation.
- Implement backpressure mechanisms for high-volume standby requests.

**Pitfalls & Best Practices**
- Handle edge cases like partial seat availability (when available seats < requested count).
- Clear communication about how the standby system works to manage user expectations.
- Implement reasonable timeouts for standby offers (typically 10-15 minutes).
- Consider loyalty program integration for prioritizing standby requests.
- Handle reactivation of expired standby requests if users still want to remain on the list.

### Notification System

**Concept & Principles**
- Event-driven architecture for various notifications.
- Multiple delivery channels (email, SMS, push notifications).
- Notification templating and personalization.
- Delivery tracking and resend policies.

**Implementation Example**

```java
@Service
public class NotificationService {
    
    @Autowired
    private EmailService emailService;
    
    @Autowired
    private SmsService smsService;
    
    @Autowired
    private PushNotificationService pushService;
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private NotificationRepository notificationRepository;
    
    @KafkaListener(topics = "booking-confirmed")
    public void sendBookingConfirmation(BookingConfirmedEvent event) {
        Long userId = event.getUserId();
        User user = userService.findById(userId);
        
        // Create notification record
        Notification notification = new Notification();
        notification.setUserId(userId);
        notification.setType("BOOKING_CONFIRMATION");
        notification.setReferenceId(event.getBookingId().toString());
        notification.setContent(buildBookingConfirmationContent(event));
        notification.setStatus("PENDING");
        notification = notificationRepository.save(notification);
        
        // Send via user's preferred channels
        List<String> channels = user.getNotificationPreferences().getOrDefault(
            "BOOKING_CONFIRMATION", Arrays.asList("EMAIL"));
        
        boolean sent = false;
        
        if (channels.contains("EMAIL") && user.getEmail() != null) {
            try {
                emailService.sendBookingConfirmation(
                    user.getEmail(), 
                    user.getFirstName(), 
                    event.getBookingId(),
                    event.getMovieTitle(),
                    event.getTheatreName(),
                    event.getShowtime(),
                    event.getConfirmationCode(),
                    event.getQrCodeUrl()
                );
                sent = true;
            } catch (Exception e) {
                // Log failure
            }
        }
        
        if (channels.contains("SMS") && user.getPhone() != null) {
            try {
                smsService.sendBookingConfirmation(
                    user.getPhone(),
                    event.getMovieTitle(),
                    event.getShowtime(),
                    event.getConfirmationCode()
                );
                sent = true;
            } catch (Exception e) {
                // Log failure
            }
        }
        
        if (channels.contains("PUSH") && user.getDeviceToken() != null) {
            try {
                pushService.sendBookingConfirmation(
                    user.getDeviceToken(),
                    event.getMovieTitle(),
                    event.getShowtime()
                );
                sent = true;
            } catch (Exception e) {
                // Log failure
            }
        }
        
        // Update notification status
        notification.setStatus(sent ? "SENT" : "FAILED");
        notification.setSentAt(sent ? LocalDateTime.now() : null);
        notificationRepository.save(notification);
        
        // If all channels failed, queue for retry
        if (!sent) {
            queueForRetry(notification.getId());
        }
    }
    
    public void sendStandbyAlert(Long userId, Long showtimeId, int seatCount, LocalDateTime expiresAt) {
        // Similar implementation for standby notifications
        // ...
    }
    
    // Similar methods for other notification types
    
    private void queueForRetry(Long notificationId) {
        // Add to retry queue with exponential backoff
    }
    
    private String buildBookingConfirmationContent(BookingConfirmedEvent event) {
        // Build notification content from event data
        return "...";
    }
}
```

**Notification Types**

1. **Booking Confirmation**
   - Sent immediately after successful booking
   - Includes QR code/barcode for entry
   - Contains movie, theatre, time, and seat details

2. **Booking Reminder**
   - Sent 24 hours and/or 3 hours before showtime
   - Includes theatre location and parking info

3. **Standby Alert**
   - Sent when seats become available for standby users
   - Includes time-limited offer details
   - Clear call-to-action to book seats

4. **Cancellation Confirmation**
   - Sent after successful cancellation
   - Includes refund details if applicable

5. **Marketing Notifications**
   - New movie releases
   - Special promotions
   - Theater events

**Performance & Scaling**
- Asynchronous processing through message queues.
- Batch processing for marketing notifications.
- Rate limiting per user to prevent notification fatigue.
- Staggered delivery for large-scale promotional campaigns.

**Pitfalls & Best Practices**
- Implement comprehensive retry policies with exponential backoff.
- Respect user notification preferences and compliance with regulations (CAN-SPAM, GDPR).
- Monitor delivery rates and engagement metrics to optimize notification strategy.
- Implement templates for consistent messaging and easier maintenance.
- Provide users with granular control over notification types and channels.

### Search & Filtering

**Concept & Principles**
- Full-text search for movies, theatres, and actors.
- Geospatial search for nearby theatres.
- Faceted filtering for movies (genre, language, format).
- Personalized search results based on user preferences.

**Implementation Example (Elasticsearch)**

```java
@Document(indexName = "movies")
public class MovieDocument {
    @Id
    private String id;
    
    @Field(type = FieldType.Text, analyzer = "standard")
    private String title;
    
    @Field(type = FieldType.Text, analyzer = "standard")
    private String originalTitle;
    
    @Field(type = FieldType.Integer)
    private Integer runtimeMinutes;
    
    @Field(type = FieldType.Date)
    private LocalDate releaseDate;
    
    @Field(type = FieldType.Keyword)
    private String mpaaRating;
    
    @Field(type = FieldType.Text)
    private String plotSummary;
    
    @Field(type = FieldType.Keyword)
    private List<String> genres;
    
    @Field(type = FieldType.Keyword)
    private String language;
    
    @Field(type = FieldType.Keyword)
    private String country;
    
    @Field(type = FieldType.Text, analyzer = "standard")
    private String director;
    
    @Field(type = FieldType.Text, analyzer = "standard")
    private List<String> writers;
    
    @Field(type = FieldType.Nested)
    private List<CastMember> cast;
    
    @Field(type = FieldType.Boolean)
    private boolean isActive;
    
    // Getters and setters
}

@Document(indexName = "theatres")
public class TheatreDocument {
    @Id
    private String id;
    
    @Field(type = FieldType.Text, analyzer = "standard")
    private String name;
    
    @Field(type = FieldType.Text)
    private String address;
    
    @Field(type = FieldType.Keyword)
    private String city;
    
    @Field(type = FieldType.Keyword)
    private String state;
    
    @Field(type = FieldType.Keyword)
    private String zipCode;
    
    @GeoPointField
    private GeoPoint location;
    
    @Field(type = FieldType.Boolean)
    private boolean hasImax;
    
    @Field(type = FieldType.Boolean)
    private boolean hasDolby;
    
    @Field(type = FieldType.Boolean)
    private boolean hasRecliners;
    
    // Getters and setters
}

@Service
public class SearchService {
    
    @Autowired
    private ElasticsearchRestTemplate elasticsearchTemplate;
    
    @Autowired
    private MovieRepository movieRepository;
    
    @Autowired
    private TheatreRepository theatreRepository;
    
    // Search movies with filters
    public Page<MovieDocument> searchMovies(
            String query, 
            List<String> genres, 
            String language, 
            String mpaaRating,
            boolean nowPlaying,
            Pageable pageable) {
        
        // Build query
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
        
        // Add text search if query provided
        if (query != null && !query.isEmpty()) {
            boolQuery.must(QueryBuilders.multiMatchQuery(query)
                .field("title", 2.0f)
                .field("originalTitle")
                .field("plotSummary")
                .field("director")
                .field("cast.name")
                .type(MultiMatchQueryBuilder.Type.BEST_FIELDS));
        }
        
        // Add filters
        if (genres != null && !genres.isEmpty()) {
            boolQuery.filter(QueryBuilders.termsQuery("genres", genres));
        }
        
        if (language != null && !language.isEmpty()) {
            boolQuery.filter(QueryBuilders.termQuery("language", language));
        }
        
        if (mpaaRating != null && !mpaaRating.isEmpty()) {
            boolQuery.filter(QueryBuilders.termQuery("mpaaRating", mpaaRating));
        }
        
        // Only active movies
        boolQuery.filter(QueryBuilders.termQuery("isActive", true));
        
        // Only now playing if requested
        if (nowPlaying) {
            boolQuery.filter(QueryBuilders.rangeQuery("releaseDate")
                .lte(LocalDate.now())
                .gte(LocalDate.now().minusMonths(3)));
        }
        
        // Create search query
        NativeSearchQuery searchQuery = new NativeSearchQueryBuilder()
            .withQuery(boolQuery)
            .withPageable(pageable)
            .build();
            
        // Execute search
        return elasticsearchTemplate.search(searchQuery, MovieDocument.class)
            .map(SearchHit::getContent);
    }
    
    // Search nearby theatres
    public List<TheatreDocument> findNearbyTheatres(double lat, double lon, double distanceKm) {
        // Create a geo distance query
        GeoDistanceQueryBuilder geoQuery = QueryBuilders.geoDistanceQuery("location")
            .point(lat, lon)
            .distance(distanceKm, DistanceUnit.KILOMETERS);
            
        // Build the query
        NativeSearchQuery searchQuery = new NativeSearchQueryBuilder()
            .withQuery(geoQuery)
            .withSort(SortBuilders.geoDistanceSort("location", lat, lon)
                .order(SortOrder.ASC))
            .build();
            
        // Execute search
        return elasticsearchTemplate.search(searchQuery, TheatreDocument.class)
            .map(SearchHit::getContent)
            .toList();
    }
    
    // Find movies playing at a specific theatre
    public List<MovieDocument> findMoviesAtTheatre(String theatreId) {
        // This would typically involve a join-like operation or a denormalized index
        // ...
        return Collections.emptyList();
    }
}
```

**Indexing Strategy**

```java
@Service
public class SearchIndexService {
    
    @Autowired
    private MovieRepository movieRepository;
    
    @Autowired
    private ElasticsearchOperations elasticsearchOperations;
    
    @KafkaListener(topics = "movie-updated")
    public void handleMovieUpdate(MovieUpdatedEvent event) {
        String movieId = event.getMovieId();
        
        // Fetch updated movie from the database
        Movie movie = movieRepository.findById(movieId).orElse(null);
        if (movie == null) {
            return;
        }
        
        // Convert to document
        MovieDocument document = convertToDocument(movie);
        
        // Index or update in Elasticsearch
        elasticsearchOperations.save(document);
    }
    
    // Bulk reindex
    public void reindexAllMovies() {
        // Delete existing index
        elasticsearchOperations.indexOps(MovieDocument.class).delete();
        
        // Create new index with mapping
        elasticsearchOperations.indexOps(MovieDocument.class).create();
        
        // Fetch all movies from database in batches
        long count = movieRepository.count();
        int batchSize = 100;
        long pages = (count + batchSize - 1) / batchSize;
        
        for (int i = 0; i < pages; i++) {
            Pageable pageable = PageRequest.of(i, batchSize);
            List<Movie> movies = movieRepository.findAll(pageable).getContent();
            
            List<MovieDocument> documents = movies.stream()
                .map(this::convertToDocument)
                .collect(Collectors.toList());
                
            // Bulk index
            elasticsearchOperations.save(documents);
        }
    }
    
    private MovieDocument convertToDocument(Movie movie) {
        // Conversion logic
        // ...
        return new MovieDocument();
    }
}
```

**Performance & Scaling**
- Shard Elasticsearch indices appropriately for search volume.
- Implement autocomplete with n-grams or completion suggester.
- Use scroll API for deep pagination of search results.
- Consider denormalizing data for common search patterns.

**Pitfalls & Best Practices**
- Balance precision vs. recall in search algorithms.
- Handle typos and misspellings with fuzzy matching.
- Regularly reindex to keep search data fresh.
- Use aliases for zero-downtime reindexing.
- Implement relevance scoring tuning based on user behavior.
- Consider language-specific analyzers for international content.

### Event-Driven Architecture (Kafka)

**Concept & Principles**
- Decoupling services through asynchronous events.
- Enabling real-time data pipelines and integrations.
- Ensuring reliability with message persistence.
- Supporting horizontal scaling of producers and consumers.

**Key Event Types**

1. **BookingEvents**
   - BookingCreated: Initial booking creation
   - BookingConfirmed: Payment successful
   - BookingCancelled: User cancels booking
   - BookingModified: User adds/changes seats

2. **InventoryEvents**
   - SeatsLocked: Temporary reservation
   - SeatsReleased: Reservation expired
   - SeatAvailabilityChanged: Inventory update

3. **UserEvents**
   - UserRegistered: New user signup
   - UserProfileUpdated: Profile changes
   - UserPreferencesChanged: Notification preferences

4. **ContentEvents**
   - MovieAdded: New movie in catalog
   - MovieUpdated: Details changed
   - ShowtimeAdded: New screening scheduled
   - ShowtimeCancelled: Screening cancelled

**Implementation Example (Kafka with Spring)**

```java
@Configuration
public class KafkaProducerConfig {
    
    @Value("${kafka.bootstrap-servers}")
    private String bootstrapServers;
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        // Enable idempotence for exactly-once semantics
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        return new DefaultKafkaProducerFactory<>(props);
    }
    
    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}

@Configuration
public class KafkaConsumerConfig {
    
    @Value("${kafka.bootstrap-servers}")
    private String bootstrapServers;
    
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        props.put(JsonDeserializer.TRUSTED_PACKAGES, "com.movietickets.events");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        return new DefaultKafkaConsumerFactory<>(props);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
}

@Service
public class BookingEventProducer {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void publishBookingConfirmed(Booking booking) {
        BookingConfirmedEvent event = new BookingConfirmedEvent();
        event.setBookingId(booking.getId());
        event.setUserId(booking.getUserId());
        event.setShowtimeId(booking.getShowtimeId());
        event.setMovieTitle(booking.getMovieTitle());
        event.setTheatreName(booking.getTheatreName());
        event.setShowtime(booking.getShowtime());
        event.setConfirmationCode(booking.getConfirmationCode());
        event.setSeats(booking.getSeats());
        event.setAmount(booking.getTotalAmount());
        
        // Use booking ID as key for ordering
        kafkaTemplate.send("booking-confirmed", booking.getId().toString(), event);
    }
    
    public void publishBookingCancelled(Booking booking, List<Long> cancelledSeatIds) {
        BookingCancelledEvent event = new BookingCancelledEvent();
        event.setBookingId(booking.getId());
        event.setUserId(booking.getUserId());
        event.setShowtimeId(booking.getShowtimeId());
        event.setCancelledSeatIds(cancelledSeatIds);
        event.setCancelledSeatCount(cancelledSeatIds.size());
        
        kafkaTemplate.send("booking-cancelled", booking.getId().toString(), event);
    }
    
    // Additional methods for other booking events
}

@Service
public class NotificationEventConsumer {
    
    @Autowired
    private NotificationService notificationService;
    
    @KafkaListener(topics = "booking-confirmed", groupId = "notification-service-group")
    public void handleBookingConfirmed(BookingConfirmedEvent event) {
        try {
            notificationService.sendBookingConfirmation(event);
        } catch (Exception e) {
            // Log error and possibly retry
        }
    }
    
    @KafkaListener(topics = "booking-cancelled", groupId = "notification-service-group")
    public void handleBookingCancelled(BookingCancelledEvent event) {
        try {
            notificationService.sendCancellationConfirmation(
                event.getUserId(), 
                event.getBookingId()
            );
        } catch (Exception e) {
            // Log error and possibly retry
        }
    }
    
    // Additional handlers for other events
}
```

**Performance & Scaling**
- Partition topics based on expected throughput.
- Horizontal scaling of consumer groups for parallel processing.
- Compaction for certain topics to maintain state (e.g., latest movie info).
- Schema evolution strategy for backward compatibility.

**Pitfalls & Best Practices**
- Implement idempotent consumers to handle message duplicates.
- Use dead letter queues for failed message processing.
- Monitor consumer lag to detect processing bottlenecks.
- Implement proper error handling and retries with backoff.
- Consider exactly-once semantics for critical workflows.
- Document event schemas and maintain a schema registry.

### Data Retention & Archiving

**Concept & Principles**
- Meeting the 5-year data retention requirement.
- Cost-effective storage of historical data.
- Compliance with data protection regulations.
- Efficient data retrieval for audits or analytics.

**Implementation Approach**

1. **Data Lifecycle Management**
   ```java
   @Service
   public class DataArchiveService {
       
       @Autowired
       private BookingRepository bookingRepository;
       
       @Autowired
       private ArchiveRepository archiveRepository;
       
       @Autowired
       private S3Client s3Client;
       
       @Value("${archive.bucket-name}")
       private String bucketName;
       
       // Run monthly to archive bookings older than 1 year
       @Scheduled(cron = "0 0 1 1 * ?") // At 1 AM on the 1st day of each month
       public void archiveOldBookings() {
           LocalDateTime oneYearAgo = LocalDateTime.now().minusYears(1);
           
           // Get paginated old bookings to avoid memory issues
           int page = 0;
           int pageSize = 1000;
           Page<Booking> bookings;
           
           do {
               bookings = bookingRepository.findByShowtimeBeforeAndArchivedFalse(
                   oneYearAgo, PageRequest.of(page, pageSize));
               
               if (bookings.hasContent()) {
                   List<BookingArchive> archives = new ArrayList<>();
                   List<Long> archivedIds = new ArrayList<>();
                   
                   for (Booking booking : bookings) {
                       // Convert to archive entity with all necessary data
                       BookingArchive archive = convertToArchive(booking);
                       archives.add(archive);
                       archivedIds.add(booking.getId());
                   }
                   
                   // Save to archive storage
                   archiveRepository.saveAll(archives);
                   
                   // Mark as archived in source database
                   bookingRepository.markAsArchived(archivedIds);
                   
                   // For very old data (3+ years), export to S3 and remove from archive DB
                   archiveToS3IfNeeded(archives);
               }
               
               page++;
           } while (bookings.hasNext());
       }
       
       // Export very old archives to S3
       private void archiveToS3IfNeeded(List<BookingArchive> archives) {
           LocalDateTime threeYearsAgo = LocalDateTime.now().minusYears(3);
           
           List<BookingArchive> veryOldArchives = archives.stream()
               .filter(a -> a.getShowtime().isBefore(threeYearsAgo))
               .collect(Collectors.toList());
               
           if (!veryOldArchives.isEmpty()) {
               // Group by year and month for organized storage
               Map<YearMonth, List<BookingArchive>> groupedArchives = veryOldArchives.stream()
                   .collect(Collectors.groupingBy(a -> YearMonth.from(a.getShowtime())));
                   
               for (Map.Entry<YearMonth, List<BookingArchive>> entry : groupedArchives.entrySet()) {
                   YearMonth yearMonth = entry.getKey();
                   List<BookingArchive> monthlyArchives = entry.getValue();
                   
                   // Convert to JSON
                   String archiveJson = convertToJson(monthlyArchives);
                   
                   // Upload to S3
                   String key = String.format("bookings/%d/%02d/archives.json", 
                       yearMonth.getYear(), yearMonth.getMonthValue());
                       
                   s3Client.putObject(
                       PutObjectRequest.builder()
                           .bucket(bucketName)
                           .key(key)
                           .build(),
                       RequestBody.fromString(archiveJson)
                   );
                   
                   // After confirmed S3 upload, delete from archive database
                   List<Long> idsToDelete = monthlyArchives.stream()
                       .map(BookingArchive::getId)
                       .collect(Collectors.toList());
                       
                   archiveRepository.deleteAllById(idsToDelete);
               }
           }
       }
       
       // Retrieve archived booking data (for customer service or audits)
       public BookingArchive retrieveArchivedBooking(String confirmationCode) {
           // First check the archive database
           BookingArchive archive = archiveRepository.findByConfirmationCode(confirmationCode);
           
           if (archive != null) {
               return archive;
           }
           
           // If not found, search in S3 archives
           // This would be implemented with S3 Select or by retrieving and searching
           // through archived JSON files based on estimated date range
           // ...
           
           return null;
       }
       
       private BookingArchive convertToArchive(Booking booking) {
           // Conversion logic that ensures all required data is preserved
           // ...
           return new BookingArchive();
       }
       
       private String convertToJson(List<BookingArchive> archives) {
           // JSON serialization
           // ...
           return "";
       }
   }
   ```

2. **Archive Data Schema**
   ```java
   @Entity
   @Table(name = "booking_archives")
   public class BookingArchive {
       @Id
       private Long id;
       
       @Column(name = "original_booking_id")
       private Long originalBookingId;
       
       @Column(name = "confirmation_code")
       private String confirmationCode;
       
       @Column(name = "user_id")
       private Long userId;
       
       @Column(name = "user_email", length = 100)
       private String userEmail;
       
       @Column(name = "movie_title", length = 255)
       private String movieTitle;
       
       @Column(name = "theatre_name", length = 255)
       private String theatreName;
       
       @Column(name = "showtime")
       private LocalDateTime showtime;
       
       @Column(name = "seat_info", columnDefinition = "TEXT")
       private String seatInfo;
       
       @Column(name = "total_amount")
       private BigDecimal totalAmount;
       
       @Column(name = "payment_method", length = 50)
       private String paymentMethod;
       
       @Column(name = "transaction_id", length = 100)
       private String transactionId;
       
       @Column(name = "booking_time")
       private LocalDateTime bookingTime;
       
       @Column(name = "booking_status", length = 20)
       private String bookingStatus;
       
       @Column(name = "archive_time")
       private LocalDateTime archiveTime;
       
       // Getters and setters
   }
   ```

**Performance & Scaling**
- Batch processing for archival operations to minimize impact on production systems.
- Time-based partitioning for efficient data management and query performance.
- Tiered storage strategy (hot storage → warm storage → cold storage).
- Compressed storage formats for space efficiency.

**Pitfalls & Best Practices**
- Ensure data consistency between production and archive storage.
- Implement robust data encryption for sensitive information.
- Maintain proper index structures for efficient data retrieval.
- Regularly test data restoration procedures.
- Document retention and purging policies in compliance with regulations.
- Consider data anonymization for privacy compliance after extended retention periods.

## Performance & Scalability Considerations

1. **Read vs. Write Optimization**
   - Movie ticketing systems tend to be read-heavy (browsing movies, theatres, showtimes) with periodic write spikes (popular movie releases).
   - Optimize for read performance with appropriate caching and read replicas.
   - Scale write capacity for peak times (new releases, weekends).

2. **Distributed Caching**
   - Implement Redis Cluster for high-performance, distributed caching.
   - Cache movie and theatre information aggressively.
   - Use short TTLs for volatile data like seat availability.

3. **Database Scaling**
   - Horizontal sharding by geography/region for multi-region deployments.
   - Vertical partitioning to separate hot and cold data paths.
   - Consider read replicas for reporting and analytics workloads.

4. **CDN Integration**
   - Use CDNs for static assets (images, movie posters, trailers).
   - Implement edge caching for semi-static content (theatre details, movie information).
   - Optimize media delivery for mobile devices.

5. **Concurrency Handling**
   - Robust seat locking mechanism to prevent double bookings.
   - Optimistic concurrency control for non-critical updates.
   - Rate limiting for high-traffic endpoints.

6. **Asynchronous Processing**
   - Move non-critical operations (notifications, analytics) to background jobs.
   - Implement event-driven architecture for system-wide state propagation.
   - Use message queues for workload distribution and buffering.

7. **Service Scaling**
   - Horizontally scale stateless services (search, movie, theatre).
   - Implement auto-scaling based on traffic patterns.
   - Consider service isolation for critical components (payment processing).

8. **Search Optimization**
   - Elasticsearch cluster with appropriate sharding for search throughput.
   - Implement geo-sharding for location-based searches.
   - Optimize indexing strategy for common search patterns.

9. **Monitoring & Alerting**
   - Real-time monitoring of seat availability and booking rates.
   - Alert on anomalies or potential system overloads.
   - Track user conversion funnels for UX optimization.

10. **Geographic Distribution**
    - Regional deployments for multi-market systems.
    - Data locality to minimize latency for users.
    - Global load balancing with traffic routing optimization.

## Common Pitfalls & How to Avoid Them

1. **Race Conditions in Seat Booking**
   - **Pitfall**: Multiple users attempting to book the same seats simultaneously.
   - **Solution**: Implement distributed locking with Redis or pessimistic locking at the database level.

2. **Inefficient Cache Invalidation**
   - **Pitfall**: Stale data causing inconsistencies (showing sold seats as available).
   - **Solution**: Event-based cache invalidation and short TTLs for volatile data.

3. **Payment Gateway Failures**
   - **Pitfall**: Payment failures resulting in lost bookings or double charges.
   - **Solution**: Implement idempotency keys, proper error handling, and clear feedback to users.

4. **Standby System Bottlenecks**
   - **Pitfall**: Poor standby implementation causing seat waste or customer frustration.
   - **Solution**: Time-limited offers, clear communication, and fair queuing mechanisms.

5. **Overselling of Seats**
   - **Pitfall**: Database inconsistencies leading to multiple bookings for the same seat.
   - **Solution**: Transactional seat reservation and booking confirmation.

6. **Peak Traffic Handling**
   - **Pitfall**: System overload during blockbuster releases causing poor user experience.
   - **Solution**: Load testing, auto-scaling, and traffic throttling where appropriate.

7. **Data Retention Compliance**
   - **Pitfall**: Failing to meet 5-year retention requirements or improper data handling.
   - **Solution**: Systematic archiving process with proper security and retrieval capabilities.

8. **Poor Search Performance**
   - **Pitfall**: Slow or inaccurate search results frustrating users.
   - **Solution**: Optimize Elasticsearch indices, implement autocomplete, and tune relevance scoring.

9. **Notification Delivery Failures**
   - **Pitfall**: Users missing important booking confirmations or standby alerts.
   - **Solution**: Multiple notification channels, delivery tracking, and retry mechanisms.

10. **Inconsistent User Experience**
    - **Pitfall**: Different behavior across web and mobile platforms.
    - **Solution**: API-first design, consistent business logic centralization, and thorough testing across platforms.

## Best Practices & Maintenance

1. **Automated Testing**
   - Comprehensive unit and integration tests for critical flows (booking, payment).
   - Regular load testing for peak traffic scenarios.
   - Chaos engineering to test resilience against partial failures.

2. **Monitoring & Observability**
   - Real-time dashboards for system health and business metrics.
   - Distributed tracing to diagnose cross-service issues.
   - Log aggregation and analysis for troubleshooting.

3. **Database Management**
   - Regular index optimization and query analysis.
   - Scheduled database maintenance during off-peak hours.
   - Proper backup and recovery procedures.

4. **Security Audits**
   - Regular security assessments, especially for payment flows.
   - PCI-DSS compliance for payment card handling.
   - Penetration testing for external-facing interfaces.

5. **Deployment Strategy**
   - Blue-green deployments for zero-downtime updates.
   - Canary releases for risky changes.
   - Feature flags for controlled rollouts.

6. **Caching Policy**
   - Document and regularly review TTL settings.
   - Implement circuit breakers for cache dependencies.
   - Monitor cache hit rates and optimize accordingly.

7. **API Versioning**
   - Clear versioning strategy for backward compatibility.
   - Scheduled deprecation of old API versions.
   - Comprehensive API documentation.

8. **Data Integrity Checks**
   - Regular reconciliation of seat inventory vs. bookings.
   - Automated detection of data anomalies.
   - Periodic data consistency checks.

9. **Performance Optimization**
   - Regular performance reviews and optimizations.
   - Clear performance SLAs for critical operations.
   - Query optimization and service profiling.

10. **Disaster Recovery**
    - Documented DR procedures with regular drills.
    - Multi-region failover capability.
    - Business continuity planning for critical functions.

## How to Discuss in a Principal Engineer Interview

1. **Requirements Clarification**
   - Start by discussing core functional and non-functional requirements.
   - Highlight the importance of preventing double bookings and ensuring secure transactions.
   - Discuss the 5-year data retention requirement and its implications.

2. **Architecture Overview**
   - Present the high-level microservice architecture.
   - Explain the key data flows (browsing, booking, standby).
   - Discuss the rationale behind service boundaries and data store choices.

3. **Critical System Components**
   - Deep dive into the seat inventory system and concurrency control.
   - Explain the payment processing flow and security measures.
   - Discuss the standby system implementation.

4. **Data Management**
   - Explain the database design for theatre seats and bookings.
   - Discuss the archiving strategy for the 5-year retention requirement.
   - Address data consistency challenges in a distributed system.

5. **Scalability Approach**
   - Explain how the system handles traffic spikes during new releases.
   - Discuss caching strategies for performance optimization.
   - Address geographic scaling for multi-market deployments.

6. **Fault Tolerance**
   - Describe how the system handles partial failures.
   - Explain the idempotency patterns for critical operations.
   - Discuss the monitoring and alerting strategy.

7. **Security Considerations**
   - Explain the approach to secure payment processing.
   - Discuss user data protection and privacy compliance.
   - Address potential security vulnerabilities and mitigations.

8. **Real-World Challenges**
   - Discuss common operational issues in ticketing systems.
   - Explain how to handle refunds and cancellations.
   - Address fraud prevention and detection.

9. **Continuous Improvement**
   - Discuss how user behavior analytics drive system evolution.
   - Explain the approach to A/B testing new features.
   - Address technical debt management.

10. **Future Enhancements**
    - Discuss potential AI/ML integrations for recommendations.
    - Address emerging technologies like mobile wallets or biometric authentication.
    - Explain how the system could evolve based on changing user expectations.

## Conclusion

This design outlines a **robust, scalable movie ticketing system** that effectively addresses the key requirements while preparing for future growth. The architecture leverages modern microservices patterns and event-driven design to ensure reliability, performance, and maintainability.

The system emphasizes **concurrency control** to prevent double bookings, implements a flexible **standby mechanism** for waitlisting, and ensures **secure payment processing** for financial transactions. The **search and filtering** capabilities provide a rich user experience, while the **notification system** keeps users informed throughout their journey.

By adopting a polyglot persistence approach with **relational databases** for transactional data, **NoSQL** for flexible schemas, **Redis** for caching and locking, and **Elasticsearch** for search, the system optimizes for different access patterns and scaling requirements.

The **data retention and archiving** strategy satisfies the 5-year requirement while managing costs through tiered storage, ensuring both compliance and performance. The **event-driven architecture** with Kafka enables loose coupling, system resilience, and real-time updates across services.

This architecture balances immediate business needs with long-term scalability, providing a solid foundation for a movie ticketing platform that can evolve with changing market demands and user expectations.
