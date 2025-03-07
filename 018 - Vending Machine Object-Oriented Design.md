# Vending Machine Object-Oriented Design

## Table of Contents
- [Design Overview](#design-overview)
- [Component Responsibilities](#component-responsibilities)
- [Class Implementations (Java)](#class-implementations-java)
  - [VendingMachine Class](#vendingmachine-class)
  - [Slot Class](#slot-class)
  - [Item Class](#item-class)
  - [Coin (Denomination)](#coin-denomination)
  - [InventoryManager Class](#inventorymanager-class)
  - [State Interface and Concrete States](#state-interface-and-concrete-states)
- [Interactions and Workflow Example](#interactions-and-workflow-example)
- [Design Principles and Patterns Applied](#design-principles-and-patterns-applied)
- [Extensibility and Future Enhancements](#extensibility-and-future-enhancements)

## Design Overview
We will design a vending machine system using an object-oriented approach, emphasizing encapsulation, inheritance, and polymorphism. The solution will follow SOLID principles and use the State design pattern to manage the machine's behavior in different states (Idle, Selecting, Dispensing). The key components of the system include:

- **VendingMachine** – The main context class that coordinates interactions and holds the current state, inventory, and balance.  
- **Slot** – Represents a physical compartment in the machine, each holding a certain Item type and its quantity.  
- **Item** – Represents a product with a name, price, and quantity (stock count).  
- **Coin** – Represents currency denominations (e.g., coins of certain values).  
- **InventoryManager** – Manages the stock of items in the machine (checking stock levels, updating counts after dispensing).  
- **State** – An interface representing the machine's state; concrete implementations (IdleState, SelectingState, DispensingState) define behavior for actions in each state.

Using the State pattern allows the vending machine to alter its behavior based on its current state, avoiding complex conditional logic. Each state class will handle actions like coin insertion, product selection, and dispensing appropriately, and trigger transitions to other states as needed.

## Component Responsibilities

- **VendingMachine**: Maintains the current state, the inserted balance, and interfaces with the inventory. It delegates actions (insert coin, select product, dispense) to the current State. It also manages transitions between states and overall transaction flow (including calculating change and resetting for the next user).

- **Slot**: Encapsulates an item type in the machine and how many of that item are available. For example, a slot might be identified by a code (like "A1") and contain a specific snack with its price and current stock count.

- **Item**: Contains product information – a name (e.g., "Soda"), a price, and a stock count. The stock count represents how many units of this item are available (typically within its slot). In this design, each slot has one Item type, and the Item’s stock corresponds to the quantity in that slot.

- **Coin**: Represents a coin denomination. We can model this as an enum for simplicity, where each constant has a value (for example, Penny = 1 cent, Nickel = 5 cents, etc.). The vending machine will accept certain coins and sum their values to track the current balance.

- **InventoryManager**: Responsible for managing items and slots. It keeps a collection of all slots in the machine, each with its item and stock. It provides methods to check if a product is in stock, decrement stock when an item is dispensed, and possibly restock items. This separation follows the Single Responsibility Principle by isolating inventory logic from the main machine logic.

- **State (Idle, Selecting, Dispensing)**: Defines the interface for different states of the vending machine and the behavior of actions in those states.  

  - **IdleState**: No transaction in progress – waiting for user to insert coins (and/or make a selection).  
  - **SelectingState**: The machine has received coins (balance > 0) and is ready for or awaiting product selection. The user may continue inserting coins or choose a product.  
  - **DispensingState**: A purchase has been made (sufficient funds provided for a selected item) and the machine is dispensing the item and any change. During this state, no other coins or selections should be accepted until the current transaction completes.

## Class Implementations (Java)

### VendingMachine Class
The `VendingMachine` class is the central context. It holds references to the state instances and delegates actions to the current state. It also contains the machine’s balance (the total value of inserted coins) and the currently selected product (if any), as well as the `InventoryManager`. After a transaction, it resets to the Idle state.

```java
// VendingMachine.java
public class VendingMachine {

    private InventoryManager inventory;
    private int balance;            // current balance in cents (coins inserted)
    private String selectedCode;    // code of currently selected item (if any)

    // State instances for state pattern
    private State idleState;
    private State selectingState;
    private State dispensingState;
    private State currentState;

    public VendingMachine() {
        this.inventory = new InventoryManager();
        // Initialize state instances
        this.idleState = new IdleState();
        this.selectingState = new SelectingState();
        this.dispensingState = new DispensingState();
        this.currentState = idleState;  // start in Idle
        this.balance = 0;
        this.selectedCode = null;
    }

    // Action methods delegate to the current state's behavior:
    public void insertCoin(Coin coin) throws Exception {
        currentState.insertCoin(this, coin);
    }

    public void selectProduct(String code) throws Exception {
        currentState.selectProduct(this, code);
        // If selection was successful and state changed to Dispensing, automatically dispense
        if (currentState == dispensingState) {
            currentState.dispense(this);
        }
    }

    public void dispenseProduct() throws Exception {
        // In some designs, a separate method or button press triggers dispensing
        currentState.dispense(this);
    }

    public void cancelTransaction() {
        // Allow user to cancel and get refund
        if (balance > 0) {
            // return coins (for simplicity, just resetting balance here)
        }
        balance = 0;
        selectedCode = null;
        currentState = idleState;
    }

    // Package-private or public helper methods for State classes to use:
    void addBalance(int amount) { this.balance += amount; }
    void deductBalance(int amount) { this.balance -= amount; }
    int getBalance() { return balance; }
    void resetBalance() { this.balance = 0; }

    InventoryManager getInventoryManager() { return inventory; }
    String getSelectedCode() { return selectedCode; }
    void setSelectedCode(String code) { this.selectedCode = code; }

    // Methods to change or get current state (called by State implementations):
    void setCurrentState(State state) { this.currentState = state; }
    State getIdleState() { return idleState; }
    State getSelectingState() { return selectingState; }
    State getDispensingState() { return dispensingState; }
}
```

**Design notes**: The `VendingMachine` uses composition to include an `InventoryManager` and multiple `State` instances. It starts in an `Idle` state with zero balance. The `insertCoin` and `selectProduct` methods simply forward the call to the current state's corresponding method. After a product is selected, if the state transitions to `DispensingState`, we immediately call `dispense()` on the state to dispense the item. A `cancelTransaction` method is provided to illustrate handling a user cancellation (refund logic would go there). The helper methods like `addBalance` and `setSelectedCode` are used by state classes to modify the `VendingMachine`'s context.

### Slot Class
The `Slot` class represents a physical slot in the vending machine containing items. Each slot has an identifier code (like "A1", "B2", or a number) and an associated `Item`. Typically, one slot holds a single type of item. The quantity of that item available in the slot is tracked by the `Item`’s stock count.

```java
// Slot.java
public class Slot {
    private String code;   // e.g., "A1", "B2" to identify the slot
    private Item item;     // The product in this slot (with its price and stock)

    public Slot(String code, Item item) {
        this.code = code;
        this.item = item;
    }
    public String getCode() { return code; }
    public Item getItem() { return item; }
}
```

### Item Class
The `Item` class represents a product that can be dispensed by the vending machine. It contains a name (for identification/display), a price, and a stock count (how many are available, usually in its slot). When an item is dispensed, the stock count will be decremented via the `InventoryManager`.

```java
// Item.java
public class Item {
    private String name;
    private int price;    // price in cents (to avoid floating point issues)
    private int stock;    // number of items available in stock

    public Item(String name, int price, int stock) {
        this.name = name;
        this.price = price;
        this.stock = stock;
    }
    public String getName() { return name; }
    public int getPrice() { return price; }
    public int getStock() { return stock; }
    public void setStock(int stock) { this.stock = stock; }
}
```

**Design notes**: We use an integer for `price` (e.g., in cents) to maintain precision. The `Item` class is straightforward and focuses on item data. It could be extended to include additional info (like an ID or description). This class does not manage where it’s placed or how it’s selected, preserving the Single Responsibility Principle.

### Coin (Denomination)
`Coin` can be modeled as an enum since the set of denominations is fixed and known (for example, penny, nickel, dime, quarter in US currency). Each coin has a value. The vending machine will accept these coins and update the balance accordingly.

```java
// Coin.java
public enum Coin {
    PENNY(1),    // 1 cent
    NICKEL(5),   // 5 cents
    DIME(10),    // 10 cents
    QUARTER(25); // 25 cents

    private final int value;  // value of the coin in cents

    Coin(int value) {
        this.value = value;
    }
    public int getValue() {
        return value;
    }
}
```

**Design notes**: Using an enum for `Coin` makes it easy to add new denominations if needed (just add another constant) and prevents invalid coin values. In an extended design, we might also have a coin inventory to keep track of how many coins of each type are in the machine for giving change.

### InventoryManager Class
The `InventoryManager` encapsulates all inventory-related operations. It contains the collection of slots (and thereby items) in the machine. It provides methods to check availability and update stock levels when items are dispensed or restocked.

```java
// InventoryManager.java
public class InventoryManager {
    private Map<String, Slot> slots = new HashMap<>();

    public void addSlot(String code, Item item) {
        // Initialize a slot with a product and add to inventory
        Slot slot = new Slot(code, item);
        slots.put(code, slot);
    }

    public Slot getSlot(String code) {
        return slots.get(code);
    }

    public boolean isInStock(String code) {
        Slot slot = slots.get(code);
        if (slot == null) return false;
        return slot.getItem().getStock() > 0;
    }

    public void decrementStock(String code) throws Exception {
        Slot slot = slots.get(code);
        if (slot == null) {
            throw new Exception("Invalid slot code: " + code);
        }
        Item item = slot.getItem();
        if (item.getStock() <= 0) {
            throw new Exception("Product " + code + " is out of stock");
        }
        item.setStock(item.getStock() - 1);
    }

    // (Optional) method to restock an item
    public void restockItem(String code, int quantity) throws Exception {
        Slot slot = slots.get(code);
        if (slot == null) throw new Exception("Invalid slot code");
        Item item = slot.getItem();
        item.setStock(item.getStock() + quantity);
    }
}
```

**Design notes**: The `InventoryManager` uses a `Map<String, Slot>` to map slot codes to `Slot` objects for quick lookup. The `isInStock` method checks if the given slot exists and has at least one item available. The `decrementStock` method reduces the count of an item after a successful vend, throwing exceptions for invalid codes or out-of-stock conditions.

### State Interface and Concrete States
The `State` interface defines the actions that can occur in any state: inserting a coin, selecting a product, and dispensing. Each concrete state class implements these actions according to the rules of that state. This design uses polymorphism to let the `VendingMachine` behave differently depending on its current state object. State transitions are handled by changing the `VendingMachine`'s current state within these methods.

```java
// State.java
public interface State {
    void insertCoin(VendingMachine vm, Coin coin) throws Exception;
    void selectProduct(VendingMachine vm, String productCode) throws Exception;
    void dispense(VendingMachine vm) throws Exception;
}
```

```java
// IdleState.java
public class IdleState implements State {

    @Override
    public void insertCoin(VendingMachine vm, Coin coin) {
        // Accept coin and update balance
        vm.addBalance(coin.getValue());
        // Transition to Selecting state since we now have credit
        vm.setCurrentState(vm.getSelectingState());
    }

    @Override
    public void selectProduct(VendingMachine vm, String productCode) throws Exception {
        // Cannot select without inserting money first
        throw new Exception("Please insert coins before selecting a product.");
    }

    @Override
    public void dispense(VendingMachine vm) throws Exception {
        // Dispense action is not valid in Idle state
        throw new Exception("No product selected to dispense.");
    }
}
```

```java
// SelectingState.java
public class SelectingState implements State {

    @Override
    public void insertCoin(VendingMachine vm, Coin coin) {
        // Accept additional coins
        vm.addBalance(coin.getValue());
        // Stay in Selecting state
    }

    @Override
    public void selectProduct(VendingMachine vm, String productCode) throws Exception {
        // Check if the product exists and is in stock
        if (!vm.getInventoryManager().isInStock(productCode)) {
            throw new Exception("Sorry, the selected item is out of stock.");
        }
        Slot slot = vm.getInventoryManager().getSlot(productCode);
        if (slot == null) {
            throw new Exception("Invalid selection code.");
        }
        int price = slot.getItem().getPrice();
        // Verify sufficient funds
        if (vm.getBalance() < price) {
            throw new Exception("Insufficient funds. Please insert more coins to buy this item.");
        }
        // Sufficient funds provided – proceed to dispense
        vm.setSelectedCode(productCode);
        vm.setCurrentState(vm.getDispensingState());
    }

    @Override
    public void dispense(VendingMachine vm) throws Exception {
        // User must select a product first
        throw new Exception("No product selected. Please make a selection first.");
    }
}
```

```java
// DispensingState.java
public class DispensingState implements State {

    @Override
    public void insertCoin(VendingMachine vm, Coin coin) throws Exception {
        // Machine is currently dispensing; do not accept coins
        throw new Exception("Cannot insert coin during dispensing. Please wait.");
    }

    @Override
    public void selectProduct(VendingMachine vm, String productCode) throws Exception {
        // Already dispensing the current selection
        throw new Exception("Already dispensing an item. Please wait.");
    }

    @Override
    public void dispense(VendingMachine vm) throws Exception {
        // Complete the transaction: dispense item and return change
        String code = vm.getSelectedCode();
        if (code == null) {
            throw new Exception("No item selected to dispense.");
        }
        // 1. Deduct the item from inventory
        vm.getInventoryManager().decrementStock(code);
        Slot slot = vm.getInventoryManager().getSlot(code);
        Item item = slot.getItem();
        int price = item.getPrice();

        // 2. Deduct the price from the balance
        vm.deductBalance(price);

        // 3. Calculate change
        int change = vm.getBalance();
        if (change > 0) {
            // Prepare change in coins (pseudo-code for demonstration)
            // Example: collect coins from largest to smallest
            // Return them or display the info
        }

        // 4. Dispense the item (in a real machine, trigger mechanical release)
        System.out.println("Dispensing " + item.getName());
        if (change > 0) {
            System.out.println("Returning change: " + change + " cents");
        }

        // 5. Reset machine for next customer
        vm.resetBalance();
        vm.setSelectedCode(null);
        vm.setCurrentState(vm.getIdleState());
    }
}
```

**Design notes**:  
- **IdleState**: Only accepts coin insertion. Selecting a product or dispensing in this state is invalid because there’s no credit and no selection yet.  
- **SelectingState**: Accepts additional coins and allows product selection. It checks inventory and balance, transitioning to `DispensingState` if enough funds and stock are available. Otherwise, it throws exceptions for out-of-stock or insufficient funds.  
- **DispensingState**: Finalizes the purchase. It will not accept new coins or selections while dispensing. After decrementing stock, deducting the price, and returning change, it resets the machine to `IdleState`.  

## Interactions and Workflow Example
1. **Idle**: The machine starts in IdleState. When a customer inserts a coin, the machine transitions to SelectingState.  
2. **Selecting**: The user can insert more coins or select a product. If the item is in stock and the user has inserted enough funds, the machine transitions to DispensingState.  
3. **Dispensing**: The machine dispenses the product and any change, then resets and returns to IdleState.  

If an error occurs (out-of-stock or insufficient funds), an exception is thrown and the machine remains in SelectingState (the user can insert more coins, cancel, or choose another item).

## Design Principles and Patterns Applied
- **Single Responsibility Principle (SRP)**: Each class has a focused responsibility.  
- **Open/Closed Principle (OCP)**: The design is open to extension but closed to modification. New states or coin types can be added without altering existing code.  
- **Liskov Substitution Principle (LSP)**: All concrete State classes implement the same interface, so they’re interchangeable in the `VendingMachine`.  
- **Interface Segregation Principle (ISP)**: The `State` interface defines a small set of actions. Each state implements only what’s relevant or throws an exception if the action is invalid.  
- **Dependency Inversion Principle (DIP)**: High-level modules (like `VendingMachine`) depend on abstractions (like `State`), not concrete implementations.  
- **Encapsulation**: Data fields are private, and changes happen via well-defined methods.  
- **Polymorphism**: The `VendingMachine` calls methods on the current `State` object, and the actual behavior depends on which state is active.  
- **State Design Pattern**: Manages the machine’s modes explicitly. Each state class knows how to respond to actions in that mode, and transitions occur by changing the active state object.

## Extensibility and Future Enhancements
- **Supporting Digital Payments**: We could introduce a `PaymentProcessor` interface for card or mobile wallet payments. The `VendingMachine` might have an `acceptPayment` method that coexists with coin insertion.  
- **Touch-Screen Interface**: The selection process can be adapted for a touchscreen by mapping UI inputs to the same methods (`insertCoin`, `selectProduct`).  
- **Adding New Products or Slots**: The `InventoryManager` can easily handle additional slots via `addSlot`. The rest of the system automatically supports new items.  
- **Maintenance and Alerts**: Additional states like `MaintenanceState` or `OutOfServiceState` can be introduced without changing existing state classes.  
- **Change Management Optimizations**: A more advanced system would track the coin inventory for giving change and might refuse purchases if it lacks sufficient coins.  

Each component can be understood and developed independently. The interactions between them are well-defined, and the system remains flexible for additional features or modifications in the future.
