---
{"dg-publish":true,"permalink":"/solid-principles/","dg-note-properties":{}}
---

# SOLID Principles – Summary & Code Examples

The SOLID principles are five design guidelines for object-oriented programming. They help build software that is maintainable, scalable, and resistant to bugs when requirements change. The core objective is to achieve loose coupling between components.

---

## 1. Single Responsibility Principle (SRP)

**Definition**  
A class should have only one reason to change – it must handle a single responsibility.

**Analogy**  
A waiter takes orders. A chef cooks. They do not perform each other's tasks.

**❌ Violation – One class handling multiple tasks**
```java
class Order {
    void calculateTotal() { /* math logic */ }
    void saveToDatabase() { /* SQL logic */ }
}
```

**✅ Compliant – Split into separate classes**
```java
class OrderCalculator {
    void calculateTotal(Order o) {
        System.out.println("Calculating total...");
    }
}

class OrderDatabase {
    void save(Order o) {
        System.out.println("Saving to database...");
    }
}
```

---

## 2. Open/Closed Principle (OCP)

**Definition**  
Software entities should be open for extension but closed for modification. New functionality must be added by creating new classes, not by altering existing ones.

**Analogy**  
Adding a new discount type (e.g., VIP) should involve creating a new discount class, not rewriting the existing discount logic.

**❌ Violation – Modifying existing code for each new type**
```java
class Discount {
    double get(String type) {
        if (type.equals("Gold")) return 20.0;
        if (type.equals("Silver")) return 10.0;
        // Adding "VIP" requires editing this class
        return 0;
    }
}
```

**✅ Compliant – Extending through new classes**
```java
interface Discount {
    double get();
}

class GoldDiscount implements Discount {
    public double get() { return 20.0; }
}

class SilverDiscount implements Discount {
    public double get() { return 10.0; }
}

// New type added without touching existing classes
class VipDiscount implements Discount {
    public double get() { return 50.0; }
}
```

---

## 3. Liskov Substitution Principle (LSP)

**Definition**  
Objects of a superclass must be replaceable with objects of a subclass without altering the correctness of the program.

**Analogy**  
If you have a list of birds that can fly, an ostrich (which cannot fly) should not inherit from that flying bird class. Separate the behaviors properly.

**❌ Violation – Child breaks the parent contract**
```java
class Bird {
    void fly() { System.out.println("Flying high!"); }
}

class Ostrich extends Bird {
    @Override
    void fly() {
        throw new RuntimeException("Ostriches can't fly!");
    }
}

void makeBirdFly(Bird b) { b.fly(); } // Passing Ostrich causes a crash
```

**✅ Compliant – Separate hierarchies based on behavior**
```java
class Bird { }

class FlyingBird extends Bird {
    void fly() { System.out.println("Flying high!"); }
}

class Sparrow extends FlyingBird { }

class Ostrich extends Bird { }
// Ostrich never gets passed to a method expecting fly()
```

---

## 4. Interface Segregation Principle (ISP)

**Definition**  
Clients should not be forced to implement methods they do not use. Large interfaces must be split into smaller, focused ones.

**Analogy**  
A robot working in a restaurant can perform work but does not eat. It should only implement a `Workable` interface, not a fat `RestaurantWorker` interface that includes `eat()`.

**❌ Violation – Fat interface forcing unused methods**
```java
interface RestaurantWorker {
    void work();
    void eat();
}

class Robot implements RestaurantWorker {
    public void work() { System.out.println("Cleaning table"); }
    public void eat() { /* forced to implement, but useless */ }
}
```

**✅ Compliant – Segregated interfaces**
```java
interface Workable { void work(); }
interface Eatable { void eat(); }

class Robot implements Workable {
    public void work() { System.out.println("Cleaning table"); }
}

class Human implements Workable, Eatable {
    public void work() { System.out.println("Serving food"); }
    public void eat() { System.out.println("Eating lunch"); }
}
```

---

## 5. Dependency Inversion Principle (DIP)

**Definition**  
High-level modules must not depend on low-level modules. Both should depend on abstractions (interfaces). Details should depend on abstractions, not the other way around.

**Analogy**  
A notification system should rely on a general `MessageSender` interface. This allows switching between email, SMS, or WhatsApp without rewriting the notification class.

**❌ Violation – High-level class tied to a concrete sender**
```java
class Notification {
    EmailSender sender = new EmailSender(); // hardcoded

    void send(String msg) {
        sender.sendEmail(msg);
    }
}
```

**✅ Compliant – Depends on abstraction injected from outside**
```java
interface MessageSender {
    void send(String msg);
}

class EmailSender implements MessageSender {
    public void send(String msg) {
        System.out.println("Email: " + msg);
    }
}

class SmsSender implements MessageSender {
    public void send(String msg) {
        System.out.println("SMS: " + msg);
    }
}

class Notification {
    private MessageSender sender;

    Notification(MessageSender sender) {
        this.sender = sender;
    }

    void send(String msg) {
        sender.send(msg);
    }
}

// Usage – easy to swap
Notification emailNotif = new Notification(new EmailSender());
Notification smsNotif = new Notification(new SmsSender());
```

---

## Final Takeaway

SOLID is about writing code that tolerates change. Applying these principles reduces ripple effects when requirements evolve, making the system easier to test, debug, and extend over time.