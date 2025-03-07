# LLD for a Hotel Booking System

## Table of Contents
- [Overview & Key Requirements](#overview--key-requirements)
- [Class Design (Core Entities)](#class-design-core-entities)
  - [Hotel](#hotel)
  - [Room](#room)
  - [Booking](#booking)
    - [BookingItem](#bookingitem)
    - [AddOn](#addon)
  - [User](#user)
  - [Payment](#payment)
  - [Overbooking Logic](#overbooking-logic)
- [Availability & Booking Flow](#availability--booking-flow)
  - [Checking Availability](#checking-availability)
  - [Creating a Booking](#creating-a-booking)
  - [Canceling a Booking](#canceling-a-booking)
- [Data Storage & Microservices](#data-storage--microservices)
  - [Database Fundamentals](#database-fundamentals)
  - [NoSQL Database (MongoDB) for Hotel Metadata](#nosql-database-mongodb-for-hotel-metadata)
  - [System Design Patterns](#system-design-patterns)
  - [Spring Boot & Service Implementation](#spring-boot--service-implementation)
  - [Elasticsearch for Search](#elasticsearch-for-search)
  - [Caching with Redis](#caching-with-redis)
  - [Kafka for Asynchronous Processing](#kafka-for-asynchronous-processing)
- [Performance & Scaling](#performance--scaling)
- [Common Pitfalls & How to Avoid Them](#common-pitfalls--how-to-avoid-them)
- [Best Practices & Maintenance](#best-practices--maintenance)
- [How to Discuss in a Principal Engineer Interview](#how-to-discuss-in-a-principal-engineer-interview)

## Overview & Key Requirements
A Hotel Booking System typically handles:
1. Hotels & Rooms: Management of properties, rooms, availability, dynamic pricing, etc.
2. Booking Lifecycle:
   - Searching available rooms
   - Creating, modifying, or canceling bookings (including partial or multi-room bookings)
   - Applying overbooking rules (up to X% above capacity, configurable)
3. Payments & Add-Ons: Handling billing for rooms, extra services (like breakfast, spa), partial or full refunds upon cancellation.
4. User Accounts: Keeping user profiles and preferences.
5. Security: Ensuring secure payment and data protection.

We want to allow a configurable X% overbooking to accommodate no-shows. Each booking can include multiple rooms, with dynamic or seasonal pricing and optional add-on paid services. Concurrency must be handled so that multiple customers booking the same room still yields correct availability.

## Class Design (Core Entities)

### Hotel
```java
public class Hotel {
    private Long id;
    private String name;
    private String address;
    private List<Room> rooms;

    private double overbookingPercentage; // e.g., 5.0 means 5% overbook allowed

    // Constructors, getters, setters...
}
```
**Key points**:
- A `Hotel` aggregates multiple `Room` objects.
- `overbookingPercentage` could also be stored in a system-wide config. This guides how many rooms can be sold beyond physical capacity in a given date range.

### Room
```java
public class Room {
    private Long id;
    private String roomNumber;    // e.g., "101", "Suite-3"
    private RoomType roomType;    // e.g., SINGLE, DOUBLE, SUITE
    private BigDecimal basePrice; // base nightly rate
    private Long hotelId;         // link back to the parent hotel

    // Constructors, getters, setters...
}
```
**Key points**:
- `RoomType` might be an enum or a more detailed class.
- `basePrice` is a starting point for dynamic pricing; actual price may vary by season or demand.

### Booking
```java
public class Booking {
    private Long bookingId;
    private Long userId;
    private List<BookingItem> bookedRooms; // each references one room + date range

    private LocalDate checkInDate;
    private LocalDate checkOutDate;
    private BookingStatus status; // CREATED, CONFIRMED, CANCELLED, etc.

    private BigDecimal totalPrice; // includes add-ons
    private List<AddOn> addOns;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    // Constructors, getters, setters...
}
```
**Key points**:
- A single `Booking` can have multiple rooms (`bookedRooms`).
- `addOns` might include breakfast, airport pickup, and so on.
- `status` changes as booking goes from creation to confirmation or cancellation.

#### BookingItem
```java
public class BookingItem {
    private Long roomId;
    private LocalDate startDate;
    private LocalDate endDate;
    private BigDecimal pricePerNight;
    // Possibly discount, tax info...

    // Constructors, getters, setters...
}
```
**Key points**:
- Each `BookingItem` corresponds to one room for a specific date range.
- `pricePerNight` can be dynamic based on season or demand.

#### AddOn
```java
public class AddOn {
    private String name;      // e.g., "Breakfast Buffet"
    private BigDecimal cost;  // cost per day or flat

    // Constructors, getters, setters...
}
```
**Key points**:
- Can be multiplied by the number of nights or people.
- Designed for easy extension (e.g., different cost models).

### User
```java
public class User {
    private Long userId;
    private String name;
    private String email;
    // Possibly phone, loyalty points...

    // Constructors, getters, setters...
}
```
**Key points**:
- Basic user info. Full authentication or additional payment details typically handled by separate services.

### Payment
```java
public class Payment {
    private Long paymentId;
    private Long bookingId;
    private BigDecimal amount;
    private PaymentStatus status;  // INITIATED, SUCCESS, FAILED
    private PaymentMethod method;  // e.g., CREDIT_CARD, UPI, WALLET

    // Constructors, getters, setters...
}
```
**Key points**:
- Ties to one `Booking`; multiple Payment records can exist if partial payments are allowed.
- `PaymentMethod` can be an enum or entity if external integrations are complex.

### Overbooking Logic
Concept: If a hotel has 100 physical rooms and `overbookingPercentage = 5`, the system can accept up to 105 bookings for the same date range to account for cancellations/no-shows.
```java
public class OverbookingConfig {
    private double maxOverbookPercent; // e.g., 5.0

    // Constructors, getters, setters...
}
```
**Implementation**:
- During availability checks, if `confirmed_count < room_count * (1 + overbookPercent/100)`, booking is possible. Otherwise, reject or waitlist.

## Availability & Booking Flow

### Checking Availability
**Key Logic**:
1. For a given date range, find how many rooms are already booked.
2. Compare confirmed bookings against the physical capacity plus overbooking margin.
3. If below threshold, booking is possible.

**Pseudo-code**:
```java
public class AvailabilityService {
    public boolean isAvailable(Long hotelId, Room room,
                               LocalDate start, LocalDate end,
                               double overbookPercent) {
        int totalBookedRooms = bookingDao.countActiveBookings(
            hotelId, room.getId(), start, end
        );
        int physicalCapacity = 1; // for a single room
        int allowedCapacity = (int) (physicalCapacity * (1 + overbookPercent / 100));

        return totalBookedRooms < allowedCapacity;
    }

    public List<Room> findAvailableRooms(Long hotelId,
                                         LocalDate start,
                                         LocalDate end) {
        // Check each room to see if it's under the threshold
        // Return list of available rooms
    }
}
```

### Creating a Booking
1. **Select Rooms**: User chooses one or more rooms for the desired date range.
2. **Check Availability**: Verify each chosen room’s availability.
3. **Create Booking**: Initialize a `Booking` record with status `CREATED` or `PENDING_CONFIRMATION`.
4. **Collect Payment**: Initiate payment for total cost (room nights + add-ons + taxes).
5. **Confirm Booking**: Once payment completes, update `Booking.status` to `CONFIRMED`.

### Canceling a Booking
1. Set `Booking.status = CANCELLED`.
2. Trigger refund logic in `Payment` records if required.
3. Reduce “active bookings” count for affected rooms, freeing capacity.

## Data Storage & Microservices

### Database Fundamentals
- **Relational DB**: (MySQL/PostgreSQL) for user, booking, and payment tables, ensuring ACID transactions.
- **NoSQL**: (MongoDB) could store partial data like hotel inventory or logs that change often.

**Schema Example (Relational)**:
- **Users**: `(user_id, name, email, ...)`
- **Hotels**: `(hotel_id, name, address, overbooking_percent, ...)`
- **Rooms**: `(room_id, hotel_id, room_number, base_price, ...)`
- **Bookings**: `(booking_id, user_id, status, check_in, check_out, total_price, ...)`
- **BookingItems**: `(booking_id, room_id, start_date, end_date, price_per_night, ...)`
- **Payments**: `(payment_id, booking_id, amount, status, method, ...)`

### NoSQL Database (MongoDB) for Hotel Metadata
- Could store hotel data or user preferences in MongoDB if it’s unstructured or changes frequently.
- A caching approach or partial NoSQL store can handle daily occupancy stats for quick availability checks.

### System Design Patterns
- **Microservices**:
  - `HotelService` (hotel & room info)
  - `BookingService` (booking logic, overbooking checks)
  - `PaymentService` (payment flows, external gateway integration)
  - `NotificationService` (emails, SMS about booking confirmations)
- **SAGA Pattern**: For distributed transactions (booking creation, payment, room availability). If payment fails, mark booking as CANCELLED.
- **Event-Driven**: Use messaging (e.g., Kafka) for asynchronous communication between services.

### Spring Boot & Service Implementation
```java
@SpringBootApplication
public class BookingServiceApp {
    public static void main(String[] args) {
        SpringApplication.run(BookingServiceApp.class, args);
    }
}

@RestController
@RequestMapping("/bookings")
public class BookingController {

    @Autowired
    private BookingService bookingService;

    @PostMapping
    public Booking createBooking(@RequestBody BookingRequest request) {
        return bookingService.createBooking(request);
    }

    @PutMapping("/{bookingId}/cancel")
    public Booking cancelBooking(@PathVariable Long bookingId) {
        return bookingService.cancelBooking(bookingId);
    }
}
```
**Key points**:
- Each microservice can be a separate Spring Boot application.
- Communication can be REST or messaging-based.

### Elasticsearch for Search
- Index hotels by location, name, or availability.
- `HotelService` updates Elasticsearch when hotel data changes.
- Advanced queries or free-text search for users looking up hotels and rooms.

### Caching with Redis
- Store frequently accessed data (room availability, partial booking info, ephemeral sessions).
- Speeds up availability checks and reduces load on main DB.

### Kafka for Asynchronous Processing
- Topics like `BookingEvents`, `PaymentEvents`, `OverbookingAlerts`.
- `BookingService` publishes an event when a booking is created. `PaymentService` consumes it, processes payment, then publishes payment confirmation. `NotificationService` listens to notify users.

## Performance & Scaling
1. **Horizontal Scalability**: Each microservice can scale out behind a load balancer.
2. **Caching**: Redis for real-time availability to reduce DB load.
3. **Sharding**: Large tables might be sharded by region or date ranges.
4. **Read Replicas**: MySQL read replicas handle heavy read traffic.
5. **Async Communication**: Kafka decouples services and can handle bursts.
6. **Auto-Scaling**: Container orchestration (e.g., Kubernetes) can spawn more pods when usage spikes.

## Common Pitfalls & How to Avoid Them
1. **Overbooking Race Conditions**: Concurrent bookings can exceed allowed capacity.
   - **Solution**: Use optimistic locking or distributed locks for availability counters.
2. **Payment & Booking Consistency**: Payment might fail after booking creation.
   - **Solution**: Use a `PENDING_PAYMENT` status; cancel if payment fails.
3. **Refund Logic**: Complex partial refunds require clear business rules.
4. **Cache Invalidation**: Changes in DB must reflect in Redis or any cache.
   - **Solution**: Publish events or implement a systematic invalidation strategy.
5. **Complex Pricing**: Seasonal or dynamic rates may complicate logic.
   - **Solution**: Store in a `RatePlan` table or a separate dynamic pricing service.
6. **Security**: Payment data must be stored securely, or tokenized.

## Best Practices & Maintenance
1. **Modular Services**: Clearly separate `BookingService` and `PaymentService`.
2. **Logging & Monitoring**: Collect logs, metrics, and trace data for debugging concurrency issues.
3. **Testing Overbooking Edge Cases**: Thorough tests for date overlaps and concurrency under load.
4. **Schema Evolution**: Plan for dynamic rates or add-on expansions (e.g., a flexible NoSQL store).
5. **Security**: Enforce TLS for all calls, role-based access, secure payment data.
6. **Disaster Recovery**: Backup strategies, replication, possibly multiple data centers if global.

## How to Discuss in a Principal Engineer Interview
- **Domain Clarity**: Explain hotels, overbooking, dynamic pricing, then show how classes (Hotel, Room, Booking) map to that domain.
- **Data Modeling & Storage**: Highlight how relational data (bookings, payments) differs from NoSQL usage (e.g., flexible hotel metadata).
- **Design Patterns**: Emphasize microservices, saga workflows, event-driven architecture.
- **Scalability Approach**: Mention horizontal scaling, Redis caching, Elasticsearch indexing.
- **Security**: Discuss token-based auth, secure payment gateway integration, and encryption at rest.
- **Trade-Offs**: Overbooking can lead to cancellations or relocations; discuss concurrency and fallback strategies.
- **Implementation Detail**: Provide code examples or how you’d integrate with a payment gateway.  
- **Future Enhancements**: Loyalty programs, cross-selling (e.g., flights, car rentals), or more add-ons. Show how the design remains open for extension.
