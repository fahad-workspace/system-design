# Low-Level Design: Elevator System

## Table of Contents
- [Overview](#overview)
- [Key Entities and Class Design](#key-entities-and-class-design)
- [Scheduling Algorithm](#scheduling-algorithm)
- [Implementation Considerations](#implementation-considerations)
- [Performance and Scaling Characteristics](#performance-and-scaling-characteristics)
- [Common Pitfalls and How to Avoid Them](#common-pitfalls-and-how-to-avoid-them)
- [Best Practices for Implementation & Maintenance](#best-practices-for-implementation--maintenance)
- [How to Discuss This in an Interview](#how-to-discuss-this-in-an-interview)
- [Example Java Implementation](#example-java-implementation)

## Overview
Designing an Elevator Control System involves creating a software model that manages multiple elevators and responds to user requests efficiently. The system must handle:
- Concurrent requests from different floors and elevators
- Scheduling decisions (which request to serve next)
- Safety checks (door safety, overload prevention)
- Internal and external requests (inside elevator vs. hall calls)

The goal is to ensure robust scheduling, efficient concurrency, and reliable safety features.

## Key Entities and Class Design
An Elevator System can be broken down into several core components:

- **Elevator**  
  Represents a single elevator car, tracking:
  - `currentFloor`
  - `direction` (up, down, or idle)
  - A list of pending floor stops (requests)
  - A `DoorSystem` for managing doors

- **ElevatorController**  
  Oversees all elevators, receiving requests and assigning them to the appropriate elevator. It maintains a global view of elevator positions and statuses, dispatching requests via a scheduling strategy (e.g., nearest car).

- **Request**  
  - **ExternalRequest**: A hall call, specifying floor and desired direction (up/down).  
  - **InternalRequest**: A destination floor selected from inside the elevator.  

- **DoorSystem**  
  Manages the opening/closing of elevator doors, ensuring safe operation. It prevents movement with the doors open and reopens if an obstruction is detected.

This separation of responsibilities keeps the design modular.

## Scheduling Algorithm
Elevator systems commonly use directional scheduling:
- **Simple Directional Scheduling**: An elevator serves all pending requests in its current direction before switching directions.  
- **Load Balancing for Multiple Elevators**: The controller distributes hall calls across elevators to minimize overall wait time, typically assigning a call to the closest or least-burdened elevator.  
- **Priority Handling**: Certain requests (e.g., emergency stops, fire alarms) may override normal scheduling. The system must insert these as high-priority tasks.

## Implementation Considerations
- **Multi-threading**: Each elevator can run on its own thread, continuously processing its queue. Requests arrive concurrently, so concurrency is necessary.  
- **Synchronization**: Shared data (such as an elevator's queue) must be updated in a thread-safe manner (locks, synchronized blocks).  
- **Data Structures**: For each elevator, maintain priority queues for up/down stops to serve floors in sorted order, improving efficiency.  
- **Timing Simulation**: In a simulation, factor in travel time between floors and door open/close durations.

## Performance and Scaling Characteristics
- **Optimized Request Processing**: Use efficient data structures (priority queues) and O(N) or better strategies to select the best elevator.  
- **Scalability**: Works for buildings with many floors and multiple elevators. In large-scale deployments, a hierarchical or distributed controller might be required.  
- **Dynamic Load Adjustment**: Advanced designs learn traffic patterns and anticipate high-load periods, reassigning idle elevators to busy zones.

## Common Pitfalls and How to Avoid Them
- **Race Conditions**: Use proper synchronization to avoid inconsistent states.  
- **Deadlocks**: Carefully design locking so threads do not wait on each other in cycles.  
- **Inefficient Scheduling**: Avoid naive algorithms that cause frequent back-and-forth movement.  
- **Ignoring Safety**: Always check door status and weight capacity before moving; incorporate emergency logic.

## Best Practices for Implementation & Maintenance
- **Modular Design**: Keep scheduling, door logic, and elevator movement in separate classes.  
- **Logging and Monitoring**: Detailed logs help trace errors. Monitor metrics like average wait time or door faults.  
- **Testing and Simulation**: Test concurrency edge cases and large-scale request loads. Use a simulation environment to verify scheduling correctness and concurrency handling.

## How to Discuss This in an Interview
- **Core Entities**: Emphasize the Elevator, Controller, Requests, and DoorSystem, explaining each component’s role.  
- **Scheduling Strategy**: Describe the directional scheduling approach, load balancing, and how priorities are handled.  
- **Concurrency**: Show awareness of multithreading challenges (race conditions, deadlocks) and solutions.  
- **Safety and Real-World Constraints**: Mention door sensors, emergency stops, and weight limits.  
- **Extensibility**: Point out how easily new algorithms or advanced features (like predictive dispatching) can be introduced.

## Example Java Implementation

```java
// Enum for elevator direction
enum Direction {
    UP, DOWN, IDLE
}

// Class to represent an external hall call request
class ExternalRequest {
    private int floor;
    private Direction direction;

    public ExternalRequest(int floor, Direction direction) {
        this.floor = floor;
        this.direction = direction;
    }

    public int getFloor() { return floor; }
    public Direction getDirection() { return direction; }
}

// Class to represent an internal request (destination selected inside elevator)
class InternalRequest {
    private int destinationFloor;

    public InternalRequest(int floor) {
        this.destinationFloor = floor;
    }

    public int getDestinationFloor() { return destinationFloor; }
}

// Elevator class representing a single elevator car
class Elevator {
    int id;
    private int currentFloor;
    private Direction direction;
    DoorSystem door;

    // Queues for pending stops in each direction
    java.util.PriorityQueue<Integer> upQueue;
    java.util.PriorityQueue<Integer> downQueue;

    public Elevator(int id, int startFloor) {
        this.id = id;
        this.currentFloor = startFloor;
        this.direction = Direction.IDLE;
        this.door = new DoorSystem();

        // Min-heap for upQueue (natural order), Max-heap for downQueue (reverse order)
        this.upQueue = new java.util.PriorityQueue<>();
        this.downQueue = new java.util.PriorityQueue<>(java.util.Collections.reverseOrder());
    }

    public int getCurrentFloor() { return currentFloor; }
    public Direction getDirection() { return direction; }
    public boolean isIdle() { return direction == Direction.IDLE; }

    // Add a new stop request to this elevator's queue (thread-safe)
    public synchronized void addStop(int floor) {
        if (floor == currentFloor) return;  // already here
        if (direction == Direction.IDLE) {
            // If idle, choose direction based on the target floor
            direction = (floor > currentFloor) ? Direction.UP : Direction.DOWN;
        }
        if (floor > currentFloor) {
            upQueue.offer(floor);
        } else if (floor < currentFloor) {
            downQueue.offer(floor);
        }
        System.out.println("Elevator " + id + " scheduled stop at floor " + floor);
    }

    // Simulate one step of elevator movement
    public synchronized void moveOneStep() {
        if (direction == Direction.IDLE) return;

        int nextFloor = currentFloor;

        // Handle UP direction
        if (direction == Direction.UP) {
            if (upQueue.isEmpty()) {
                // No more up stops; switch to down if any requests exist
                if (!downQueue.isEmpty()) {
                    direction = Direction.DOWN;
                } else {
                    direction = Direction.IDLE;
                    return;
                }
            }
        }

        if (direction == Direction.UP) {
            nextFloor = currentFloor + 1;
        } else if (direction == Direction.DOWN) {
            // Handle DOWN direction
            if (downQueue.isEmpty()) {
                if (!upQueue.isEmpty()) {
                    direction = Direction.UP;
                    nextFloor = currentFloor + 1;
                } else {
                    direction = Direction.IDLE;
                    return;
                }
            }
            if (direction == Direction.DOWN) {
                nextFloor = currentFloor - 1;
            }
        }

        currentFloor = nextFloor;
        System.out.println("Elevator " + id + " moved to floor " + currentFloor);

        // Check if we reached a requested stop
        if (direction == Direction.UP && !upQueue.isEmpty() && currentFloor == upQueue.peek()) {
            upQueue.poll();
            arriveAtFloor();
        } else if (direction == Direction.DOWN && !downQueue.isEmpty() && currentFloor == downQueue.peek()) {
            downQueue.poll();
            arriveAtFloor();
        }
    }

    // Handle arriving at a floor
    private void arriveAtFloor() {
        System.out.println("Elevator " + id + " stopping at floor " + currentFloor);
        door.open();
        // Passengers would enter/exit here in a real system
        door.close();

        // Adjust direction if no more requests in the current queue
        if (direction == Direction.UP && upQueue.isEmpty()) {
            direction = downQueue.isEmpty() ? Direction.IDLE : Direction.DOWN;
        } else if (direction == Direction.DOWN && downQueue.isEmpty()) {
            direction = upQueue.isEmpty() ? Direction.IDLE : Direction.UP;
        }
    }
}

// DoorSystem class for elevator doors
class DoorSystem {
    private boolean open;

    public DoorSystem() {
        this.open = false;
    }

    public void open() {
        open = true;
        System.out.println("Doors opening");
    }

    public void close() {
        open = false;
        System.out.println("Doors closing");
    }

    public boolean isOpen() { return open; }
}

// ElevatorController class to manage multiple elevators
class ElevatorController {
    private java.util.List<Elevator> elevators;

    public ElevatorController(int numElevators) {
        elevators = new java.util.ArrayList<>();
        for (int i = 1; i <= numElevators; i++) {
            elevators.add(new Elevator(i, 0)); // start at floor 0
        }
    }

    // Handle an external hall call
    public synchronized void handleExternalRequest(int floor, Direction direction) {
        Elevator bestElevator = null;
        int bestScore = Integer.MAX_VALUE;

        // Try to find a suitable elevator
        for (Elevator e : elevators) {
            if (e.isIdle()) {
                int distance = Math.abs(e.getCurrentFloor() - floor);
                if (distance < bestScore) {
                    bestScore = distance;
                    bestElevator = e;
                }
            } else if (e.getDirection() == direction) {
                // Elevator going the same direction
                if ((direction == Direction.UP && e.getCurrentFloor() <= floor)
                    || (direction == Direction.DOWN && e.getCurrentFloor() >= floor)) {
                    int distance = Math.abs(e.getCurrentFloor() - floor);
                    if (distance < bestScore) {
                        bestScore = distance;
                        bestElevator = e;
                    }
                }
            }
        }

        if (bestElevator == null) {
            // Fallback: pick the elevator with the fewest pending stops
            int minLoad = Integer.MAX_VALUE;
            for (Elevator e : elevators) {
                int load = e.isIdle() ? 0 : e.upQueue.size() + e.downQueue.size();
                if (load < minLoad) {
                    minLoad = load;
                    bestElevator = e;
                }
            }
        }

        System.out.println("Controller: assigning elevator " + (bestElevator != null ? bestElevator.id : "none")
                           + " to floor " + floor);
        if (bestElevator != null) {
            bestElevator.addStop(floor);
        }
    }

    // Handle an internal request
    public synchronized void handleInternalRequest(int elevatorId, int destinationFloor) {
        if (elevatorId <= 0 || elevatorId > elevators.size()) return;
        Elevator e = elevators.get(elevatorId - 1);
        e.addStop(destinationFloor);
    }

    // Simulate one time step in the system
    public void step() {
        for (Elevator e : elevators) {
            e.moveOneStep();
        }
    }

    // Run the simulation for a given number of steps
    public void runSteps(int steps) {
        for (int i = 0; i < steps; i++) {
            step();
        }
    }

    // Main method to demonstrate usage
    public static void main(String[] args) {
        ElevatorController controller = new ElevatorController(2);
        // External calls
        controller.handleExternalRequest(5, Direction.UP);
        controller.handleExternalRequest(2, Direction.UP);
        controller.handleExternalRequest(8, Direction.DOWN);

        controller.runSteps(10);

        // Internal request
        controller.handleInternalRequest(1, 7);
        controller.runSteps(5);
    }
}
```

This example demonstrates:
- Separate classes for core components (`Elevator`, `ElevatorController`, `DoorSystem`).
- Two priority queues (up/down) per elevator for directional scheduling.
- Thread-safe methods (`synchronized` blocks).
- A simple nearest-car logic in `handleExternalRequest`.

It can be extended to handle additional details (safety overrides, advanced scheduling, door sensors, or concurrency with multiple threads).
