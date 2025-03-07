# Parking Lot Management System OOD Design

## Table of Contents
- [1. Class Definitions](#1-class-definitions)
- [2. Relationships](#2-relationships)
- [3. Constraints Handling](#3-constraints-handling)
- [4. Implementation Considerations (with Java Examples)](#4-implementation-considerations-with-java-examples)
- [5. Performance and Scaling Considerations](#5-performance-and-scaling-considerations)
- [6. Common Pitfalls and Best Practices](#6-common-pitfalls-and-best-practices)
- [7. Interview Discussion Tips](#7-interview-discussion-tips)

---

## 1. Class Definitions

**ParkingLot:** Represents the entire parking facility (often a singleton instance). It contains collections of parking spots (possibly grouped by floors) and provides methods to issue tickets and free spots. Key attributes may include:
- A list of ParkingFloor (or a direct list of ParkingSpot if single-level).
- A record of active tickets.
- Configuration like capacity or rates.

**ParkingFloor (optional):** If the lot has multiple levels, this class can encapsulate a floor’s identifier and the spots on that floor. It organizes spots and may track floor-specific info (e.g., a display board of free spots).

**ParkingSpot:** An abstract/base class for parking spaces. It has a unique spot ID, a spot type/size, and an “occupied” indicator. Subclasses (or a type field) can represent specific spot sizes/types (e.g., CompactSpot, LargeSpot, MotorcycleSpot, HandicappedSpot, ElectricSpot). Each spot may hold a reference to the currently parked Vehicle.

**Vehicle:** An abstract/base class for vehicles. Attributes include licensePlate and vehicle type/size. Subclasses or an enum can represent categories (Car, Motorcycle, Truck, ElectricCar). The system uses vehicle type/size to determine appropriate spots.

**ParkingTicket:** Captures the details of a parked vehicle’s session (unique ticket ID, vehicle info, assigned spot, timestamps for entry/exit, status). Issued when a vehicle enters; used for payment and spot release upon exit.

**ParkingAttendant:** Represents a person or system agent managing operations. Attributes may include ID, name, permissions. Attendants can issue tickets, assist with payments, and override constraints if needed. In software, they might call ParkingLot methods to handle entry/exit.

**EntranceGate/ExitGate (or Panel):** Optional classes to simulate entry/exit kiosks. Entrance might create a ticket, exit might finalize payment. 

**PaymentProcessor/Payment:** Handles payment. Could be an interface (PaymentStrategy) with different payment method implementations (cash, credit, etc.).

**RateCalculator/ParkingRate:** Calculates fees (could vary by vehicle type, duration, special conditions).

**DisplayBoard:** Shows available spots (optionally per floor).

These classes work together: vehicles arrive, get a ParkingTicket (linked to a free ParkingSpot), and upon exit, pay and free that spot.

---

## 2. Relationships

A typical UML diagram includes:
- **Composition:** ParkingLot has ParkingFloor(s) and ParkingSpot(s). Destroying ParkingLot implies destroying floors/spots.  
- **Association:** ParkingLot uses ParkingTicket and Vehicle. An attendant interacts with ParkingLot but is not owned by it.  
- **Inheritance:** Subclasses for ParkingSpot (e.g., CompactSpot) and for Vehicle (e.g., Car, Motorcycle). Possibly also for Payment or RateCalculator strategies.  
- **Aggregation:** ParkingLot might maintain a collection of active ParkingTicket entries. Those tickets exist while the lot is operational.  
- **Relationships in action:** A Vehicle arrives, the system finds a suitable spot, creates a ParkingTicket linking vehicle and spot. On exit, the ticket is used to calculate fees, payment is processed, and the spot is freed.

---

## 3. Constraints Handling

**Vehicle Sizes and Spot Sizes:**  
- Different vehicle types (car, motorcycle, truck, electric). Each spot can handle certain sizes. For example, a motorcycle fits smaller spots, a truck needs large. Logic could be an enum or subclass check.

**Parking Spot Availability & Allocation:**  
- The system must track free spots and assign them efficiently. For concurrency, `issueTicket` might be synchronized to avoid conflicts when multiple vehicles arrive simultaneously.  
- If the lot is full for a certain vehicle type, the system indicates “Parking Full.”

**Payment Processing & Ticket Validation:**  
- On exit, the system calculates fees based on duration (and possibly vehicle/spot type). Different payment methods can be supported via strategy. Once paid, the ticket is marked as resolved, and the spot is freed.

**Special Cases (Disabled, Reserved, EV spots):**  
- **Disabled/Handicapped:** Spots for drivers with disability permits. The system checks for a permit flag before assigning these.  
- **Reserved:** Some spots are reserved for specific users/times; the system excludes them unless the user matches the reservation.  
- **Electric Vehicle (EV) Charging:** Electric spots are assigned to EVs. Possibly the system can prefer electric spots for an electric vehicle, or allow fallback to other spots depending on policy.

---

## 4. Implementation Considerations (with Java Examples)

Below are sample classes and methods (simplified):

```java
public enum VehicleType { MOTORBIKE, CAR, TRUCK, ELECTRIC }
public enum SpotType { MOTORBIKE, COMPACT, LARGE, HANDICAPPED, ELECTRIC }

// Abstract Vehicle class
abstract class Vehicle {
    private String licensePlate;
    private VehicleType type;

    public Vehicle(String plate, VehicleType type) {
        this.licensePlate = plate;
        this.type = type;
    }
    public VehicleType getType() { return type; }
    public String getLicensePlate() { return licensePlate; }
}

// Concrete vehicles
class Car extends Vehicle {
    public Car(String plate) { super(plate, VehicleType.CAR); }
}
class Motorcycle extends Vehicle {
    public Motorcycle(String plate) { super(plate, VehicleType.MOTORBIKE); }
}
class Truck extends Vehicle {
    public Truck(String plate) { super(plate, VehicleType.TRUCK); }
}
class ElectricCar extends Vehicle {
    public ElectricCar(String plate) { super(plate, VehicleType.ELECTRIC); }
}

// Abstract ParkingSpot
abstract class ParkingSpot {
    private String spotId;
    private SpotType type;
    protected Vehicle currentVehicle;

    public ParkingSpot(String id, SpotType type) {
        this.spotId = id;
        this.type = type;
    }
    public SpotType getType() { return type; }
    public boolean isAvailable() { return currentVehicle == null; }

    public boolean canFitVehicle(Vehicle vehicle) {
        VehicleType vt = vehicle.getType();
        switch (this.type) {
            case LARGE:
                return (vt == VehicleType.CAR || vt == VehicleType.TRUCK || vt == VehicleType.ELECTRIC);
            case COMPACT:
                return (vt == VehicleType.CAR || vt == VehicleType.ELECTRIC || vt == VehicleType.MOTORBIKE);
            case MOTORBIKE:
                return (vt == VehicleType.MOTORBIKE);
            case ELECTRIC:
                return (vt == VehicleType.ELECTRIC);
            case HANDICAPPED:
                return (vt == VehicleType.CAR || vt == VehicleType.ELECTRIC || vt == VehicleType.TRUCK);
        }
        return false;
    }

    public void assignVehicle(Vehicle vehicle) {
        if (!canFitVehicle(vehicle) || !isAvailable()) {
            throw new IllegalStateException("Cannot park here!");
        }
        this.currentVehicle = vehicle;
    }
    public void removeVehicle() {
        this.currentVehicle = null;
    }
}

// Concrete spot classes
class CompactSpot extends ParkingSpot {
    public CompactSpot(String id) { super(id, SpotType.COMPACT); }
}
class LargeSpot extends ParkingSpot {
    public LargeSpot(String id) { super(id, SpotType.LARGE); }
}
class MotorbikeSpot extends ParkingSpot {
    public MotorbikeSpot(String id) { super(id, SpotType.MOTORBIKE); }
}
class ElectricSpot extends ParkingSpot {
    public ElectricSpot(String id) { super(id, SpotType.ELECTRIC); }
}
class HandicappedSpot extends ParkingSpot {
    public HandicappedSpot(String id) { super(id, SpotType.HANDICAPPED); }
}

// ParkingLot managing spots, tickets, etc.
class ParkingLot {
    private String name;
    private List<ParkingFloor> floors = new ArrayList<>();
    private Map<String, ParkingTicket> activeTickets = new HashMap<>();
    private Map<SpotType, Queue<ParkingSpot>> freeSpotsByType = new HashMap<>();

    public ParkingTicket issueTicket(Vehicle vehicle) {
        SpotType requiredSpotType = findSuitableSpotType(vehicle.getType());
        ParkingSpot spot = getNextAvailableSpot(requiredSpotType);
        if (spot == null) {
            throw new IllegalStateException("Parking Full for vehicle type: " + vehicle.getType());
        }
        spot.assignVehicle(vehicle);
        ParkingTicket ticket = new ParkingTicket(vehicle, spot);
        activeTickets.put(ticket.getTicketNumber(), ticket);
        return ticket;
    }

    private SpotType findSuitableSpotType(VehicleType vt) {
        switch(vt) {
            case MOTORBIKE: return SpotType.MOTORBIKE;
            case TRUCK:     return SpotType.LARGE;
            case ELECTRIC:  return SpotType.ELECTRIC;
            case CAR:       return SpotType.COMPACT;
            default:        return SpotType.LARGE;
        }
    }

    private ParkingSpot getNextAvailableSpot(SpotType type) {
        Queue<ParkingSpot> queue = freeSpotsByType.get(type);
        if(queue == null || queue.isEmpty()) {
            if(type == SpotType.COMPACT) {
                return getNextAvailableSpot(SpotType.LARGE);
            }
            if(type == SpotType.ELECTRIC) {
                ParkingSpot spot = getNextAvailableSpot(SpotType.ELECTRIC);
                if(spot == null) spot = getNextAvailableSpot(SpotType.COMPACT);
                if(spot == null) spot = getNextAvailableSpot(SpotType.LARGE);
                return spot;
            }
            return null;
        }
        return queue.poll();
    }

    public double payAndExit(String ticketId, PaymentMethod paymentMethod) {
        ParkingTicket ticket = activeTickets.get(ticketId);
        if(ticket == null) {
            throw new IllegalArgumentException("Invalid ticket");
        }
        ticket.markExitTime();
        double amountDue = RateCalculator.calculateFee(ticket);
        paymentMethod.pay(amountDue);
        ticket.markPaid();
        ParkingSpot spot = ticket.getSpot();
        spot.removeVehicle();
        freeSpotsByType.get(spot.getType()).offer(spot);
        activeTickets.remove(ticketId);
        return amountDue;
    }

    // other methods like addFloor, initializeFreeSpots, etc.
}

// ParkingTicket tracking a single parking session
class ParkingTicket {
    private static int ticketCounter = 0;
    private String ticketNumber;
    private Vehicle vehicle;
    private ParkingSpot spot;
    private long entryTime;
    private Long exitTime;
    private boolean paid;

    public ParkingTicket(Vehicle vehicle, ParkingSpot spot) {
        this.ticketNumber = "T" + (++ticketCounter);
        this.vehicle = vehicle;
        this.spot = spot;
        this.entryTime = System.currentTimeMillis();
        this.paid = false;
    }
    public String getTicketNumber() { return ticketNumber; }
    public ParkingSpot getSpot() { return spot; }
    public long getEntryTime() { return entryTime; }
    public Long getExitTime() { return exitTime; }
    public void markExitTime() { this.exitTime = System.currentTimeMillis(); }
    public void markPaid() { this.paid = true; }
    public boolean isPaid() { return paid; }
}

// Payment methods
interface PaymentMethod {
    boolean pay(double amount);
}

class CashPayment implements PaymentMethod {
    public boolean pay(double amount) {
        System.out.println("Cash paid: $" + amount);
        return true;
    }
}

class CardPayment implements PaymentMethod {
    public boolean pay(double amount) {
        System.out.println("Card charged: $" + amount);
        return true;
    }
}

// RateCalculator with simplified logic
class RateCalculator {
    static double hourlyRateCar = 3.0;
    static double hourlyRateMotorbike = 1.5;
    static double hourlyRateTruck = 5.0;

    static double calculateFee(ParkingTicket ticket) {
        long end = (ticket.getExitTime() != null) ? ticket.getExitTime() : System.currentTimeMillis();
        long hours = (end - ticket.getEntryTime()) / (1000 * 60 * 60);
        if ((end - ticket.getEntryTime()) % (1000 * 60 * 60) != 0) hours++;

        VehicleType type = ticket.getSpot().getType() == SpotType.MOTORBIKE ? VehicleType.MOTORBIKE
                          : ticket.getVehicle().getType();
        double rate;
        switch(type) {
            case MOTORBIKE: rate = hourlyRateMotorbike; break;
            case TRUCK:     rate = hourlyRateTruck;     break;
            default:        rate = hourlyRateCar;
        }
        return hours * rate;
    }
}

class ParkingAttendant {
    private String id;
    private String name;
    private ParkingLot lot;

    public ParkingAttendant(String id, String name, ParkingLot lot) {
        this.id = id; this.name = name; this.lot = lot;
    }

    public ParkingTicket provideTicket(Vehicle vehicle) {
        return lot.issueTicket(vehicle);
    }
    public void processExit(String ticketId, PaymentMethod payment) {
        double fee = lot.payAndExit(ticketId, payment);
        System.out.println("Collected fee $" + fee + " for ticket " + ticketId);
    }
}
```

Key notes:
- `ParkingLot.issueTicket` and `payAndExit` show the main flow for entry/exit.
- `ParkingTicket` tracks a single session.
- `RateCalculator` handles fee logic.
- `PaymentMethod` is a strategy pattern for payment.

---

## 5. Performance and Scaling Considerations

- **Efficient Lookups:** Using queues or indexed structures for free spots allows O(1) or O(log n) allocation and release.  
- **Concurrency/Throughput:** In heavy-traffic scenarios with multiple entries/exits, concurrency must be managed. Possibly synchronize on the relevant methods or use thread-safe data structures.  
- **Scaling Out:** For extremely large lots or multiple facilities, a distributed or microservices approach might be needed. For a single location, a well-structured in-memory design plus a database for persistence generally suffices.  
- **Memory Usage:** The overhead for storing spot/ticket data is minor for typical real-world sizes. If the system must store historical records of thousands of tickets, use a database and keep only the active set in memory.  
- **Extensibility:** The design is open for adding new vehicle types, spot types, or payment methods with minimal changes.

---

## 6. Common Pitfalls and Best Practices

**Pitfalls:**
- **God Class Syndrome:** Dumping all logic into a single class (e.g., ParkingLot). Delegate tasks to specialized classes (Vehicle, Spot, Payment, etc.).  
- **Ignoring Spot-Vehicle Constraints:** Must consistently enforce size compatibility or electric/handicapped restrictions.  
- **Not Freeing/Releasing Spots Correctly:** Failure to mark a spot as available after exit leads to incorrect availability state.  
- **No Handling for Special Scenarios:** Lost tickets, disabled permits, or partial payment.  
- **Hard-Coded Values:** Minimally, store rates and capacity in configurable structures or an external config.  

**Best Practices:**
- **Single Responsibility Principle:** Each class has a focused role (e.g., ParkingSpot knows occupancy, ParkingLot manages overall resources).  
- **Open-Closed Principle:** Inheritance or strategy patterns allow adding new vehicle or payment types without changing core logic.  
- **Design Patterns:** Use Singleton for ParkingLot if only one instance. Strategy for payments. Possibly a factory for vehicles or spots.  
- **Composition:** ParkingLot contains floors/spots, signifying a strong lifecycle tie.  
- **Thread Safety:** Use concurrency controls in methods that modify shared structures (e.g., `issueTicket`).  
- **Validation and Error Handling:** Gracefully handle “lot full” or “invalid ticket” scenarios.  

---

## 7. Interview Discussion Tips

- **Start with Requirements:** Summarize the parking system’s needs (vehicle-size constraints, tickets, payments, special spots, concurrency).  
- **Outline Key Classes:** Present ParkingLot, ParkingSpot, Vehicle, Ticket, Payment. Show how they fit.  
- **Show Relationships with a Use Case:** Vehicle arrives → ticket issued → suitable spot assigned → exit, fee calculation → spot freed.  
- **Address Constraints:** Spot availability, different vehicle types, handicapped/EV spots.  
- **Highlight OO Principles and Patterns:** Explain how inheritance, composition, or the strategy pattern solves specific problems.  
- **Mention Scaling and Future Extensions:** Concurrency, data persistence, or bigger features can be added without breaking the design.  
- **Communicate Clearly:** Organize the solution into logical steps or headings (classes, relationships, constraints, etc.).  
- **Justify Alternatives:** If asked why you picked X over Y, be prepared to discuss trade-offs and remain open to simpler or more flexible approaches.  

This overall design meets typical parking lot system requirements: it organizes classes around real-world concepts, enforces vehicle-spot constraints, manages ticket lifecycles, handles payment, and can scale to moderately large usage with efficient data structures and some concurrency care.
