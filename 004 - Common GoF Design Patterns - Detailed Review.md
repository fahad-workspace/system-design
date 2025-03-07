# Common GoF Design Patterns: Detailed Review  

## Table of Contents
- [Singleton Pattern](#singleton-pattern)
- [Factory Pattern](#factory-pattern)
- [Builder Pattern](#builder-pattern)
- [Observer Pattern](#observer-pattern)
- [Strategy Pattern](#strategy-pattern)
- [Adapter Pattern](#adapter-pattern)
- [Decorator Pattern](#decorator-pattern)

---

## Singleton Pattern
**Explanation:**  
The Singleton is a creational pattern ensuring a class has only one instance and providing a global point of access to it. This is useful for shared resources like configuration, caches, or logging where exactly one object should coordinate actions. The class itself controls its instantiation (often by making the constructor private and exposing a static method) so no more than one instance ever exists.  

**Key Principles:**  
- Centralizes and controls object creation, enforcing a single shared instance.  
- Can improve performance or consistency by reusing one object instead of many.  
- Somewhat breaks Single Responsibility by managing its own lifecycle.  
- Introduces a global state that can couple parts of the code.

**Real-World Use Cases:**  
- Configuration or Settings Manager (one object holds application-wide settings).  
- Logger (single instance ensures consistent logging).  
- Resource Pool/Connection Manager (e.g., database connection pool).  
- Factories or Registries (sometimes a single factory instance).

**Java Implementation Example:**
```java
class ConfigManager {
    private static volatile ConfigManager instance;
    private Properties config;

    private ConfigManager() {
        // load configuration settings
        config = new Properties();
        config.put("app.name", "MyApp");
    }

    public static ConfigManager getInstance() {
        if (instance == null) {
            synchronized (ConfigManager.class) {
                if (instance == null) {
                    instance = new ConfigManager();
                }
            }
        }
        return instance;
    }

    public String getSetting(String key) {
        return config.getProperty(key);
    }
}
```
- Uses double-checked locking with a `volatile` instance for thread safety.  
- An alternative is an enum-based Singleton, which is thread-safe by default.

**Performance and Scalability:**  
- Minimal overhead—just one object.  
- If `getInstance()` is synchronized or in heavy use, it can be a bottleneck.  
- In distributed systems, each JVM has its own Singleton unless an external coordination mechanism is used.

**Common Pitfalls & Avoidance:**  
- **Global State & Coupling:** Overuse leads to untestable code. Use singletons sparingly.  
- **Thread-Safety Issues:** Ensure proper synchronization.  
- **Lifecycle Management:** Singleton often lives for the program’s duration.  
- **Misuse as God Object:** Keep responsibilities focused.

**Best Practices:**  
- Use an enum for simplicity.  
- Distinguish between lazy and eager instantiation.  
- Hide the constructor and provide a clear `getInstance()`.  
- Consider using dependency injection frameworks instead of Singletons for large systems.

**Interview Tips:**  
Emphasize knowing proper use cases and drawbacks (global state). Mention the thread-safe aspects, and how you’ve refactored or used singletons carefully when a single instance was truly needed.

---

## Factory Pattern
**Explanation:**  
A creational pattern that abstracts the object creation process. Instead of calling constructors directly, you call a factory method that decides which subclass or implementation to instantiate. This promotes flexibility in adding new types.  

**Key Principles:**  
- Decouples the client from specific implementations.  
- Supports the Open/Closed Principle by letting you add new product types without changing client code.  
- Encapsulates creation logic in one place.

**Real-World Use Cases:**  
- Toolkit/Framework Object Creation (GUI factories).  
- Database Drivers (e.g., get connections without knowing vendor specifics).  
- Parsing or Content Creation (creating parser instances).  
- Plugin Systems (factory to load appropriate modules).

**Java Implementation Example (Simple Factory):**
```java
interface Shape {
    void draw();
}

class Circle implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing a Circle");
    }
}

class Rectangle implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing a Rectangle");
    }
}

class ShapeFactory {
    public static Shape getShape(String type) {
        if ("CIRCLE".equalsIgnoreCase(type)) {
            return new Circle();
        } else if ("RECTANGLE".equalsIgnoreCase(type)) {
            return new Rectangle();
        }
        throw new IllegalArgumentException("Unknown shape type: " + type);
    }
}
```
Usage:
```java
Shape s1 = ShapeFactory.getShape("circle");
s1.draw(); // "Drawing a Circle"

Shape s2 = ShapeFactory.getShape("rectangle");
s2.draw(); // "Drawing a Rectangle"
```
**Performance and Scalability:**  
- Minimal overhead (an extra method call).  
- If the factory logic is large (like a big if-else chain), maintenance can be tricky. Using registration or polymorphism helps.

**Common Pitfalls & Avoidance:**  
- **Overusing Factories:** Use them when you anticipate variations.  
- **Complex Factory Logic:** Break down via maps or polymorphic approaches instead of giant switches.  
- **Hiding Dependencies:** Document required resources.  
- **Testing Difficulty:** Abstract the factory or pass a mock if needed.

**Best Practices:**  
- Program to interfaces.  
- Keep the factory focused on creation.  
- Use Abstract Factory for families of related objects.  
- If many product variations, consider dynamic registration (map of type → constructor reference).

**Interview Tips:**  
Explain the difference between simple Factory, Factory Method, and Abstract Factory. Give real examples such as number formatters or database connections. Emphasize how it supports open/closed principle.

---

## Builder Pattern
**Explanation:**  
A creational pattern that separates the construction of a complex object from its representation, allowing step-by-step building. Useful when objects have many optional parameters or require a multi-step creation process.  

**Key Principles:**  
- Avoids “telescoping constructors” by allowing a fluent step-by-step approach.  
- Encapsulates the assembly logic outside the main class, often producing immutable objects.  
- Follows Single Responsibility by keeping construction details in the builder.

**Real-World Use Cases:**  
- Objects with many optional parameters (e.g., large configuration classes).  
- Immutable object construction (commonly seen in frameworks).  
- Fluent DSL for assembling complex objects.

**Java Implementation Example:**
```java
class Computer {
    private String cpu;
    private int ram;
    private boolean graphicsCard;
    private String os;

    private Computer(Builder builder) {
        this.cpu = builder.cpu;
        this.ram = builder.ram;
        this.graphicsCard = builder.graphicsCard;
        this.os = builder.os;
    }

    public static class Builder {
        private String cpu;
        private int ram;
        private boolean graphicsCard = false;
        private String os = "Linux";

        public Builder(String cpu, int ram) {
            this.cpu = cpu;
            this.ram = ram;
        }

        public Builder withGraphicsCard(boolean value) {
            this.graphicsCard = value;
            return this;
        }

        public Builder withOS(String os) {
            this.os = os;
            return this;
        }

        public Computer build() {
            return new Computer(this);
        }
    }
}

// Usage
Computer pc = new Computer.Builder("Intel i7", 16)
                   .withGraphicsCard(true)
                   .withOS("Windows 11")
                   .build();
```
**Performance and Scalability:**  
- Minor overhead from creating the builder object.  
- Extremely beneficial for clarity and maintainability, especially as the object grows in complexity.

**Common Pitfalls & Avoidance:**  
- **Inconsistent State:** Mark required fields in the builder’s constructor; validate in `build()`.  
- **Verbose Implementation:** Often writing builder code is repetitive; can use code generation.  
- **Confusion with Factory:** Builder is for assembling a single complex object, whereas Factory is for choosing an implementation.

**Best Practices:**  
- Make the final object immutable.  
- Use method chaining (fluent API).  
- Initialize defaults in the builder.  
- Use it when many fields exist, especially optional ones.

**Interview Tips:**  
Mention how it solves the telescoping constructor problem. Reference usage in libraries (like the widely known “Effective Java” approach). Show that you grasp the difference from factories.

---

## Observer Pattern
**Explanation:**  
A behavioral pattern where an object (the Subject) maintains a list of dependents (Observers) and notifies them automatically when its state changes. Also known as publish-subscribe.  

**Key Principles:**  
- Loose coupling: the subject doesn’t need to know how observers handle the update.  
- Aligns with Open/Closed: you can add new observers without modifying the subject.  
- Subject and observers communicate via a simple interface (e.g., an `update()` method).

**Real-World Use Cases:**  
- GUI frameworks (events are observer-based).  
- Model-View-Controller (Model notifies Views of data changes).  
- Messaging/Notification systems, logging frameworks.  
- Reactive programming (subscribe to streams).

**Java Implementation Example:**
```java
interface Observer {
    void update(float temperature, float humidity);
}

class WeatherStation {
    private List<Observer> observers = new ArrayList<>();
    private float temperature;
    private float humidity;

    public void addObserver(Observer o) {
        observers.add(o);
    }

    public void removeObserver(Observer o) {
        observers.remove(o);
    }

    public void setMeasurements(float temp, float hum) {
        this.temperature = temp;
        this.humidity = hum;
        notifyObservers();
    }

    private void notifyObservers() {
        for (Observer o : observers) {
            o.update(temperature, humidity);
        }
    }
}

class Display implements Observer {
    private String name;
    public Display(String name) { this.name = name; }

    @Override
    public void update(float temperature, float humidity) {
        System.out.println(name + " Display -> Temp:" + temperature + ", Humidity:" + humidity);
    }
}

// Usage:
WeatherStation station = new WeatherStation();
Observer lcdDisplay = new Display("LCD");
Observer mobileApp = new Display("MobileApp");

station.addObserver(lcdDisplay);
station.addObserver(mobileApp);

station.setMeasurements(30.4f, 65.0f);
// LCD Display -> Temp:30.4, Humidity:65.0
// MobileApp Display -> Temp:30.4, Humidity:65.0
```
**Performance and Scalability:**  
- O(n) notifications for n observers.  
- If observers are slow or if there are many, it can slow the subject unless you use asynchronous queues.

**Common Pitfalls & Avoidance:**  
- **Memory Leaks:** Observers must be removed if no longer needed.  
- **Update Storms:** Consider throttling or checking if data changed before notifying.  
- **Threading Issues:** If multiple threads modify the observers list or subject state, synchronize or use thread-safe collections.

**Best Practices:**  
- Use consistent naming (like `addXyzListener`).  
- Decide whether data is pushed or pulled.  
- Ensure you clean up observers in object lifecycle.

**Interview Tips:**  
Mention how it appears in UI frameworks or an event bus. Emphasize decoupling and the publish-subscribe concept. Also discuss pitfalls like memory leaks or potential for complex event cascades.

---

## Strategy Pattern
**Explanation:**  
A behavioral pattern that defines a family of algorithms, encapsulates each one, and makes them interchangeable. The client (context) can pick which strategy to use at runtime.  

**Key Principles:**  
- Encourages composition over inheritance: the context has a strategy, rather than being forced to subclass for each algorithm.  
- Adheres to Open/Closed: new strategies can be added without modifying the context.  
- Separates the context’s usage logic from the actual algorithm.

**Real-World Use Cases:**  
- Sorting with custom comparators (the comparator is a strategy).  
- Encryption/Compression (different algorithms as strategies).  
- Payment processing (credit card, PayPal, etc. as distinct strategies).  
- Pathfinding or AI behavior (choose among different approaches).

**Java Implementation Example:**
```java
interface CompressionStrategy {
    void compressFiles(List<String> files);
}

class ZipCompressionStrategy implements CompressionStrategy {
    @Override
    public void compressFiles(List<String> files) {
        System.out.println("Compressing " + files.size() + " files using ZIP...");
    }
}

class SevenZCompressionStrategy implements CompressionStrategy {
    @Override
    public void compressFiles(List<String> files) {
        System.out.println("Compressing " + files.size() + " files using 7z...");
    }
}

class FileCompressor {
    private CompressionStrategy strategy;
    public FileCompressor(CompressionStrategy strategy) {
        this.strategy = strategy;
    }
    public void setStrategy(CompressionStrategy strategy) {
        this.strategy = strategy;
    }
    public void compressFiles(List<String> files) {
        strategy.compressFiles(files);
    }
}

// Usage:
List<String> files = List.of("file1.txt", "file2.txt");
FileCompressor compressor = new FileCompressor(new ZipCompressionStrategy());
compressor.compressFiles(files);

compressor.setStrategy(new SevenZCompressionStrategy());
compressor.compressFiles(files);
```
**Performance and Scalability:**  
- Minimal overhead for the extra method call.  
- Strategy can improve performance by choosing the right algorithm for the situation.

**Common Pitfalls & Avoidance:**  
- **Overusing Strategy:** If there's only one algorithm, it’s unnecessary abstraction.  
- **Too Many Classes:** If you have dozens of tiny strategy classes, manage them carefully or consider using enums/lambdas for simpler ones.  
- **Context-Strategy Coupling:** The context should know nothing about concrete strategy details.  
- **Inconsistent Interfaces:** All strategies must fit the same interface.

**Best Practices:**  
- Keep strategies stateless or store minimal internal state.  
- If it’s a single-method interface, consider lambdas in modern Java for concise strategy objects.  
- Avoid giant `if` to pick strategy; consider a factory or config injection.

**Interview Tips:**  
Explain how it removes the need for big conditional logic. Show real examples like sorting or compression. Distinguish it from Factory (Strategy is about different algorithms for the same task; Factory is about which class to create). Acknowledge you only use it when multiple interchangeable behaviors are genuinely needed.

---

## Adapter Pattern
**Explanation:**  
A structural pattern that allows objects with incompatible interfaces to collaborate by inserting an intermediary that translates one interface into another. The adapter implements a target interface while holding a reference to an adaptee.  

**Key Principles:**  
- Converts the interface of a class into one the client expects.  
- Uses composition or inheritance to delegate calls.  
- Promotes reuse by enabling existing classes to fit new contexts without changes.

**Real-World Use Cases:**  
- Integrating legacy or third-party code with a new interface.  
- Bridging different data formats.  
- Converting old API calls to match a new API.  
- Java examples include `InputStreamReader` (adapts `InputStream` to a `Reader`).

**Java Implementation Example:**
```java
class Thermometer {
    public double getTemperatureF() {
        return 98.6;
    }
}

interface TemperatureSensor {
    double getTemperatureC();
}

class ThermometerAdapter implements TemperatureSensor {
    private Thermometer thermometer;

    public ThermometerAdapter(Thermometer therm) {
        this.thermometer = therm;
    }

    @Override
    public double getTemperatureC() {
        double f = thermometer.getTemperatureF();
        return (f - 32) * 5.0 / 9.0;
    }
}
```
**Performance and Scalability:**  
- Tiny overhead of a wrapper method call.  
- Can keep adding more adapters for new integrations.  
- Organize them well to avoid confusion if many exist.

**Common Pitfalls & Avoidance:**  
- **Unneeded Complexity:** Only use when interfaces truly mismatch.  
- **Partial Adapting:** Ensure the adapter fully implements the target interface or document unsupported calls.  
- **Performance of Conversion:** If conversion is expensive, it can become a bottleneck.  
- **Multiple Layers of Adapters:** Try not to chain too many adapters.

**Best Practices:**  
- Use composition over inheritance (object adapter).  
- Clearly name adapter classes (e.g., `ThermometerAdapter`).  
- Document the adaptation (what does it convert?).  
- Keep the adapter as a minimal glue layer.

**Interview Tips:**  
Compare it to a power plug converter. Mention typical usage in bridging mismatched APIs. Reference JDK examples like `InputStreamReader`. Emphasize how you can reuse legacy code by writing a small adapter rather than rewriting everything.

---

## Decorator Pattern
**Explanation:**  
A structural pattern that dynamically attaches additional responsibilities to an object without altering its structure. Decorators wrap a component, adding behavior before or after delegating to it. Both the decorator and the original object share the same interface, so they are interchangeable.  

**Key Principles:**  
- Follows Open/Closed by letting you introduce new decorators for new features.  
- Uses composition (each decorator has-a component) instead of inheritance.  
- You can stack multiple decorators, each adding its own functionality.

**Real-World Use Cases:**  
- **Java I/O Streams** (e.g., `BufferedInputStream` wraps an `InputStream` to add buffering).  
- GUI items that can be decorated with scrollbars, borders, etc.  
- Adding logging or caching to an object (wrap it with a decorator that does extra work).  
- Collections: `Collections.unmodifiableList(list)` returns a decorator that disallows modification.

**Java Implementation Example (Notifier):**
```java
interface Notifier {
    void send(String message);
}

class BasicNotifier implements Notifier {
    @Override
    public void send(String message) {
        System.out.println("Basic Notifier: Sending message: " + message);
    }
}

abstract class NotifierDecorator implements Notifier {
    protected Notifier wrappee;
    public NotifierDecorator(Notifier notifier) {
        this.wrappee = notifier;
    }
    @Override
    public void send(String message) {
        wrappee.send(message);
    }
}

class SMSNotifier extends NotifierDecorator {
    public SMSNotifier(Notifier notifier) {
        super(notifier);
    }
    @Override
    public void send(String message) {
        super.send(message);
        System.out.println("SMS Notifier: Sending SMS with message: " + message);
    }
}

class SlackNotifier extends NotifierDecorator {
    public SlackNotifier(Notifier notifier) {
        super(notifier);
    }
    @Override
    public void send(String message) {
        super.send(message);
        System.out.println("Slack Notifier: Sending Slack message: " + message);
    }
}

// Usage
Notifier notifier = new BasicNotifier();
notifier = new SMSNotifier(notifier);
notifier = new SlackNotifier(notifier);
notifier.send("Server is down!");
```
The output sequence shows all wrapped behaviors.

**Performance and Scalability:**  
- Each decorator adds one more level of indirection. Typically negligible overhead.  
- Decorators scale well for combining behaviors, though too many layers can complicate debugging.

**Common Pitfalls & Avoidance:**  
- **Too Many Small Classes:** Manage them properly to avoid confusion.  
- **Order Sensitivity:** The order of wrapping can matter if behaviors depend on each other.  
- **Mixing Incompatible Behaviors:** Some combos might conflict.  
- **Debugging Complexity:** Depth of wrapping can make debugging stack traces longer.

**Best Practices:**  
- Keep each decorator focused on a single responsibility (logging, caching, etc.).  
- Use an abstract decorator to avoid code duplication.  
- Document how decorators can be stacked.  
- Know the differences from Adapter (changes interface) or Proxy (access control).

**Interview Tips:**  
Highlight the Java I/O Streams example. Emphasize the ability to add features at runtime without creating a complex subclass hierarchy. Point out how it differs from Proxy or Adapter. Mention potential confusion if you nest too many decorators.
